# Alembic — 이미 있는 DB를 이력에 편입시키기 (stamp)

> 메모: 마이그레이션 이력이 없는 운영 DB에 Alembic을 도입할 때. baseline 리비전을 만들고 stamp로 '여기까지 적용된 셈 치기'

---

## 개념

Alembic은 `alembic_version` 테이블에 **현재 리비전 하나**를 기록한다.
`upgrade head`는 그 지점부터 최신까지의 리비전을 순서대로 실행한다.

문제는 **이미 테이블이 있는데 이력이 없는 DB**다. `create_all()`이나 손으로 만든 운영 DB가 여기 해당한다.
그대로 `upgrade head`를 돌리면 첫 리비전이 `CREATE TABLE`부터 시도해서 "이미 있다"로 실패한다.

`stamp`는 **아무 SQL도 실행하지 않고 `alembic_version`만 기록한다.**
"이 DB는 여기까지 적용된 셈 친다"는 선언이다.

```
alembic stamp <revision>   # 이력만 기록, 스키마는 손대지 않음
alembic upgrade head       # 그 다음 리비전부터 실행
```

## 실행

### 리비전을 두 개로 나눈다

| 리비전 | 내용 |
|---|---|
| `0001` | **기존 스키마 그대로** (지금 운영 DB의 모습) |
| `0002` | 실제로 바꾸고 싶은 것 |

그리고 기동 시 이렇게 판단한다.

```python
with target.connect() as connection:
    tables = set(inspect(connection).get_table_names())

if "alembic_version" not in tables and "member" in tables:
    command.stamp(config, "0001")     # 이력 없는 기존 DB → 기준점 지정
command.upgrade(config, "head")
```

```
빈 DB            → 0001 → 0002
기존 DB(이력 없음) → 0001 로 stamp → 0002 만 적용
이미 최신 DB      → 아무것도 하지 않음
```

### 왜 0001을 '기존 스키마'로 두나

`0001`을 **최신 스키마**로 두는 게 직관적으로 보이지만 두 가지가 깨진다.

- 기존 DB에 stamp하면 **이미 최신인 셈 쳐서** 실제 변경(`0002`)이 영영 적용되지 않는다.
- 빈 DB는 `0001` 하나로 끝나므로 **업그레이드 코드가 한 번도 실행되지 않는다.**
  배포 때 처음 도는 코드가 된다.

`0001`을 과거로 두면 빈 DB도 `0001 → 0002`를 지난다. 개발·CI·배포가 같은 경로를 밟는다.

### 리비전을 멱등하게 쓴다

운영 DB가 "예전 테이블 + 새로 생긴 테이블"처럼 섞여 있을 수 있다. 그래서 `0002`는 여러 번 돌려도 되게 쓴다.

```python
tables = set(sa.inspect(conn).get_table_names())
if "exercise" not in tables:
    op.create_table("exercise", ...)

op.execute(f"ALTER TABLE {t} ALTER COLUMN {c} SET DEFAULT {d}")   # 원래 멱등
op.execute(f"ALTER TABLE {t} DROP CONSTRAINT IF EXISTS {name}")   # 지우고
op.execute(f"ALTER TABLE {t} ADD CONSTRAINT {name} CHECK (...)")  # 다시 만든다
op.execute(f"CREATE INDEX IF NOT EXISTS {name} ON {t} ({cols})")
```

제약 이름 변경은 `DO $$ ... $$` 블록으로 존재 여부를 보고 처리한다.

```sql
DO $$
BEGIN
    IF EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'old_name' AND conrelid = 'tbl'::regclass)
       AND NOT EXISTS (SELECT 1 FROM pg_constraint WHERE conname = 'new_name' AND conrelid = 'tbl'::regclass)
    THEN
        ALTER TABLE tbl RENAME CONSTRAINT old_name TO new_name;
    END IF;
END $$;
```

### 제약을 붙이기 전에 데이터를 검사한다

새 CHECK나 UNIQUE는 **기존 행을 검사**한다. 위반이 있으면 마이그레이션이 실패하고,
PostgreSQL 오류만으로는 어느 행인지 알 수 없다. 먼저 확인해서 알려주는 편이 낫다.

```python
duplicates = conn.execute(text(
    "SELECT member_id, session_date, session_time, count(*) FROM pt_session "
    "WHERE status <> 'CANCELLED' GROUP BY 1,2,3 HAVING count(*) > 1 LIMIT 5"
)).fetchall()
if duplicates:
    raise RuntimeError(f"중복 예약이 있어 인덱스를 만들 수 없습니다: {duplicates}")
```

### env.py는 접속 정보를 앱과 공유하게

`alembic.ini`의 `sqlalchemy.url`은 비워 두고 앱의 엔진을 그대로 쓰면 접속 정보가 한 곳에만 남는다.
테스트에서 다른 DB를 대상으로 돌려야 할 때는 `Config.attributes`로 엔진을 넘긴다.

```python
def run_migrations_online() -> None:
    engine = context.config.attributes.get("engine") or default_engine
    with engine.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()
```

## 배운 점

**`stamp`는 "속이는" 명령이 아니라 기준점을 선언하는 명령이다.**
이력이 없는 DB를 도구에 편입시키는 정식 방법이다.

**downgrade를 못 쓰겠으면 못 쓴다고 막아 두는 게 낫다.** 테이블 추가나 타입 전환이 섞이면
자동 복구가 안전하지 않다. 어설픈 downgrade보다 "스냅샷에서 복구"가 정직하다.

```python
def downgrade() -> None:
    raise NotImplementedError("... RDS 스냅샷에서 복구하세요.")
```

---

- 처음 만난 곳: [gym-management-db](../../projects/gym-management-db/README.md) —
  [결정 03](../../projects/gym-management-db/decisions/03-Alembic-도입과-기존-DB-편입.md)
