# 01. PT 잔여 횟수 차감을 트리거 대신 API에서

> 메모: Oracle 스키마엔 트리거가 있었는데 PostgreSQL로 옮기며 API로 이동. 로컬은 SQL 파일, 배포는 마이그레이션으로 스키마를 만들어서 트리거를 두면 차감이 두 번 일어남

---

## 상황

Oracle 스키마에는 트리거가 있었다. PT 상태가 `COMPLETED`로 바뀌면 회원의 잔여 횟수를 1 줄인다.

```sql
-- 01_create_tables.sql (Oracle, 기록용으로 남겨 둠)
CREATE OR REPLACE TRIGGER trg_pt_session_complete
AFTER UPDATE OF status ON PT_Session
FOR EACH ROW
BEGIN
    IF :NEW.status = 'COMPLETED' AND :OLD.status != 'COMPLETED' THEN
        UPDATE Member SET remaining_pt_count = remaining_pt_count - 1
        WHERE member_id = :NEW.member_id AND remaining_pt_count > 0;
    END IF;
END;
```

PostgreSQL로 옮기면서 이 트리거를 이식하지 않았고, API에도 차감 로직이 없었다.
**수업 기록과 남은 횟수가 서로 맞지 않는 상태**로 4개월을 보냈다.

## 결정

차감을 **API의 완료 처리**에서 한다. PostgreSQL 스키마에는 트리거를 두지 않는다.

```
PATCH /sessions/{id}/complete
  → 상태를 COMPLETED 로 바꾸고
  → 같은 트랜잭션에서 remaining_pt_count 를 1 줄인다
```

## 왜

- **상태 변경과 차감이 한 트랜잭션에 묶인다.** 하나만 성공하는 상태가 없다.
- **잔여 횟수가 0이면 완료 자체를 막을 수 있다.** 조건부 UPDATE의 `rowcount`가 0이면 롤백하고 409를 돌려준다. 트리거는 이미 일어난 UPDATE를 되돌리기 어렵다.

  ```python
  result = db.execute(
      update(Member)
      .where(Member.member_id == pt_session.member_id, Member.remaining_pt_count > 0)
      .values(remaining_pt_count=Member.remaining_pt_count - 1)
  )
  if result.rowcount == 0:
      db.rollback()
      raise HTTPException(409, "잔여 PT 횟수가 없어 완료 처리할 수 없습니다.")
  ```

- **음수가 될 수 없다.** 조건을 UPDATE 문 안에 넣어서, 읽고→판단하고→쓰는 사이에 다른 요청이 끼어들 틈이 없다.
- **테스트로 고정된다.** 완료 시 차감, 두 번 완료 시 409, 잔여 0에서 완료 실패 후 상태가 그대로인지까지 검증한다.

## 왜 다른 건 안 썼나

**트리거를 PostgreSQL에도 이식**

스키마를 만드는 경로가 둘이다 — 로컬은 `sql/01_create_tables_pg.sql`, 배포는 마이그레이션.
트리거를 SQL 파일에만 넣으면 환경에 따라 차감이 되거나 안 된다.
양쪽에 다 넣으면 API 차감과 겹쳐 **한 번 완료에 두 번 깎인다.**

트리거는 애플리케이션 로그에도 남지 않는다. 왜 줄었는지 추적이 어렵다.

**API에서 조회 → 판단 → UPDATE**

읽는 시점과 쓰는 시점 사이에 다른 요청이 끼어들면 음수가 될 수 있다.
조건을 UPDATE 문 안에 두는 쪽을 택했다.
