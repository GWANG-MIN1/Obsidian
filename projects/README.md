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

|  | gym-management-db | aws-serverless-agent | serverless-uptime-monitor |
|---|---|---|---|
| 저장소 | [GWANG-MIN1/gym-management-db](https://github.com/GWANG-MIN1/gym-management-db) | [GWANG-MIN1/aws-serverless-agent](https://github.com/GWANG-MIN1/aws-serverless-agent) | [GWANG-MIN1/serverless-uptime-monitor](https://github.com/GWANG-MIN1/serverless-uptime-monitor) |
| 커밋 · 작업일 | 56개 · 6일 | 151개 · 23일 | 53개 · 11일 |
| 결정 노트 | [9개](gym-management-db/decisions/README.md) | [13개](aws-serverless-agent/decisions/README.md) | [11개](serverless-uptime-monitor/decisions/README.md) |
| 트러블슈팅 | [10개](gym-management-db/troubleshooting/README.md) | [13개](aws-serverless-agent/troubleshooting/README.md) | [6개](serverless-uptime-monitor/troubleshooting/README.md) |
| 개념 노트 | [`infra-lab/db-lab/`](../infra-lab/db-lab/README.md) | [`infra-lab/aws-lab/`](../infra-lab/aws-lab/README.md) | [`infra-lab/aws-lab/`](../infra-lab/aws-lab/README.md) (미작성) |

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
