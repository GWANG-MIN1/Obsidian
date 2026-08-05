# 02 Prometheus 아키텍처

Prometheus는 **대상에게 주기적으로 찾아가 메트릭을 긁어오는(pull) 시계열 데이터베이스**다.  
"에이전트가 서버로 보낸다(push)"는 기존 모니터링과 방향이 반대이고, 그 차이가 Kubernetes와 잘 맞는 이유이기도 하다.

---

## 전체 구조

```
                 ┌──────────────────────────────────────┐
                 │            Prometheus                │
  서비스          │  ┌────────────┐   ┌──────────────┐   │
  디스커버리 ────▶│  │  Retrieval  │──▶│    TSDB      │   │
  (K8s API)      │  │  (scrape)   │   │  (로컬 디스크) │   │
                 │  └─────┬───────┘   └──────┬───────┘   │
                 │        │                  │           │
                 │        │           ┌──────▼───────┐   │
                 │        │           │  Rule Engine │   │
                 │        │           └──────┬───────┘   │
                 │        │                  │           │
                 │        │           ┌──────▼───────┐   │
                 │        │           │  HTTP API    │◀──┼── Grafana / PromQL
                 └────────┼───────────┴──────┬───────┴───┘
                          │                  │
                 ┌────────▼────────┐   ┌─────▼────────┐
                 │  /metrics 엔드포인트│   │ Alertmanager │
                 │  (앱·exporter)   │   └──────────────┘
                 └─────────────────┘
```

| 구성요소 | 역할 |
|---|---|
| **Retrieval** | 타겟의 `/metrics`를 주기적으로 HTTP GET |
| **TSDB** | 시계열을 로컬 디스크에 저장 (블록 단위) |
| **Rule Engine** | 레코딩 룰 계산, 알림 룰 평가 |
| **HTTP API** | PromQL 질의 (Grafana가 여기에 붙는다) |
| **Alertmanager** | 발화한 알림의 라우팅·중복제거·발송 (**별도 프로세스**) |

> Alertmanager는 Prometheus에 포함된 것이 아니라 **독립 컴포넌트**다. Prometheus는 "알림이 발화했다"까지만 하고, 누구에게 어떻게 보낼지는 Alertmanager가 정한다. → `05-alerting/`

---

## Pull 모델

```
[ Push 방식 ]  앱 ──메트릭 전송──▶ 수집 서버
[ Pull 방식 ]  Prometheus ──HTTP GET /metrics──▶ 앱
```

| | Pull (Prometheus) | Push (StatsD, 전통적 APM) |
|---|---|---|
| 타겟 파악 | 수집 측이 목록을 안다 → **다운을 감지할 수 있다** | 안 오면 죽은 건지 안 보낸 건지 모른다 |
| 설정 위치 | 수집 서버 한 곳 | 모든 앱 |
| 방화벽 | 수집 → 타겟 방향 접근 필요 | 타겟 → 수집 방향 |
| 디버깅 | `curl localhost:9090/metrics`로 직접 확인 | 어렵다 |
| 짧게 사는 작업 | 불리 (배치 잡은 긁기 전에 끝난다) | 유리 |

> **Pull의 가장 큰 이점은 `up` 메트릭이 공짜로 생긴다는 것이다.** 긁으러 갔는데 응답이 없으면 `up=0`이 되어 다운을 즉시 안다. Push에서는 "안 온다"가 곧 "죽었다"인지 알 수 없다.  
> 짧게 사는 배치 잡은 **Pushgateway**로 예외 처리한다. 다만 남용하면 pull 모델의 이점이 사라지므로 진짜 배치에만 쓴다.

---

## 데이터 모델

```
http_requests_total{method="POST", handler="/api", status="200"}  1027  @1700000000
└──── 메트릭 이름 ────┘└────────── 라벨 ──────────┘               └값┘  └타임스탬프┘
```

**메트릭 이름 + 라벨 조합 하나 = 시계열 하나.** 라벨 값이 하나만 달라져도 별개의 시계열이다.

### ⚠️ 카디널리티 — Prometheus를 죽이는 것

```
http_requests_total{user_id="..."}        ← 사용자 10만 명 = 시계열 10만 개  💥
http_requests_total{path="/api/v1/u/123"} ← 경로마다 다른 ID = 무한 증가     💥
```

| 라벨로 써도 되는 것 | 라벨로 쓰면 안 되는 것 |
|---|---|
| method, status, handler(템플릿) | user_id, session_id, request_id |
| namespace, pod, container | 타임스탬프, UUID |
| env, region, cluster | 이메일, IP 전체 |

