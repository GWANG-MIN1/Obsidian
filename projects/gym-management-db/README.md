# gym-management-db

헬스장 운영 데이터를 PostgreSQL로 설계하고 FastAPI·Docker·AWS로 배포한 백엔드 프로젝트.

- 저장소: https://github.com/GWANG-MIN1/gym-management-db
- 기간: 2025-11-24 ~ 2026-09-06 (커밋 56개 · 실제 작업일 6일 · 3단계)
- 한 줄 요약: **DB 스키마 설계에서 시작해 배포·모니터링까지 이어 붙이고,
  코드 리뷰를 받아 검증 체계를 세운 백엔드 프로젝트**

> 저장소 README가 "이 프로젝트가 무엇인가"라면, 이 노트는 **"내가 무엇을 겪었나"** 다.
> 같은 내용을 옮겨 적지 않는다.
> 다른 프로젝트는 [`projects/`](../README.md) 에 있다.

---

## [타임라인](타임라인.md)

커밋 56개 · 2025-11-24 ~ 2026-09-06 · 3단계 (과제 → 클라우드 확장 → 리뷰 반영)

---

## 결정 — 왜 그렇게 만들었나

| | 노트 |
|---|---|
| 01 | [PT 차감을 트리거 대신 API에서](decisions/01-PT-차감을-트리거-대신-API에서.md) |
| 02 | [중복 예약을 부분 유니크 인덱스로](decisions/02-중복-예약을-부분-유니크-인덱스로.md) |
| 03 | [Alembic 도입과 기존 DB 편입](decisions/03-Alembic-도입과-기존-DB-편입.md) |
| 04 | [스키마 정의 3곳 유지와 자동 비교](decisions/04-스키마-정의-3곳-유지와-자동-비교.md) |
| 05 | [읽기 커넥션 분리](decisions/05-읽기-커넥션-분리.md) |
| 06 | [API Key만 넣고 역할 분리는 보류](decisions/06-API-Key만-넣고-역할-분리는-보류.md) |
| 07 | [커넥션 풀 크기를 15로](decisions/07-커넥션-풀-크기를-15로.md) |
| 08 | [테스트를 실제 PostgreSQL로](decisions/08-테스트를-실제-PostgreSQL로.md) |
| 09 | [오류 매핑을 제약 이름 기준으로](decisions/09-오류-매핑을-제약-이름-기준으로.md) |

## 트러블슈팅 — 무엇에 막혔나

| | 노트 |
|---|---|
| 01 | [테스트가 개발 DB 테이블을 삭제](troubleshooting/01-테스트가-개발DB-테이블을-삭제.md) |
| 02 | [기존 스키마에 배포하면 등록이 422](troubleshooting/02-기존-스키마에-배포하면-등록이-422.md) |
| 03 | [GET /members p95 2초 — 원인 오진](troubleshooting/03-GET-members-p95-2초.md) |
| 04 | [.gitignore 인라인 주석으로 무시 실패](troubleshooting/04-gitignore-인라인-주석으로-무시-실패.md) |
| 05 | [CI가 위반이 있어도 항상 성공](troubleshooting/05-CI가-위반이-있어도-항상-성공.md) |
| 06 | [CI에서만 테스트 7개 실패](troubleshooting/06-CI에서만-테스트-7개-실패.md) |
| 07 | [큰 숫자 입력이 500](troubleshooting/07-큰-숫자-입력이-500.md) |
| 08 | [린트를 우회했다가 검증이 무력화](troubleshooting/08-린트를-우회했다가-검증이-무력화.md) |
| 09 | [Secrets Manager 조회 실패](troubleshooting/09-Secrets-Manager-조회-실패.md) |
| 10 | [배포 후 헬스체크가 안 떴다](troubleshooting/10-배포-후-헬스체크가-안-떴다.md) |

## 개념 노트

이 프로젝트에서 나왔지만 다음에도 쓸 지식은 [`infra-lab/db-lab/`](../../infra-lab/db-lab/) 에 둔다.

- [x] [부분 유니크 인덱스](../../infra-lab/db-lab/부분-유니크-인덱스.md)
- [x] [SERIAL vs IDENTITY](../../infra-lab/db-lab/SERIAL-vs-IDENTITY.md)
- [x] [Alembic baseline 과 stamp](../../infra-lab/db-lab/Alembic-baseline-과-stamp.md)
- [x] [psql ON_ERROR_STOP](../../infra-lab/db-lab/psql-ON_ERROR_STOP.md)
- [x] [SQLAlchemy Mapped[T] 널 추론](../../infra-lab/db-lab/SQLAlchemy-Mapped-널-추론.md)
- [x] [Read Replica vs Multi-AZ](../../infra-lab/db-lab/Read-Replica-vs-Multi-AZ.md)

---

## 아직 안 한 것과 이유

**역할(관리자/트레이너/회원) 기반 권한 분리**
사용자 테이블·비밀번호 해시·토큰 발급과 갱신까지 따라온다. 이 프로젝트의 주제(DB 설계와 운영)에서
비중이 뒤집힌다고 판단해, 쓰기 잠금(API Key) 하나만 넣고 **없는 것은 없다고 README에 적었다.**
→ [결정 06](decisions/06-API-Key만-넣고-역할-분리는-보류.md)

**무중단 배포와 롤백**
CD가 기존 컨테이너를 먼저 내리고 새로 띄우는 구조라 배포 중 짧은 중단이 있고, 실패 시 자동 복구가 없다.
AWS 인프라를 내린 상태라 지금 손대도 검증할 방법이 없다.

**개선 후 부하 테스트 재측정**
README의 p95 수치는 전부 **개선 전** 값이다. 페이지네이션·인덱스·읽기 분리를 넣었지만
같은 조건(EC2 t3.micro + RDS db.t3.micro)을 다시 만들지 않으면 비교가 의미 없다.
→ [트러블 03](troubleshooting/03-GET-members-p95-2초.md)

**Alembic downgrade**
`0002`는 테이블 추가와 SERIAL → IDENTITY 전환을 포함해 자동 복구가 안전하지 않다.
어설픈 downgrade보다 "스냅샷에서 복구"가 정직하다고 보고 `NotImplementedError`로 막았다.

**Replica 헬스체크**
`/health`는 Primary만 확인한다. Replica가 끊겨도 200이고 조회만 실패한다.
실제로 Replica를 운영할 때 보완할 것.

---

## 다른 프로젝트와의 대비

[aws-serverless-agent](../aws-serverless-agent/README.md) 와 **검증 체계가 정반대**다.
이쪽은 코드 리뷰를 받아 자동 테스트 56개를 넣었고, 저쪽은 22일 내내 실배포로 검증하고도
자동 테스트가 0이다. 그리고 저쪽은 **완성을 선언한 8일 뒤 재배포에서 전 요청 500**을 만났다
→ [트러블 12](../aws-serverless-agent/troubleshooting/12-재배포하니-전-요청이-500.md)

여기 [트러블 09](troubleshooting/09-Secrets-Manager-조회-실패.md) 가 남긴
*"사람이 콘솔에서 값을 복사해 붙여 넣는 단계는 재현되지 않는다"* 와 같은 결론에
**다른 경로로** 도달한 사례다. → [두 프로젝트 비교](../README.md#두-프로젝트가-서로를-비춘다)

