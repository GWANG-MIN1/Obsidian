# gym-management-db

헬스장 운영 데이터를 PostgreSQL로 설계하고 FastAPI·Docker·AWS로 배포한 백엔드 프로젝트.

- 저장소: https://github.com/GWANG-MIN1/gym-management-db
- 기간:
- 한 줄 요약:

> 저장소 README가 "이 프로젝트가 무엇인가"라면, 이 노트는 **"내가 무엇을 겪었나"** 다.
> 같은 내용을 옮겨 적지 않는다.

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

## 개념 노트

이 프로젝트에서 나왔지만 다음에도 쓸 지식은 `infra-lab/` 에 둔다. 여기서는 링크만.

- [ ] 부분 유니크 인덱스
- [ ] SERIAL vs IDENTITY
- [ ] Alembic baseline / stamp
- [ ] psql `ON_ERROR_STOP`
- [ ] SQLAlchemy `Mapped[T]` 의 널 허용 추론
- [ ] Read Replica vs Multi-AZ standby

---

## 아직 안 한 것과 이유

- 역할(관리자/트레이너/회원) 기반 권한 분리 —
- 무중단 배포와 롤백 —
- 개선 후 부하 테스트 재측정 —