```
경험칙: 라벨 값의 가짓수는 수십~수백까지. 수천을 넘으면 설계를 다시 본다.
```

```bash
# 카디널리티가 폭발한 메트릭 찾기
curl -s localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName'
```

> 고유 식별자가 필요한 정보는 **메트릭이 아니라 로그**에 남긴다. 이게 3요소를 나누는 실질적 이유다. → `06-logging/`

---

## 메트릭 타입 4가지

| 타입 | 성질 | 예시 | 조회 방법 |
|---|---|---|---|
| **Counter** | 증가만 (재시작 시 0) | 요청 수, 에러 수 | 반드시 `rate()` |
| **Gauge** | 오르내림 | 메모리 사용량, 큐 길이 | 그대로 |
| **Histogram** | 버킷별 누적 분포 | 응답 시간 | `histogram_quantile()` |
| **Summary** | 클라이언트가 계산한 분위수 | 응답 시간 | 그대로 (집계 불가) |

```
# Counter — 이름은 관례상 _total
http_requests_total 1027

# Gauge
node_memory_MemAvailable_bytes 2147483648

# Histogram — _bucket / _sum / _count 세 벌이 생긴다
http_request_duration_seconds_bucket{le="0.1"}  240
http_request_duration_seconds_bucket{le="0.5"}  980
http_request_duration_seconds_bucket{le="+Inf"} 1000
http_request_duration_seconds_sum   87.3
http_request_duration_seconds_count 1000
```

> **카운터를 그대로 그리면 계단만 보인다.** 항상 `rate()`로 초당 증가율을 본다. → `03-promql/`  
> **Histogram vs Summary:** Summary는 파드별로 계산한 p99를 서버 간에 합칠 수 없다(평균의 평균 문제). 여러 인스턴스를 집계해야 하는 Kubernetes 환경에서는 **Histogram을 쓴다.**

### 명명 규칙

```
<네임스페이스>_<대상>_<단위>_<접미사>

http_requests_total                  # 카운터는 _total
http_request_duration_seconds        # 단위는 기본 단위(초, 바이트)
node_memory_MemAvailable_bytes       # _bytes, _seconds — 밀리초·MB 쓰지 않는다
```

---

## Exporter

메트릭을 직접 노출하지 않는 시스템(리눅스, MySQL, Redis…) 앞에 두는 번역기다.

| Exporter | 대상 | 포트 |
|---|---|---|
| **node-exporter** | 리눅스 호스트 (CPU·메모리·디스크·네트워크) | 9100 |
| **kube-state-metrics** | K8s 오브젝트 상태 (Deployment 레플리카 수 등) | 8080 |
| **cAdvisor** | 컨테이너 자원 사용량 (kubelet에 내장) | — |
| **blackbox-exporter** | 외부에서 HTTP·TCP·DNS 프로브 | 9115 |
| mysqld/redis/postgres-exporter | 각 DB | — |

> **kube-state-metrics vs cAdvisor**는 자주 헷갈린다.  
> `kube-state-metrics` = **오브젝트의 상태** ("Deployment가 3개를 원하는데 2개만 있다")  
> `cAdvisor` = **실제 자원 사용량** ("이 컨테이너가 CPU 0.3코어를 쓴다")

### 앱을 직접 계측하기

```python
from prometheus_client import Counter, Histogram, start_http_server

REQUESTS = Counter("http_requests_total", "총 요청 수", ["method", "status"])
LATENCY  = Histogram("http_request_duration_seconds", "응답 시간", ["handler"])

@LATENCY.labels(handler="/api").time()
def handle():
    REQUESTS.labels(method="GET", status="200").inc()

start_http_server(8000)     # /metrics 노출
```

---

## 서비스 디스커버리

정적 목록 대신 **인프라에 물어봐서 타겟을 자동으로 찾는다.** 파드가 뜨고 지는 Kubernetes에서 필수다.

```yaml
scrape_configs:
  # 정적 — 고정된 서버
  - job_name: node
    static_configs:
      - targets: ["10.0.1.10:9100", "10.0.1.11:9100"]

  # Kubernetes — API 서버에 물어본다
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod                    # pod / service / endpoints / node / ingress
    relabel_configs:
      # 어노테이션이 붙은 파드만 수집
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      # 어노테이션으로 포트 지정
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      # 네임스페이스를 라벨로
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

### relabeling — 수집 전에 라벨을 가공

```yaml
relabel_configs:      # 스크랩 '전' — 어떤 타겟을 긁을지 결정
  - action: keep      # 조건에 맞는 것만 남긴다
  - action: drop      # 조건에 맞는 것을 버린다
  - action: replace   # 라벨 값을 바꾼다
  - action: labelmap  # 라벨 이름을 일괄 변환

