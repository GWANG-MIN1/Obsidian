# projects

직접 만든 것들. [`../infra-lab/`](../infra-lab/README.md) 가 **"무엇을 배웠나"** 라면
여기는 **"무엇을 만들었고 그 과정에서 무엇을 겪었나"** 다.

각 프로젝트는 같은 구성을 따른다.

```
README.md          한 줄 요약 · 숫자로 말할 것 · 아직 안 한 것과 이유
타임라인.md         커밋 기록을 구간으로 접은 것
decisions/         무엇을 왜 그렇게 정했나 (버린 선택지 포함)
troubleshooting/   무엇에 막혔고 왜 그랬나 (반증된 가설 포함)
```

> **저장소 README가 "이 프로젝트가 무엇인가"라면, 이 노트는 "내가 무엇을 겪었나"다.**
> 같은 내용을 옮겨 적지 않는다.

---

## 목록

| | 프로젝트 | 한 줄 | 기간 |
|---|---|---|---|
| 01 | [gym-management-db](gym-management-db/README.md) | 헬스장 운영 DB를 설계하고 FastAPI·Docker·AWS로 배포한 백엔드 | 2025-11-24 ~ 2026-09-06 |
| 02 | [aws-serverless-agent](aws-serverless-agent/README.md) | AWS 서버리스 위의 AI 에이전트를 부품부터 다시 쌓은 학습 프로젝트 | 2026-05-23 ~ 2026-06-25 |
| 03 | [serverless-uptime-monitor](serverless-uptime-monitor/README.md) | EC2 없이 서버리스로 만든 업타임 모니터링 서비스 | 2026-05-14 ~ 2026-08-01 |
| 04 | [riskdetector](riskdetector/README.md) | 계약서 독소조항을 찾아 주는 AI 서비스 — **캡스톤 팀 프로젝트 · 장려상** | 2026-03-23 ~ 2026-06-07 |

