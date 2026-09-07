# db-lab

PostgreSQL과 스키마 운영에서 만난 개념들.
[gym-management-db](../../projects/gym-management-db/README.md) 프로젝트에서 나왔지만,
**다음 프로젝트에서도 다시 볼 내용**이라 프로젝트 폴더가 아니라 여기에 둔다.

각 노트는 `개념 정리 + 실행 가능한 명령 + 배운 점` 구성을 따른다.

| | 노트 | 한 줄 |
|---|---|---|
| 01 | [부분 유니크 인덱스](부분-유니크-인덱스.md) | 조건에 맞는 행만 유일성 검사 — "취소된 예약은 슬롯을 비운다" |
| 02 | [SERIAL vs IDENTITY](SERIAL-vs-IDENTITY.md) | 자동 증가 두 방식의 차이와 전환 방법 |
| 03 | [Alembic baseline 과 stamp](Alembic-baseline-과-stamp.md) | 이력 없는 기존 DB를 마이그레이션 도구에 편입시키기 |
| 04 | [psql ON_ERROR_STOP](psql-ON_ERROR_STOP.md) | 없으면 오류가 나도 exit 0 — CI가 실패를 놓친다 |
| 05 | [SQLAlchemy Mapped 널 추론](SQLAlchemy-Mapped-널-추론.md) | 타입 애너테이션이 NOT NULL을 결정한다 |
| 06 | [Read Replica vs Multi-AZ](Read-Replica-vs-Multi-AZ.md) | 읽기 분산과 장애 대비 — 목적이 다르다 |

## 관통하는 주제

01·05는 **"스키마에 규칙을 어디까지 담을 것인가"**,
03·04는 **"변경과 검증을 어떻게 자동화할 것인가"**,
02·06은 **"기본값과 옵션이 실제로 무엇을 바꾸는가"**에 대한 것이다.
