# SERIAL vs GENERATED AS IDENTITY

> 메모: 둘 다 자동 증가지만 소유권·표준 준수·기본값 지정 가능 여부가 다르다. 기존 SERIAL 컬럼을 IDENTITY로 바꾸는 방법도 함께

---

## 개념

```sql
-- PostgreSQL 전통 방식
id SERIAL PRIMARY KEY

-- SQL 표준 (PostgreSQL 10+)
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

`SERIAL`은 문법 설탕이다. 위 한 줄이 실제로는 이렇게 풀린다.

```sql
CREATE SEQUENCE member_member_id_seq;
ALTER TABLE member ALTER COLUMN member_id SET DEFAULT nextval('member_member_id_seq');
ALTER SEQUENCE member_member_id_seq OWNED BY member.member_id;
```

| | SERIAL | IDENTITY |
|---|---|---|
| 표준 | PostgreSQL 전용 | SQL 표준 |
| 구현 | 시퀀스 + `DEFAULT nextval(...)` | 컬럼 속성 |
| 값 직접 지정 | 가능 (그냥 `DEFAULT`라서) | `ALWAYS`면 불가, `BY DEFAULT`면 가능 |
| `information_schema` | `column_default`에 `nextval(...)` | `is_identity = YES` |

**`ALWAYS`의 의미** — 값을 직접 넣으면 오류가 난다. 실수로 `id`를 지정해 시퀀스와
어긋나는 일을 막아 준다. 꼭 넣어야 하면 `OVERRIDING SYSTEM VALUE`를 쓴다.

```sql
INSERT INTO member (member_id, name) OVERRIDING SYSTEM VALUE VALUES (1, '홍길동');
```

## 실행

### SERIAL → IDENTITY 전환

기존 테이블을 바꿀 때. **데이터를 다시 쓰지 않는 메타데이터 변경**이라 빠르다.

```sql
-- 1) 시퀀스 이름을 먼저 알아 둔다 (DEFAULT 를 지우면 못 찾는다)
SELECT pg_get_serial_sequence('member', 'member_id');   -- public.member_member_id_seq

-- 2) 다음에 쓸 값을 계산
SELECT coalesce(max(member_id), 0) + 1 FROM member;

-- 3) 전환
ALTER TABLE member ALTER COLUMN member_id DROP DEFAULT;
DROP SEQUENCE IF EXISTS public.member_member_id_seq;
ALTER TABLE member ALTER COLUMN member_id ADD GENERATED ALWAYS AS IDENTITY;
ALTER TABLE member ALTER COLUMN member_id RESTART WITH 12;   -- 2)에서 구한 값
```

**`RESTART WITH`를 빼먹으면** 새 시퀀스가 1부터 시작해서 기존 행과 PK가 충돌한다.
전환 자체는 성공하고, 다음 INSERT에서 터진다.

### 이미 IDENTITY인지 확인

```sql
SELECT table_name, column_name, is_identity, column_default
FROM information_schema.columns
WHERE table_schema = current_schema() AND column_name LIKE '%_id';
```

마이그레이션을 여러 번 실행해도 안전하게 하려면 이걸 먼저 보고 건너뛴다.

```python
if is_identity == "YES":
    continue
```

### 시드 데이터에서 주의할 점

IDENTITY(또는 SERIAL) 테이블에 **PK를 직접 지정해서** 넣으면 시퀀스가 그대로 있는다.
그 뒤 애플리케이션이 INSERT하면 1번부터 다시 발급해서 **기존 행과 충돌한다.**

그래서 시드 파일은 PK를 DB에 맡기고, FK는 UNIQUE 값으로 찾아 연결하는 게 안전하다.

```sql
INSERT INTO pt_session (member_id, trainer_id, session_date, session_time, status)
SELECT m.member_id, t.trainer_id, v.session_date::DATE, v.session_time, v.status
FROM (VALUES ('010-1111-2221', 'Trainer Kim', '2025-03-10', '10:00', 'COMPLETED'))
     AS v (phone, trainer_name, session_date, session_time, status)
INNER JOIN member  AS m ON v.phone = m.phone
INNER JOIN trainer AS t ON v.trainer_name = t.name;
```

## 배운 점

**새로 만들면 IDENTITY, 기존 것은 굳이 안 바꿔도 된다.** 전환에서 얻는 건 표준 준수와
실수 방지이고, 그 자체로 성능이 좋아지지는 않는다.

다만 **정의가 여러 곳에 있으면 맞춰 두는 게 낫다.** SQL 파일은 IDENTITY인데 ORM은 SERIAL이면
두 환경의 `information_schema`가 달라지고, 스키마 비교 테스트가 매번 걸린다.

---

- 처음 만난 곳: [gym-management-db](../../projects/gym-management-db/README.md) —
  [트러블 02](../../projects/gym-management-db/troubleshooting/02-기존-스키마에-배포하면-등록이-422.md)
