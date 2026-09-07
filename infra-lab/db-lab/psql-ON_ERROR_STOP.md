# psql ON_ERROR_STOP

> 메모: 이 옵션이 없으면 psql은 SQL 오류가 나도 남은 문장을 계속 실행하고 종료 코드 0으로 끝난다 → CI가 실패를 놓친다

---

## 개념

`psql`은 대화형 도구가 기본이다. **한 문장이 실패해도 다음 문장을 계속 실행**하고,
마지막에 **종료 코드 0**으로 끝난다. 사람이 화면을 보고 있다는 전제다.

스크립트나 CI에서는 이 기본값이 위험하다.

```bash
psql -f schema.sql          # 중간에 실패해도 exit 0
echo $?                     # 0  ← 성공으로 보인다
```

`ON_ERROR_STOP=1`을 주면 첫 오류에서 멈추고 **종료 코드 3**을 돌려준다.

```bash
psql -v ON_ERROR_STOP=1 -f schema.sql
```

**환경변수가 아니라 psql 변수다.** `export ON_ERROR_STOP=1`은 아무 효과가 없다.
`-v`로 넘기거나 스크립트 안에서 `\set ON_ERROR_STOP on`을 쓴다.

## 실행

### CI에서

```yaml
- name: Apply schema
  run: psql -v ON_ERROR_STOP=1 -f sql/01_create_tables_pg.sql

- name: Load sample data
  run: psql -v ON_ERROR_STOP=1 -f sql/02_insert_sample_data_pg.sql
```

### "실패해야 정상"인 것을 검사할 때

제약이 실제로 막는지 확인하려면 **성공하면 실패**로 뒤집어야 한다.

```bash
assert_rejected() {
  if psql -v ON_ERROR_STOP=1 -q -c "$1" >/dev/null 2>&1; then
    echo "::error::허용되면 안 되는 SQL 이 통과했습니다 — $2"
    exit 1
  fi
  echo "거부됨(정상): $2"
}

assert_rejected \
  "INSERT INTO pt_session (member_id, trainer_id, session_date, session_time)
   SELECT 1, 1, CURRENT_DATE, '99:99'" \
  "존재하지 않는 시각 99:99"
```

### 값 검증은 DO 블록으로

`SELECT count(*)`를 출력만 하면 CI는 그 숫자를 보지 않는다. 예외를 던져야 실패한다.

```sql
DO $$
DECLARE
  expected CONSTANT jsonb := '{"member":10,"trainer":10,"pt_session":13}';
  table_name text;
  actual bigint;
BEGIN
  FOR table_name IN SELECT jsonb_object_keys(expected) LOOP
    EXECUTE format('SELECT count(*) FROM %I', table_name) INTO actual;
    IF actual <> (expected ->> table_name)::bigint THEN
      RAISE EXCEPTION '% 건수가 % 여야 하는데 % 입니다', table_name, expected ->> table_name, actual;
    END IF;
  END LOOP;
END $$;
```

### 트랜잭션과 함께

`-1`(또는 `--single-transaction`)을 같이 주면 전체를 한 트랜잭션으로 묶는다.
실패 시 아무것도 남지 않아 "절반만 적용된 스키마"를 피할 수 있다.

```bash
psql -v ON_ERROR_STOP=1 -1 -f schema.sql
```

## 배운 점

**CI에서 쓰는 도구는 "실패했을 때 어떻게 끝나는지"부터 확인해야 한다.**
`psql`처럼 기본값이 관대한 도구가 있고, 그 위에 세운 검증은 통과해도 아무 의미가 없다.

같은 종류의 함정

| 도구 | 기본 동작 | 대응 |
|---|---|---|
| `psql` | 오류 나도 계속, exit 0 | `-v ON_ERROR_STOP=1` |
| 셸 스크립트 | 중간 명령 실패해도 계속 | `set -euo pipefail` |
| 파이프라인 | 마지막 명령의 종료 코드만 반영 | `set -o pipefail` |
| `\|\| true` | 무조건 성공 | 쓰지 않는다 |

---

- 처음 만난 곳: [gym-management-db](../../projects/gym-management-db/README.md) —
  [트러블 05](../../projects/gym-management-db/troubleshooting/05-CI가-위반이-있어도-항상-성공.md)