|  | gym-management-db | aws-serverless-agent | serverless-uptime-monitor | riskdetector |
|---|---|---|---|---|
| 저장소 | [GWANG-MIN1/gym-management-db](https://github.com/GWANG-MIN1/gym-management-db) | [GWANG-MIN1/aws-serverless-agent](https://github.com/GWANG-MIN1/aws-serverless-agent) | [GWANG-MIN1/serverless-uptime-monitor](https://github.com/GWANG-MIN1/serverless-uptime-monitor) | [LilChaewon/RiskDetector](https://github.com/LilChaewon/RiskDetector) |
| 형태 | 혼자 | 혼자 | 혼자 | **팀 5명 · 내 역할 백엔드** |
| 커밋 · 작업일 | 56개 · 6일 | 151개 · 23일 | 53개 · 11일 | 361개(내 77개) · 23일 |
| 결정 노트 | [9개](gym-management-db/decisions/README.md) | [13개](aws-serverless-agent/decisions/README.md) | [11개](serverless-uptime-monitor/decisions/README.md) | [12개](riskdetector/decisions/README.md) |
| 트러블슈팅 | [10개](gym-management-db/troubleshooting/README.md) | [13개](aws-serverless-agent/troubleshooting/README.md) | [6개](serverless-uptime-monitor/troubleshooting/README.md) | [9개](riskdetector/troubleshooting/README.md) |
| 개념 노트 | [`infra-lab/db-lab/`](../infra-lab/db-lab/README.md) | [`infra-lab/aws-lab/`](../infra-lab/aws-lab/README.md) | [`infra-lab/aws-lab/`](../infra-lab/aws-lab/README.md) (미작성) | [`infra-lab/aws-lab/`](../infra-lab/aws-lab/README.md) (미작성) |

---

## 세 프로젝트가 서로를 비춘다

같은 사람이 만들었는데 **검증 체계가 셋 다 다르다.** 이 대비가 세 노트 묶음에서 제일 많이 배운 것이다.

|  | gym-management-db | aws-serverless-agent | serverless-uptime-monitor |
|---|---|---|---|
| 작업 리듬 | 공백을 사이에 두고 **세 번 돌아옴** (5개월 · 4개월) | 5주 동안 **거의 매일**, 하루에 한 조각 | 하루에 몰아치고 **몇 주를 비움** (5주 · 3주 · 3주) |
| 검증 | **자동 테스트 56개** (로컬 + CI) | **실배포 + 손으로 호출**, 스크린샷 50장 | **자동 테스트 78개 + moto 통합테스트 + CI 게이트** |
| 자동 테스트 | 56개 | **0개** | 78개 |
| 계기 | 외부 **코드 리뷰**를 받고 지적을 고침 | 스스로 세운 규칙("매일 한 가지만") | 저장소 안에 **업그레이드 로드맵 9개**를 만들어 놓고 하나씩 닫음 |
| 드러난 대가 | 5월의 판단이 9월에 어떤 결과로 돌아왔는지 | **완성 선언 8일 뒤 재배포하니 전 요청 500** | **테스트가 다 통과하는데 프로덕션이 조용히 멈춰 있었다** |

### 1) 손으로 하는 검증은 환경이 바뀌면 재현되지 않는다

gym의 [트러블 09](gym-management-db/troubleshooting/09-Secrets-Manager-조회-실패.md)가
*"사람이 콘솔에서 값을 복사해 붙여 넣는 단계는 재현되지 않는다"*로 끝나는데,
aws의 [트러블 12](aws-serverless-agent/troubleshooting/12-재배포하니-전-요청이-500.md)가
같은 결론에 **다른 경로로** 도달한다 — 22일 내내 실배포 검증을 하고도 환경이 바뀌자 무너졌다.

반대 방향도 있다. gym은 리뷰를 받기 전까지 `|| true`로 **CI를 무력화한 채 4개월**을 갔다
([트러블 08](gym-management-db/troubleshooting/08-린트를-우회했다가-검증이-무력화.md)).
aws는 검증을 매일 했지만 **그 검증을 자동화하지 않았다.**
둘 다 "검증이 있다"와 "검증이 작동한다" 사이의 간극이다.

### 2) 그런데 자동화해도 뚫린다

세 번째 프로젝트가 앞의 두 결론을 **한 칸 더 밀었다.**
uptime-monitor는 자동 테스트 78개, moto 통합테스트, ruff·pytest·fmt·tfsec CI 게이트가
**전부 있는 상태에서** 프로덕션 헬스체크가 며칠간 통째로 멈춰 있었다.
그동안 DOWN 알림은 **한 건도 없었다** — 체크가 안 도니 상태값이 `UP` 에 얼어붙었기 때문이다.
→ [트러블 01](serverless-uptime-monitor/troubleshooting/01-헬스체크가-조용히-멈췄다.md)

원인은 테스트 픽스처가 엔드포인트를 **문자열로만** 채워서 실제 DynamoDB 가 돌려주는
`Decimal` 경로를 한 번도 안 지나갔다는 것이다.
**테스트는 내가 상상한 입력만 검증한다.**

### 3) 그래서 남는 것

| 층 | 무엇을 잡나 | 어디서 |
|---|---|---|
| 코드 리뷰 | 내가 안 본 것 | gym 3단계 |
| 자동 테스트 | 내가 상상한 입력 | gym · uptime |
| 실배포 검증 | 그 순간의 환경 | aws-serverless-agent |
| **운영 신호** | **아무것도 안 일어나고 있다는 사실** | uptime [결정 05](serverless-uptime-monitor/decisions/05-요약-알림을-기존-알림-큐에-얹기.md) · [06](serverless-uptime-monitor/decisions/06-살아있음을-status-대신-신선도로.md) |

위의 세 층은 전부 **배포 전**에 작동한다. 마지막 하나만 **배포 후**에 작동하고,
세 프로젝트에서 실제로 장애를 잡아낸 것은 그것이었다.

uptime-monitor의 요약 알림은 *"지금 N개 전부 UP"* 을 하루 한 번 보내서
**정상임을 능동적으로 증명**한다. 조용한 것이 정상인지 모니터가 죽은 것인지 구분하기 위해서다.
그리고 배포 첫날 그 알림이 `stale` 경고를 띄우며 위의 버그를 드러냈다.

---

## 네 번째는 팀 프로젝트다 — 반대 방향에서 같은 결론

위 세 개는 전부 혼자 만든 것이고, [riskdetector](riskdetector/README.md) 만
**5명이 만든 캡스톤디자인 프로젝트**다(내 역할은 백엔드, 장려상).
이걸 나중에 정리하면서 가장 놀란 건 이 대비였다.

| | 솔로 3개 | riskdetector (팀) |
|---|---|---|
| 병합 커밋 | 2 · 2 · 0회 | **144회** |
| 자동 테스트 | 56 · 0 · 78개 | **1개** (`contextLoads`) |
| CI | 있음 (게이트 4종) · 있음 · 있음 | **없음** |
| 검증 | 테스트 / 실배포 / 운영 신호 | 실배포 + 손으로 호출 |

**협업 절차는 팀이 압도적으로 많은데, 검증 장치는 팀이 거의 없다.**
브랜치를 파고 PR 을 올리고 리뷰하고 머지하는 일을 144번 했는데,
그 절차가 확인한 것은 *"머지해도 되는가"* 였고 *"동작하는가"* 가 아니었다.
혼자일 때 붙이던 것(테스트 · ruff · CI 게이트 · 시크릿 스캔)을 **팀에서는 하나도 안 붙였다.**
"팀이 안 쓰니까"로 넘겼고, 그 대가가
[트러블 01](riskdetector/troubleshooting/01-Spring이-전시-5일-전에-교체됐다.md) 과
[트러블 09](riskdetector/troubleshooting/09-비밀번호를-커밋하고-마스킹으로-덮었다.md) 다.

### 시간 순서가 인과다

날짜를 겹쳐 보면 이 네 개가 **한 줄로 읽힌다.**

```
2026-03  riskdetector 시작 — Lambda·Bedrock·S3 를 팀 코드 안에서 처음 쓴다
2026-05  ├─ 05-14  serverless-uptime-monitor 시작   (Lambda·SQS·DynamoDB 를 혼자)
         └─ 05-23  aws-serverless-agent 시작        (Bedrock 을 부품부터)
2026-06  riskdetector 종료 (06-07)
2026-08~09  uptime-monitor · gym-management-db 마무리
```

**캡스톤이 아직 돌아가는 중에 솔로 학습 저장소 두 개를 시작했다.**
aws-serverless-agent 의 한 줄 요약이 *"부품부터 다시 쌓은 학습 프로젝트"* 인데,
그 "다시"의 대상이 riskdetector 에서 쓰긴 했지만 이해하지는 못한 채 넘어간 것들이다.
**팀 프로젝트가 공백을 알려 주고, 솔로 프로젝트가 그 공백을 메웠다.**

### 그리고 코드로 방어되지 않는 층이 있었다

위 [세 프로젝트 비교](#세-프로젝트가-서로를-비춘다)는 전부 **코드와 검증**의 이야기다 —
리뷰가 잡는 것, 테스트가 잡는 것, 운영 신호가 잡는 것. riskdetector 는 그 목록 밖에서 무너졌다.

**Spring Boot 백엔드가 전시 5일 전에 Supabase 로 교체됐고, 이유는 비용이었다.**
Render 에 웹 서비스 2개 + 관리형 PostgreSQL 을 상시로 띄우는 구성이 학생 팀이
계속 낼 수 있는 비용이 아니었다. 내 코드가 더 깔끔했어도 결과는 같았을 것이다.
→ [트러블 01](riskdetector/troubleshooting/01-Spring이-전시-5일-전에-교체됐다.md)

| 층 | 무엇을 잡나 | 어디서 |
|---|---|---|
| 코드 리뷰 · 테스트 · 실배포 · 운영 신호 | 위 세 프로젝트에서 정리한 것 | 솔로 3개 |
| **비용과 유지 가능성** | **"이걸로 만들 수 있나"가 아니라 "이걸 유지할 수 있나"** | riskdetector |

그리고 이건 riskdetector 만의 공백이 아니다.
[serverless-uptime-monitor](serverless-uptime-monitor/README.md) 에도
*"비용을 집계하지 않았다. 프리티어 범위라고 판단했을 뿐 청구서로 확인한 값이 없다"* 고 적혀 있고,
[aws-serverless-agent](aws-serverless-agent/README.md) 만 Phase 2 실비용 $0.07 을 확인했다.
**팀에서 대가를 치른 항목을 혼자 할 때도 여전히 안 보고 있었다.**

기술 선택은 *"이걸로 만들 수 있나"* 만이 아니라 *"이걸 유지할 수 있나"* 로도 갈린다.
후자를 계산하는 것도 백엔드 담당의 일이었고, 그때는 그 생각을 못 했다.
