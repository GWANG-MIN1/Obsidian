# SQLAlchemy 2.0 — Mapped[T]가 NOT NULL을 결정한다

> 메모: Mapped[date]는 NOT NULL, Mapped[date | None]은 NULL 허용. 타입 힌트 하나가 DDL을 바꾼다

---

## 개념

SQLAlchemy 2.0의 선언형 매핑은 **타입 애너테이션에서 널 허용 여부를 추론한다.**

```python
class Member(Base):
    name:       Mapped[str]          # NOT NULL
    feedback:   Mapped[str | None]   # NULL 허용
    created_at: Mapped[date]         # NOT NULL  ← 무심코 쓰면 이렇게 된다
```

파이썬 타입 힌트처럼 보이지만 **`CREATE TABLE`의 결과를 바꾼다.**
`Optional`이 아니면 NOT NULL이다.

`mapped_column(nullable=...)`을 직접 주면 그게 우선한다.

```python
created_at: Mapped[date] = mapped_column(Date, nullable=True)   # 애너테이션보다 우선
```

## 실행

### 확인

```python
Base.metadata.create_all(bind=engine)
```

```sql
SELECT table_name, column_name, is_nullable, coalesce(column_default, '(없음)')
FROM information_schema.columns
WHERE table_schema = 'public';
```

```
 table_name |  column_name  | is_nullable |  col_default
------------+---------------+-------------+---------------
 member     | created_at    | NO          | (없음)          ← Mapped[date]
```

### default 와 server_default 는 다르다

| | 어디서 채우나 | DDL 에 남나 |
|---|---|---|
| `default=` | 파이썬(SQLAlchemy)이 INSERT 할 때 | 안 남음 |
| `server_default=` | DB 가 채움 | `DEFAULT ...` 로 남음 |

```python
# 앱이 값을 넣는다. DDL 에는 기본값이 없다.
created_at: Mapped[date] = mapped_column(Date, default=date.today)

# DB 가 채운다. DDL 에 DEFAULT CURRENT_DATE 가 붙는다.
created_at: Mapped[date | None] = mapped_column(Date, server_default=func.current_date())
```

앞의 방식은 **그 앱이 INSERT할 때만** 값이 들어간다.
psql로 직접 넣거나 다른 서비스가 INSERT하면 NULL이 되고, NOT NULL이면 오류가 난다.

### 옮겨 갈 때가 위험하다

`default=` → `server_default=`로 바꾸면 **새로 만드는 테이블만** 기본값이 생긴다.
이미 있는 테이블은 그대로다.

```
예전 테이블: created_at NOT NULL, 기본값 없음   (Mapped[date] + 앱이 값 주입)
새 코드    : created_at 을 보내지 않음          (server_default 에 의존)
→ NOT NULL 위반
```

`create_all()`은 **없는 테이블만** 만들기 때문에 이 차이를 메우지 못한다.
마이그레이션이 필요하다.

```sql
ALTER TABLE member ALTER COLUMN created_at SET DEFAULT CURRENT_DATE;
ALTER TABLE member ALTER COLUMN created_at DROP NOT NULL;
```

## 배운 점

**타입 애너테이션이 스키마를 바꾼다는 걸 의식하고 써야 한다.**
`Mapped[date]`와 `Mapped[date | None]`은 파이썬에서는 거의 같아 보이지만 DDL이 다르다.

**모델을 고칠 때는 "이미 만들어진 테이블은 어떻게 되나"를 같이 생각한다.**
모델 변경은 새 DB에만 반영된다. 기존 DB는 마이그레이션으로만 따라온다.

---

- 처음 만난 곳: [gym-management-db](../../projects/gym-management-db/README.md) —
  [트러블 02](../../projects/gym-management-db/troubleshooting/02-기존-스키마에-배포하면-등록이-422.md)
