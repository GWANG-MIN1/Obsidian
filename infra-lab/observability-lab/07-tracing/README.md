# 07 분산 추적 (Tracing)

마이크로서비스에서 "요청이 3초 걸렸다"는 사실만으로는 아무것도 못 한다. **어느 서비스의 어느 구간에서 걸렸는지**를 알아야 한다.  
분산 추적은 요청 하나에 ID를 붙여 서비스 경계를 넘어 따라다니게 하고, 전체 여정을 하나의 타임라인으로 재구성한다.

---

## 왜 필요한가

```
[ 메트릭만 ]
  주문 API p99 = 3초   ← 느리다는 건 안다. 어디서? 모른다.

[ 트레이스 ]
  주문 API 3.2s
  ├─ 인증 서비스        0.05s
  ├─ 재고 서비스        0.12s
  ├─ 결제 서비스        2.90s   ← 여기
  │   └─ 외부 PG API   2.85s   ← 실은 여기
  └─ 알림 발행          0.08s
```

> 서비스가 3개면 로그를 눈으로 맞춰볼 수 있다. **10개가 넘어가면 불가능해진다.** 그 지점부터 트레이싱이 선택이 아니게 된다.

---

## Trace와 Span

```
Trace (요청 하나의 전체 여정) = trace_id 로 묶인 Span들의 트리

Span: 하나의 작업 단위
  ├─ trace_id     이 요청 전체의 ID (모든 span이 공유)
  ├─ span_id      이 작업의 ID
  ├─ parent_id    누가 나를 호출했는가
  ├─ name         "GET /api/orders", "SELECT orders"
  ├─ start / end  시작·종료 시각 → duration
  ├─ attributes   http.method, db.system, user.tier ...
  └─ status       OK / ERROR
```

```
시간 ────────────────────────────────────────▶
[───────── GET /api/orders (3.2s) ─────────]   root span
   [─ auth (0.05s) ─]
      [── inventory (0.12s) ──]
         [────── payment (2.9s) ──────]
            [───── PG API (2.85s) ────]
```

| 개념 | 의미 |
|---|---|
| **Trace** | 요청 하나 = span들의 트리 |
| **Span** | 작업 하나 (HTTP 호출, DB 쿼리, 큐 발행) |
| **Root span** | parent가 없는 첫 span |
| **Attribute** | span에 붙는 키-값 (라벨과 비슷하지만 카디널리티 제약이 없다) |

> 트레이스는 **카디널리티 제약이 없다.** user_id·request_id를 마음껏 넣을 수 있다 — 시계열이 아니라 개별 이벤트로 저장되기 때문이다. 메트릭에 못 넣는 정보를 여기 넣는다.

---

## 컨텍스트 전파 — 핵심 메커니즘

trace_id가 서비스 경계를 넘어야 하나의 트레이스가 된다. **HTTP 헤더로 전달**한다.

```
서비스 A ──HTTP──▶ 서비스 B ──HTTP──▶ 서비스 C
   │  traceparent: 00-<trace_id>-<span_id>-01
   └─ W3C Trace Context 표준 헤더
```

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  └────── trace_id (16바이트) ────┘└─ span_id ──┘ └flags
             version
```

> **전파가 끊기면 트레이스가 조각난다.** 한 서비스가 헤더를 전달하지 않으면 거기서부터 별개의 트레이스가 되어 아무것도 못 본다.  
> 메시지 큐를 거칠 때도 메시지 헤더에 컨텍스트를 실어야 이어진다. 비동기 경계가 가장 자주 끊기는 지점이다.

---

## OpenTelemetry (OTel)

벤더 중립 표준. **계측 코드를 한 번 쓰면 백엔드(Jaeger·Tempo·Datadog)를 바꿔도 그대로 쓴다.**

```
   앱 (OTel SDK)
        │  OTLP 프로토콜
        ▼
  OTel Collector          ← 수집·가공·분배 계층
        │
   ┌────┼────┬──────┐
   ▼    ▼    ▼      ▼
 Tempo Jaeger Prometheus  Loki
(트레이스)      (메트릭)   (로그)
```

| 구성요소 | 역할 |
|---|---|
| **API/SDK** | 앱에서 span 생성 (언어별 라이브러리) |
| **자동 계측** | 프레임워크·HTTP 클라이언트·DB 드라이버를 자동으로 감싼다 |
| **Collector** | 수신 → 가공(필터·샘플링·속성 추가) → 여러 백엔드로 전송 |
| **OTLP** | 표준 전송 프로토콜 |

> **OTel은 트레이스만이 아니라 메트릭·로그도 다룬다.** 3요소를 하나의 계측 표준으로 통일하는 것이 목표다.  
> Collector를 두면 **앱은 한 곳으로만 보내면 되고**, 백엔드 교체·샘플링 정책 변경이 앱 재배포 없이 가능해진다.

### 자동 계측부터 시작한다

```bash
# Python
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
OTEL_SERVICE_NAME=order-api \
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317 \
  opentelemetry-instrument python app.py
```

```java
// Java — 에이전트만 붙이면 코드 변경 0
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=order-api \
     -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
     -jar app.jar
```

> 자동 계측만으로 **HTTP 요청·DB 쿼리·외부 호출**이 전부 span으로 잡힌다. 대부분의 병목은 여기서 보인다.  
> 수동 계측은 비즈니스 로직 구간을 나눌 때만 추가한다.

```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("calculate_discount") as span:
    span.set_attribute("user.tier", user.tier)
    span.set_attribute("cart.items", len(cart))
    result = calculate(cart)
