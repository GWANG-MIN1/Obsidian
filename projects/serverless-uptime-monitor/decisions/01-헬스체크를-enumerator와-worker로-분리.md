# 01. 헬스체크를 enumerator + worker fan-out 으로 분리

> 메모: 1단계에서 만든 단일 람다가 엔드포인트를 순차로 돌았다. N개 × 타임아웃 10초가
> 한 람다의 실행시간에 그대로 쌓인다. 2026-06-27, 커밋 `e7cf322`

---

## 상황

1단계(5월)의 `health_check` 는 이렇게 생겼다.

```python
def lambda_handler(event, context):
    endpoints = scan_all()
    for endpoint in endpoints:
        check_endpoint(endpoint)     # HTTP GET, 타임아웃 10초
```

EventBridge 가 **1분마다** 호출한다. 엔드포인트가 늘어나면 —

| 엔드포인트 | 최악 실행시간 | |
|---:|---:|---|
| 3개 | 30초 | 데모 범위 |
| 6개 | 60초 | **다음 호출과 겹친다** |
| 20개 | 200초 | 람다 기본 타임아웃 초과 |

응답이 느린(=타임아웃까지 가는) 엔드포인트일수록 시간을 많이 먹는다.
**모니터링이 필요한 상황일수록 모니터가 먼저 죽는 구조**다.

## 결정

`health_check` 를 **enumerator** 로 축소하고, 실제 체크를 **worker** 로 뺀다.
둘 사이는 SQS 로 잇는다.

```
EventBridge (1분)
      ↓
health_check ── scan 전체 페이지 → 엔드포인트 1건씩 check_queue 로 send
      ↓  SQS check_queue (DLQ, maxReceiveCount 2, visibility 35s)
health_check_worker ── 메시지 1건 = 엔드포인트 1개, Lambda 가 병렬로 뜬다
      ↓  SQS alert_queue
alert
```

enumerator 는 HTTP 요청을 **하나도 안 한다.** DynamoDB scan 과 SQS send 만 한다.
엔드포인트가 몇 개든 실행시간이 거의 변하지 않는다.

## 왜

- **시간을 늘리는 대신 일감을 쪼갠다.** 람다 타임아웃을 900초로 올리는 것은
  같은 문제를 뒤로 미루는 것이다. fan-out 은 엔드포인트 수와 실행시간을 **분리**한다.
- **한 엔드포인트의 실패가 다른 엔드포인트를 막지 않는다.** 순차 루프에서는
  중간 하나가 예외를 던지면 뒤가 통째로 안 돌았다. 메시지 단위로 쪼개면 실패도 1건에 갇힌다.

  > 이 이점은 2026-08-01의 [트러블 01](../troubleshooting/01-헬스체크가-조용히-멈췄다.md) 에서
  > **역설적으로 뒤집혔다.** worker 는 격리됐지만 **enumerator 는 여전히 단일 장애점**이라,
  > 거기서 죽으니 전부 멈췄다. 쪼갠 것은 실행이지 책임이 아니다.

- **재시도가 공짜로 따라온다.** SQS `maxReceiveCount = 2`, 넘으면 DLQ.
  DLQ 에 메시지가 쌓이면 CloudWatch 알람이 SNS 로 알린다 (`check_dlq_not_empty`).
- **visibility timeout 35초**를 HTTP 타임아웃 10초보다 넉넉히 잡아, 처리 중인 메시지가
  다른 worker 에게 다시 배달되는 것을 막았다.

## 왜 다른 건 안 썼나

**람다 타임아웃을 늘린다**

가장 싼 수정이고 지금 당장은 통한다. 하지만 엔드포인트 수가 늘면 다시 같은 벽에 부딪히고,
그때는 **1분 주기 자체를 못 지키게** 된다. 주기가 밀리면 uptime 통계가 왜곡된다.

**엔드포인트마다 EventBridge 규칙을 하나씩**

병렬성은 얻지만 **등록 API 가 인프라를 만들게 된다.** 런타임 코드가 EventBridge 규칙을
생성/삭제해야 하고, Terraform 이 관리하는 상태와 실제 리소스가 갈라진다.
계정당 규칙 개수 한도에도 걸린다.

**enumerator 안에서 스레드/asyncio 로 병렬 요청**

코드 한 파일로 끝나서 매력적이다. 그런데 **여전히 람다 하나의 실행시간 안**이고,
동시 요청 수를 람다 메모리로 조절해야 한다. SQS 를 쓰면 그 조절을 Lambda 동시성
설정으로 밀 수 있고, 실패한 1건만 재시도된다.

**Step Functions Map**

정공법이지만 이 규모에서 상태머신 정의·비용·러닝커브가 SQS 한 줄보다 무겁다.
[aws-serverless-agent 결정 04](../../aws-serverless-agent/decisions/04-API와-Worker를-async-invoke로-분리.md) 에서
같은 종류의 판단을 했다 — 거기선 SQS 도 안 쓰고 Lambda async invoke 로 끝냈다.
**"큐가 필요한가"를 매번 다시 묻는다는 점이 같다.**

## 아직 검증 안 한 것

**엔드포인트를 실제로 늘려 보지 않았다.** 데모는 3개(Google·GitHub·Naver)다.
fan-out 이 N개에서 실제로 견디는지는 **설계상 그럴 것**이라는 근거뿐이고,
worker 동시성 한도나 DynamoDB 쓰기 스로틀링을 만나 본 적이 없다.
