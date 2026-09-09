# 02. 알림을 직접 호출 대신 SQS 큐 경유로

> 메모: worker 가 Slack/SNS 를 직접 부르지 않는다. `alert_queue` 에 메시지만 넣고 끝낸다.
> 1단계(2026-05-14)부터 이 구조였고, 나중에 요약 알림이 여기 무임승차한다.

---

## 상황

장애를 감지한 것은 헬스체크 쪽이고, 알림을 보내는 것은 Slack 웹훅과 SNS 다.
이 둘을 어떻게 이을 것인가.

가장 짧은 코드는 감지한 자리에서 바로 부르는 것이다.

```python
if previous_status != "DOWN" and status == "DOWN":
    urllib.request.urlopen(SLACK_WEBHOOK_URL, ...)   # 외부 HTTP
    sns.publish(...)                                  # 외부 API
```

## 결정

worker 는 **SQS 에 메시지를 넣는 것으로 끝낸다.** 별도의 `alert` 람다가 그 큐를 소비해
Slack 과 SNS 로 보낸다.

```
health_check_worker ──→ SQS alert_queue ──→ alert Lambda ──→ Slack Webhook
                              │                           └→ SNS Topic → Email
                              └→ alert_dlq (maxReceiveCount 3)
```

메시지 포맷은 `event_type` 하나로 갈린다 — `DOWN` / `RECOVERED` / `SUMMARY`.

## 왜

- **체크와 알림의 실패가 서로 옮지 않는다.** Slack 웹훅이 죽어도 헬스체크는 계속 돈다.
  직접 호출이면 웹훅 지연이 그대로 worker 실행시간이 되고, 최악엔 체크가 밀린다.
- **재시도 정책이 다르다.** 헬스체크는 1분 뒤 어차피 다시 돈다 — 재시도가 별 의미 없다.
  **알림은 다르다.** 놓치면 그 장애는 영원히 안 알려진다. 그래서 alert 큐만
  `maxReceiveCount = 3`, 보존 24시간(체크 큐는 2회 / 5분)으로 다르게 잡았다.

  | | maxReceiveCount | message_retention |
  |---|---:|---:|
  | `check_queue` | 2 | 300초 |
  | `alert_queue` | 3 | 86,400초 |

- **보낼 곳을 나중에 늘릴 수 있다.** 실제로 그렇게 됐다 — 4단계에서 `heartbeat` 람다가
  **같은 큐에 `SUMMARY` 를 넣는 것만으로** Slack·SNS·IAM 을 하나도 안 건드리고 붙었다.
  → [결정 05](05-요약-알림을-기존-알림-큐에-얹기.md)
- **IAM 이 좁아진다.** worker 는 SQS `SendMessage` 만 있으면 된다. SNS publish 권한도,
  Slack 웹훅 URL 도 worker 에 없다.

## 왜 다른 건 안 썼나

**worker 에서 Slack/SNS 직접 호출**

람다 하나와 큐 두 개(alert + DLQ)를 안 만들어도 된다. 대신 위의 네 가지를 전부 잃는다.
특히 **알림 발송 실패를 재시도할 자리가 없다** — worker 가 예외를 던지면 체크 결과 기록까지
같이 재시도되어 히스토리가 중복된다.

**SNS 를 먼저 두고 Slack 을 SNS 구독으로**

SNS → Slack 은 Chatbot 이나 별도 Lambda 가 또 필요하고, **Slack 메시지 포맷(attachments,
색상)을 SNS 가 못 만든다.** 지금은 `alert` 람다 하나가 Slack 용 payload 와 SNS 용
평문을 각각 만든다.

**EventBridge 이벤트 버스**

라우팅 규칙으로 `event_type` 별 대상을 나눌 수 있어 더 "정석"이다. 그런데 지금은
목적지가 **항상 둘 다**(Slack + 이메일)라 라우팅할 게 없다. 큐 하나가 더 단순하다.

## 결과

메시지 형태만 맞추면 새 알림 종류가 인프라 변경 없이 들어온다는 것이,
`SUMMARY` 를 붙일 때 실제로 확인됐다. `terraform/sns.tf` 는 9줄짜리 파일이고
4단계 내내 한 번도 안 바뀌었다.