```

---

## 샘플링 — 전부 저장할 수 없다

초당 1만 요청을 전부 저장하면 트레이스 저장 비용이 로그를 넘는다.

| 방식 | 동작 | 장단점 |
|---|---|---|
| **Head 샘플링** | 요청 시작 시 확률로 결정 (1%) | 간단·저렴. **느린 요청을 놓친다** |
| **Tail 샘플링** | 트레이스 완료 후 판단 | 에러·느린 것만 저장 가능. Collector 메모리 필요 |

```yaml
# OTel Collector — tail 샘플링
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors                    # 에러는 전부
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow                      # 1초 넘으면 전부
        type: latency
        latency: { threshold_ms: 1000 }
      - name: baseline                  # 나머지는 1%
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }
```

> **Head 샘플링 1%는 "느린 요청 100건 중 1건"만 남긴다.** 정작 보고 싶은 게 느린 요청인데.  
> 그래서 실무에서는 **에러·느린 것은 전부 + 정상은 소량**의 tail 샘플링을 쓴다. 단, 트레이스가 끝날 때까지 Collector가 들고 있어야 하므로 메모리를 본다.

---

## 백엔드

| 도구 | 특징 |
|---|---|
| **Tempo** (Grafana) | 오브젝트 스토리지 기반, 인덱스 최소화 (**Loki의 트레이스 버전**) |
| **Jaeger** | CNCF, 오래된 표준. UI가 성숙 |
| Zipkin | 가장 오래됨, 가볍다 |

> Grafana를 이미 쓰고 있다면 **Tempo가 자연스럽다.** 같은 UI에서 메트릭·로그·트레이스를 오갈 수 있다.  
> Loki와 같은 철학이다 — trace_id로 조회하는 게 주 사용 패턴이니 전문 인덱싱을 하지 않는다.

---

## 세 요소를 잇기

관측성의 실제 가치는 **셋을 오갈 수 있을 때** 나온다.

```
메트릭 (p99 스파이크)
   │  exemplar — 그 시점의 실제 trace_id 를 메트릭에 첨부
   ▼
트레이스 (payment 구간이 2.9초)
   │  span 에 기록된 trace_id 로 로그 조회
   ▼
로그 ("connection pool exhausted")
```

### Exemplar

히스토그램 버킷에 **대표 trace_id를 붙여두는 기능**이다. Grafana에서 그래프 위 점을 클릭하면 그 요청의 트레이스로 넘어간다.

```
http_request_duration_seconds_bucket{le="2.5"} 1027 # {trace_id="4bf92f35..."} 2.3 1700000000
                                                     └────── exemplar ──────┘
```

### 로그에 trace_id 넣기

```json
{"level":"error","msg":"db query failed","trace_id":"4bf92f3577b34da6..."}
```

```logql
{namespace="payment"} | json | trace_id="4bf92f3577b34da6a3ce929d0e0e4736"
```

> **로그에 `trace_id`를 넣는 것이 가장 저렴한 연결 고리다.** 트레이싱 백엔드를 아직 안 붙였더라도, 지금부터 로그에 trace_id를 남겨두면 나중에 그대로 이어진다.  
> Grafana의 데이터소스 설정에서 **derived field**로 로그의 trace_id를 Tempo 링크로 자동 변환할 수 있다.

---

## 도입 순서

```
1. 로그에 trace_id 넣기               ← 코드 변경 최소, 즉시 효과
2. 자동 계측 붙이기 (에이전트/SDK)      ← 대부분의 병목이 여기서 보인다
3. Collector + Tempo 배포             ← 저장·조회
4. 샘플링 정책 (tail: 에러·느린 것 전부)
5. 필요한 곳만 수동 계측
6. exemplar 로 메트릭 ↔ 트레이스 연결
```

> **처음부터 전부 하려 하지 않는다.** 서비스가 3~4개면 트레이싱보다 구조화 로그가 비용 대비 효과가 크다.  
> 트레이싱이 꼭 필요해지는 시점은 **"어느 서비스가 원인인지 회의로 정하고 있을 때"** 다.

---

## 배운 점

- 트레이싱은 "얼마나 느린가"가 아니라 **"어디서 느린가"** 에 답한다
- **Trace = trace_id로 묶인 Span 트리**, Span = 작업 하나 (HTTP·DB·큐)
- 트레이스는 **카디널리티 제약이 없다** — user_id 같은 값을 마음껏 붙일 수 있다
- 서비스 경계를 넘는 건 **컨텍스트 전파**(W3C `traceparent` 헤더)
- **전파가 끊기면 트레이스가 조각난다** — 비동기·메시지 큐 경계가 가장 위험
- **OpenTelemetry는 벤더 중립 표준** — 계측을 한 번 하면 백엔드를 바꿔도 된다
- Collector를 두면 앱은 한 곳으로만 보내고, 샘플링·백엔드 변경이 앱과 분리된다
- **자동 계측만으로도 대부분의 병목이 보인다** — 수동 계측은 필요한 곳만
- Head 샘플링 1%는 **정작 보고 싶은 느린 요청을 놓친다**
- 실무는 **tail 샘플링** — 에러·느린 건 전부, 정상은 소량
- Grafana를 쓴다면 **Tempo**가 자연스럽다 (Loki와 같은 철학: 인덱스 최소화)
- **exemplar**로 메트릭 그래프에서 트레이스로 바로 점프할 수 있다
- **가장 저렴한 첫걸음은 로그에 `trace_id`를 넣는 것** — 나중에 그대로 이어진다
- 서비스가 몇 개 안 되면 트레이싱보다 구조화 로그가 먼저다
