# 04. "마지막 체크 결과"와 "확정된 알림 상태"를 분리

> 메모: 한 번의 타임아웃으로 알림이 튀는 것을 막으려다, `status` 필드 하나가
> 두 가지 질문에 답하고 있다는 걸 알게 됐다. upgrade-06, 2026-07-10 커밋 `ff6d314`

---

## 상황

원래 알림 판정은 이랬다.

```python
if previous_status != "DOWN" and status == "DOWN":
    send_alert()
```

`status` 하나로 **중복 알림 방지**까지 해결한다. 짧고 잘 돈다.
문제는 **네트워크 지터로 한 번 타임아웃이 나면 그대로 DOWN 알림이 간다**는 것이다.
1분 주기라 하루에 몇 번씩 튈 수 있고, 그러면 알림 피로로 **정작 중요한 DOWN 을 놓친다.**

"연속 2회 실패해야 알린다"를 넣으려는데, 여기서 막혔다 —
`status` 를 "확정된 상태"로 쓰면 **마지막 체크 결과를 기록할 곳이 없어진다.**
그러면 uptime% 통계가 틀어진다.

## 결정

**필드 하나를 두 개로 쪼갠다.**

| 필드 | 뜻 | 쓰는 곳 |
|---|---|---|
| `status` | **마지막 체크의 raw 결과** (UP/DOWN) | 히스토리, uptime% 통계, 대시보드 |
| `alert_state` | **확정된 알림 상태** (UP/DOWN) | 알림 발송 여부 판정 |
| `consecutive_failures` | 연속 DOWN 횟수. UP 이면 0 | 임계값 비교 |

```python
prev_state = _confirmed_state(endpoint)            # alert_state (없으면 status 로 추정)
prev_failures = int(endpoint.get("consecutive_failures") or 0)
consecutive_failures = prev_failures + 1 if status == "DOWN" else 0

fire_down = (
    status == "DOWN"
    and consecutive_failures >= FAILURE_THRESHOLD
    and prev_state != "DOWN"
)
fire_recovered = status == "UP" and prev_state == "DOWN"
```

판정 규칙

- **DOWN 알림** — 임계값 도달 **and** 아직 확정 DOWN 이 아닐 때 → 알림 + `down_since` 기록
- **RECOVERED 알림** — 확정 DOWN 이었는데 UP 이 관측될 때 (임계값 무관, 복구는 즉시)
- **임계값에 못 미친 채 UP 으로 돌아오면** → 알림 없이 카운터만 0으로

## 왜

- **`status` 를 건드리지 않아서 통계가 안 틀어진다.** 히스토리는 raw 결과를 그대로 받고,
  uptime% 는 이전과 똑같이 계산된다. **알림 여부만** 분리됐다.
- **복구는 비대칭으로 다뤄야 한다.** 실패는 두 번 확인하고, 복구는 한 번에 인정한다.
  "아직 완전히 복구된 게 아닐 수도 있다"고 미루면 **RECOVERED 가 영원히 안 갈 수도** 있다.
- **`down_since` 를 확정 시점에 기록**하므로 장애 지속시간이 "알림을 보낸 순간부터"로
  일관된다. 첫 실패부터 재면 임계값 대기 시간이 장애 시간에 섞인다.

## 하위호환을 어떻게 했나

`alert_state` 는 이번에 **새로 생긴 필드**다. 기존 데이터엔 없다.

```python
def _confirmed_state(endpoint):
    state = endpoint.get("alert_state")
    if state in ("UP", "DOWN"):
        return state
    return "DOWN" if endpoint.get("status") == "DOWN" else "UP"   # raw status 로 추정
```

그리고 **코드 기본값을 1로 뒀다.**

```python
FAILURE_THRESHOLD = max(int(os.environ.get("FAILURE_THRESHOLD", "1")), 1)
```

`1` 이면 첫 DOWN 즉시 알림 — **기존 동작과 완전히 같다.**
flap 방지를 켜는 것은 Terraform 변수(`failure_threshold = 2`)뿐이다.
**코드는 기존 동작을, 인프라가 새 동작을 선택한다.** 롤백이 변수 하나로 끝난다.

## 왜 다른 건 안 썼나

**`status` 하나로 계속 가고, 임계값 미달이면 `status` 를 안 바꾼다**

가장 적게 고치는 방법이고 처음에 이걸 하려 했다. 그런데 **마지막 체크 결과를 잃는다.**
DOWN 이 한 번 관측됐는데 테이블엔 UP 으로 남으니, 대시보드가 거짓말을 하고
[결정 06](06-살아있음을-status-대신-신선도로.md) 의 stale 판정과도 어긋난다.

**최근 N개 이력을 query 해서 판정**

카운터를 안 들고 다녀도 되고 "최근 5분 중 3회 실패" 같은 유연한 규칙도 된다.
대신 **체크마다 query 가 한 번씩 더 늘고**, worker 가 자기 엔드포인트의 이력을 읽어야 한다.
카운터 하나를 `update_item` 에 얹는 쪽이 읽기 0회다.

**CloudWatch 알람의 `evaluation_periods` 에 맡긴다**

알람은 이미 그런 기능이 있다. 하지만 **알람은 지표(metric)에 걸리고, 우리가 알리려는 것은
엔드포인트 단위 상태**다. 엔드포인트마다 지표와 알람을 만들면
[결정 01](01-헬스체크를-enumerator와-worker로-분리.md) 에서 버린 "엔드포인트마다 인프라"와 같은 문제가 된다.

## 결과

테스트로 고정한 시나리오 (`tests/test_health_check_worker.py`)

- threshold=2: 첫 DOWN 은 **알림 없이** `consecutive_failures=1`
- threshold=2: 두 번째 연속 DOWN 에서 알림 + `alert_state=DOWN`
- 확정 DOWN 에서 UP → RECOVERED (임계값 무관)
- 임계값 미달 후 UP → 알림 없이 카운터만 리셋
- 이미 확정 DOWN 이면 실패가 이어져도 중복 알림 없음

## 배운 점

**한 필드가 두 질문에 답하고 있으면, 언젠가 둘 중 하나를 포기하게 된다.**
"지금 어떤가"와 "무엇을 알렸는가"는 다른 질문이다. 처음엔 같은 값이라 안 보였을 뿐이다.