metric_relabel_configs:   # 스크랩 '후' — 어떤 메트릭을 저장할지 결정
  - source_labels: [__name__]
    regex: "go_.*"
    action: drop            # 불필요한 런타임 메트릭 버리기 (카디널리티 절감)
```

> **`__`로 시작하는 라벨은 내부용**이며 저장 전에 사라진다. `__address__`, `__meta_*`가 그것이다.  
> 쿠버네티스에서는 이 relabel 설정을 손으로 쓰는 대신 **Operator의 ServiceMonitor CRD**로 선언한다. → `08-kube-prometheus-stack/`

---

## TSDB와 리텐션

```
data/
├── wal/                       # Write-Ahead Log (최근 데이터, 크래시 복구용)
├── chunks_head/
└── 01H8X.../                  # 2시간 블록 → 점차 병합(compaction)
    ├── chunks/
    ├── index
    └── meta.json
```

```yaml
prometheus:
  prometheusSpec:
    retention: 7d               # 기간 기준
    retentionSize: 40GB         # 용량 기준 (둘 중 먼저 도달하는 쪽)
```

### 용량 어림잡기

```
필요 디스크 ≈ 시계열 수 × 초당 샘플 수 × 샘플당 바이트(약 1~2B) × 보존 기간

예) 100만 시계열, 15초 간격, 15일
    1,000,000 ÷ 15 × 1.5B × (15 × 86400) ≈ 130GB
```

> Prometheus는 **장기 보관용이 아니다.** 로컬 디스크에 저장하고 기본 보존은 15일이다.  
> 몇 달~몇 년을 봐야 하면 **Thanos·Mimir·Cortex**로 원격 저장소(S3)에 넘긴다. HA(이중화)도 이 계층이 담당한다.

```yaml
# 원격 저장소로 넘기기
remote_write:
  - url: http://mimir:9009/api/v1/push
```

---

## 실전 — EKS에서의 현실

관리형 Kubernetes에서는 **컨트롤 플레인을 긁을 수 없다.**

```yaml
# EKS에서는 반드시 꺼야 하는 것들
kubeControllerManager: { enabled: false }   # AWS 관리 영역, 접근 불가
kubeScheduler:         { enabled: false }
kubeEtcd:              { enabled: false }
kubeProxy:             { enabled: false }   # 메트릭이 127.0.0.1 에 바인딩됨
```

> 이걸 켜둔 채로 두면 **타겟이 영원히 빨갛고, 관련 알림이 계속 발화**한다. 그 상태가 며칠 지나면 팀 전체가 알림을 무시하게 된다 — 모니터링이 죽는 가장 흔한 경로다. → `01-observability-basics/`

```bash
# 타겟 상태 확인
curl -s localhost:9090/api/v1/targets \
  | jq '.data.activeTargets[] | select(.health!="up") | {job:.labels.job, lastError}'
```

---

## 배운 점

- Prometheus는 **pull 모델** — 타겟의 `/metrics`를 주기적으로 HTTP GET 한다
- pull의 이점은 **`up` 메트릭이 공짜**로 생겨 다운을 즉시 감지한다는 것
- 짧게 사는 배치 잡만 예외적으로 **Pushgateway**를 쓴다
- **메트릭 이름 + 라벨 조합 하나 = 시계열 하나**
- **카디널리티 폭발이 Prometheus를 죽인다** — user_id·request_id를 라벨로 쓰면 안 된다
- 고유 식별자가 필요하면 메트릭이 아니라 **로그**에 남긴다
- 메트릭 타입 4가지: Counter(→`rate()` 필수) · Gauge · Histogram · Summary
- **여러 인스턴스를 집계해야 하면 Summary가 아니라 Histogram**을 쓴다
- 단위는 기본 단위(초·바이트), 카운터는 `_total` 접미사
- `kube-state-metrics`=오브젝트 상태, `cAdvisor`=실제 자원 사용량
- 서비스 디스커버리 + relabeling으로 타겟을 자동으로 찾고 가공한다
- `metric_relabel_configs`로 불필요한 메트릭을 버려 카디널리티를 줄인다
- **Prometheus는 장기 보관용이 아니다** — 길게 보려면 Thanos·Mimir로 원격 저장
- **EKS는 컨트롤 플레인을 스크랩할 수 없다** — 관련 타겟·룰을 반드시 끈다
