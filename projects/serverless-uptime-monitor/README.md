# serverless-uptime-monitor

EC2 없이 Lambda·EventBridge·SQS·DynamoDB 만으로 만든 업타임 모니터링 서비스.

- 저장소: https://github.com/GWANG-MIN1/serverless-uptime-monitor
- 기간: 2026-05-14 ~ 2026-08-01 (커밋 53개 · 실제 작업일 11일 · 4단계)
- 한 줄 요약: **"장애를 알려 주는 시스템"을 만들다가,
  그 시스템 자체가 조용히 죽는 것을 어떻게 알아채는가로 주제가 옮겨 간 프로젝트**

> 저장소 README가 "이 프로젝트가 무엇인가"라면, 이 노트는 **"내가 무엇을 정했고 무엇에 막혔나"** 다.
> 같은 내용을 옮겨 적지 않는다. 다른 프로젝트는 [`projects/`](../README.md) 에 있다.

> **기록의 성격** — 이 프로젝트는 개선 하나하나를 [`docs/upgrades/`](https://github.com/GWANG-MIN1/serverless-uptime-monitor/tree/main/docs/upgrades)
> 에 **"개요를 먼저 커밋하고 다음 날 구현"** 하는 형태로 남겼다 (→ [결정 10](decisions/10-스켈레톤을-먼저-커밋하고-구현은-다음-날.md)).
> 그래서 "무엇을 하려 했는가"는 저장소에 이미 있다. 여기엔 **버린 선택지**와
> **문서에 안 적힌 실패**를 적는다.

---

## [타임라인](타임라인.md)

커밋 53개 · 2026-05-14 ~ 2026-08-01 · 4단계 (뼈대 → fan-out 재설계 → 업그레이드 8종 → 요약 알림)

---

## 이 프로젝트의 축 — 조용한 실패

모니터링 시스템의 최악은 **장애를 놓치는 것이 아니라, 장애를 못 보는 상태를 모르는 것**이다.
이 프로젝트에서 반복해서 나온 형태가 그거였다.

| 무엇이 | 어떻게 조용했나 | 어디서 |
|---|---|---|
| DynamoDB `scan` 첫 페이지만 읽음 | 엔드포인트가 **에러 없이** 빠진다 | [트러블 02](troubleshooting/02-같은-페이지네이션-버그를-세-번.md) |
| 리포트 `query` 첫 페이지만 읽음 | uptime% 가 **틀린 값으로** 계산된다 | [트러블 02](troubleshooting/02-같은-페이지네이션-버그를-세-번.md) |
| enumerator 가 `Decimal` 직렬화로 죽음 | 체크가 멈췄는데 `status` 는 `UP` 에 얼어붙는다 | [트러블 01](troubleshooting/01-헬스체크가-조용히-멈췄다.md) ⭐ |

세 번째를 잡은 것은 테스트가 아니라 **운영 중인 시스템이 스스로 보낸 요약 알림**이었다.
`status` 값을 믿지 않고 `last_checked_at` **신선도**를 따로 검사한 것이 결정적이었다.
→ [결정 06](decisions/06-살아있음을-status-대신-신선도로.md)

---

## 결정 — 왜 그렇게 만들었나

| | 노트 |
|---|---|
| 01 | [헬스체크를 enumerator + worker fan-out 으로 분리](decisions/01-헬스체크를-enumerator와-worker로-분리.md) |
| 02 | [알림을 직접 호출 대신 SQS 큐 경유로](decisions/02-알림을-SQS-큐-경유로.md) |
| 03 | [체크 이력을 배치 삭제 대신 DynamoDB TTL 30일로](decisions/03-체크-이력을-TTL-30일로.md) |
| 04 | ["마지막 체크 결과"와 "확정된 알림 상태"를 분리](decisions/04-알림-상태를-raw-status와-분리.md) |
| 05 | [요약 알림을 새 배선 대신 기존 알림 큐에 얹기](decisions/05-요약-알림을-기존-알림-큐에-얹기.md) |
| 06 | [살아있음 판정을 status 대신 last_checked_at 신선도로](decisions/06-살아있음을-status-대신-신선도로.md) |
| 07 | [SSRF 를 등록 시점에 차단 (worker 재검증은 보류)](decisions/07-SSRF를-등록-시점에-차단.md) |
| 08 | [배포를 push 자동 실행에서 수동 트리거로](decisions/08-배포를-자동-push에서-수동-트리거로.md) |
| 09 | [tfsec 은 soft-fail, ruff·fmt 만 하드 게이트로](decisions/09-tfsec을-soft-fail로.md) |
| 10 | [스켈레톤을 먼저 커밋하고 구현은 다음 날](decisions/10-스켈레톤을-먼저-커밋하고-구현은-다음-날.md) |
| 11 | [로그 헬퍼를 공용 모듈 대신 핸들러마다 복제](decisions/11-로그-헬퍼를-핸들러마다-복제.md) |

## 트러블슈팅 — 무엇에 막혔나

| | 노트 |
|---|---|
| 01 | [헬스체크가 조용히 멈췄는데 DOWN 알림은 한 건도 없었다](troubleshooting/01-헬스체크가-조용히-멈췄다.md) ⭐ |
| 02 | [같은 페이지네이션 버그를 세 곳에서 세 번 고쳤다](troubleshooting/02-같은-페이지네이션-버그를-세-번.md) |
| 03 | [DNS 실패와 타임아웃이 같은 메시지로 나왔다](troubleshooting/03-DNS-실패와-타임아웃이-같은-메시지.md) |
| 04 | [매 push 마다 배포 잡이 빨간불](troubleshooting/04-매-push마다-배포가-빨간불.md) |
| 05 | [moto 테스트가 테이블을 못 찾는다 — import 시점 문제](troubleshooting/05-moto-테스트가-테이블을-못-찾는다.md) |
| 06 | [데모 이미지가 실물이 아니었다](troubleshooting/06-데모-이미지가-실물이-아니었다.md) |

## 개념 노트

이 프로젝트에서 나온 재사용 가능한 지식은 [`infra-lab/aws-lab/`](../../infra-lab/aws-lab/README.md) 에 둔다.
**아직 안 썼다.** 후보는 아래.

- [ ] DynamoDB `scan`/`query` 1MB 한도와 `LastEvaluatedKey` — 에러 없이 잘리는 종류의 버그
- [ ] DynamoDB TTL 은 만료 시각 보장이 아니라 **삭제 예약**이다 (지연 최대 48시간)
- [ ] SQS fan-out 으로 Lambda 타임아웃을 우회하기 — 시간을 나누는 것이 아니라 **일감을 나눈다**
- [ ] DLQ + `maxReceiveCount` 와 "DLQ 에 메시지가 있으면 알람"
- [ ] boto3 `Decimal` — DynamoDB 숫자는 `float` 이 아니다
- [ ] SSRF 와 IMDS(`169.254.169.254`) 차단, 그리고 DNS rebinding 이 남기는 구멍
- [ ] dead man's switch — 정상을 **능동적으로 증명**하는 알림

기존 노트 중 이 프로젝트와 겹치는 것 —
[DynamoDB 단일 vs 멀티 테이블](../../infra-lab/aws-lab/DynamoDB-단일-vs-멀티-테이블.md) ·
[Lambda async invoke로 시간 분리](../../infra-lab/aws-lab/Lambda-async-invoke로-시간-분리.md) ·
[GitHub OIDC로 CDK 배포](../../infra-lab/aws-lab/GitHub-OIDC로-CDK-배포.md) (→ [결정 08](decisions/08-배포를-자동-push에서-수동-트리거로.md) 에서 **안 쓴** 이유)

---

## 숫자로 말할 것

| | |
|---|---:|
| 커밋 | 53개 (2026-05-14 ~ 2026-08-01, 작업일 11일) |
| 병합 커밋 | 2회 — 브랜치에서 작업 후 병합 (입력검증/테스트 도입 · fan-out 재설계) |
| Lambda | 6개 (register · health_check · health_check_worker · alert · report · heartbeat) |
| DynamoDB | 2 테이블 (endpoints · check-history, TTL 30일) |
| SQS | 4개 (check + alert, 각각 DLQ 한 쌍) |
| CloudWatch 알람 | 5개 (DLQ 2 · Lambda 오류 2 · **미호출 1**) |
| API 라우트 | 5개 (HTTP API, burst 20 / rate 10 스로틀링) |
| 테스트 함수 | 78개 (`def test_` 기준, 파일 8개 — moto 통합테스트 9개 포함) |
| 업그레이드 문서 | 9개 (`docs/upgrades/01~09`) |
| 헬스체크 주기 / HTTP 타임아웃 | 1분 / 10초 |
| 요약 알림 | 매일 1회 `cron(0 23 * * ? *)` = KST 08:00 |

**주의해서 말할 것**

- 테스트 78개는 **파일에서 센 `def test_` 개수**다. `pytest` 를 이 노트를 쓰면서 다시 돌리지는 않았다.
  CI(`test` 잡)에서는 ruff → pytest → terraform fmt → tfsec 순으로 돈다.
- **`incidents` 는 "장애 횟수"가 아니라 "DOWN 으로 기록된 체크 건수"다.**
  1분 주기에서 10분짜리 장애 하나가 `incidents: 10` 으로 나온다. 저장소 README 는 이걸
  "장애 횟수"라고 적고 있는데, 전환(transition) 을 세는 코드는 없다.
- **부하 테스트·동시성 테스트가 없다.** 엔드포인트 3개(Google·GitHub·Naver) 기준으로만 돌렸다.
  fan-out 이 실제로 N개에서 견디는지는 **설계상 그럴 것**이라는 근거뿐이다. → [결정 01](decisions/01-헬스체크를-enumerator와-worker로-분리.md)
- 비용을 집계하지 않았다. 프리티어 범위라고 **판단**했을 뿐 청구서로 확인한 값이 없다.
  [aws-serverless-agent](../aws-serverless-agent/README.md) 는 Phase 2 실비용 $0.07 을 확인했는데 여기선 안 했다.
- `tfsec` 은 `soft_fail: true` 라 **경고가 있어도 CI 는 초록불**이다. "IaC 보안 스캔이 있다"와
  "IaC 보안 스캔이 배포를 막는다"는 다르다. → [결정 09](decisions/09-tfsec을-soft-fail로.md)

---

## 아직 안 한 것과 이유

**TTL 30일과 월별 리포트의 경계가 안 맞는다** *(이 노트를 쓰며 코드에서 찾은 것 — 아직 안 고침)*

리포트는 매달 1일 00:00 UTC 에 **전월 전체**를 집계하는데, 이력 TTL 은 체크 시점 기준 **30일**이다.
31일인 달은 1일치 이력이 리포트가 도는 시점에 이미 만료 대상이다.
DynamoDB TTL 삭제는 만료 후 최대 48시간까지 지연되므로 **운이 좋으면 남아 있고 아니면 없다.**
지금까지 티가 안 난 이유는 데이터가 그만큼 안 쌓였기 때문이다.
→ 고치려면 TTL 을 35일로 올리거나 리포트를 매달 1일이 아니라 2~3일에 돌려야 한다.
→ [결정 03](decisions/03-체크-이력을-TTL-30일로.md)

**worker 요청 직전 SSRF 재검증**

SSRF 차단이 **등록 시점에만** 걸려 있다. 등록 때 공인 IP 로 해석되던 도메인이 나중에 사설 IP 로
바뀌면(DNS rebinding) worker 는 그대로 요청한다. `docs/upgrades/05` 의 체크박스에도 빈칸으로 남아 있다.
→ [결정 07](decisions/07-SSRF를-등록-시점에-차단.md)

**인증이 없다**

`POST /endpoints` 가 완전 공개다. 아무나 등록·삭제할 수 있다. 스로틀링(burst 20/rate 10)과
SSRF 가드로 **남용의 폭**만 줄였지 **누가** 하는지는 안 본다.
[gym-management-db](../gym-management-db/README.md) 에서 API Key 로 쓰기를 잠갔던 것과 같은 판단을
여기선 하지 않았다 — 그냥 안 넣었다.

**`incidents` 를 전환 기준으로 세기**

위 "주의해서 말할 것" 참고. 고치려면 이력을 시간순으로 훑어 `UP→DOWN` 전환만 세면 된다.
`compute_stats` 가 `report` 와 `heartbeat` 양쪽에 복제돼 있어 두 곳을 같이 고쳐야 한다.

**알림 채널이 Slack 웹훅 URL 하나에 묶여 있다**

`slack_webhook_url` 이 Terraform 변수(sensitive)로 들어가 Lambda 환경변수에 그대로 앉는다.
Secrets Manager / SSM 으로 빼지 않았다. [gym-management-db](../gym-management-db/troubleshooting/09-Secrets-Manager-조회-실패.md) 에서
한 번 겪은 주제인데 여기선 되풀이하지 않고 **그냥 변수로 뒀다.**

---

## 다른 프로젝트와의 대비

세 프로젝트가 **같은 질문에 다른 답을 낸 순서**로 읽힌다.

| | 검증 | 무너진 지점 |
|---|---|---|
| [gym-management-db](../gym-management-db/README.md) | 없었다 → 리뷰 받고 테스트 56개 | 4개월간 `\|\| true` 로 CI 가 무력 |
| [aws-serverless-agent](../aws-serverless-agent/README.md) | 매일 실배포 + 손으로 호출, 자동 테스트 0 | 완성 8일 뒤 재배포에서 전 요청 500 |
| **serverless-uptime-monitor** | **테스트 78개 + moto + CI 게이트가 있었다** | **그래도 프로덕션에서 조용히 멈췄다** |

앞의 둘이 *"검증이 없거나 자동화되지 않았다"* 였다면 이 프로젝트는 **검증이 다 있는데도 뚫렸다.**
테스트 픽스처가 엔드포인트를 문자열로만 채워서 `Decimal` 경로를 한 번도 안 지나갔기 때문이다.
→ [트러블 01](troubleshooting/01-헬스체크가-조용히-멈췄다.md)

**테스트는 내가 상상한 입력만 검증한다.** 그래서 그 위에 한 층이 더 필요했고,
그게 [요약 알림](decisions/05-요약-알림을-기존-알림-큐에-얹기.md)이었다 —
운영 중인 시스템이 **스스로 살아 있음을 보고**하게 만드는 것.
→ [세 프로젝트 비교](../README.md#세-프로젝트가-서로를-비춘다)
