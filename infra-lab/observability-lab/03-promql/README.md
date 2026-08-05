# 03 PromQL

PromQL은 시계열을 조회·가공하는 질의 언어다. 대시보드도 알림도 결국 PromQL 한 줄이다.  
문법 자체는 작지만, **"카운터는 반드시 `rate()`"** 와 **"집계 시 라벨이 어떻게 되는가"** 두 가지만 잡으면 대부분이 풀린다.

---

## 두 가지 결과 타입

```
Instant Vector (순간 벡터)     시계열마다 '한 시점의 값 하나'
  up
  → up{job="node"} 1

Range Vector (구간 벡터)       시계열마다 '구간 안의 여러 값'
  up[5m]
  → up{job="node"} 1 @t1, 1 @t2, 0 @t3 ...
```

| | 쓰임 |
|---|---|
| Instant Vector | 그래프에 그릴 수 있다 |
| Range Vector | **그대로는 못 그린다.** `rate()` 같은 함수를 거쳐야 한다 |

> `rate(http_requests_total)` → 에러. `rate()`는 구간 벡터를 받으므로 `rate(http_requests_total[5m])`이어야 한다.  
> 반대로 `http_requests_total[5m]`만 쓰면 그래프에 못 올린다. **초보자가 가장 많이 만나는 두 에러다.**

---

## 셀렉터

```promql
up                                  # 메트릭 전체
up{job="node-exporter"}             # 정확히 일치
up{job!="kubelet"}                  # 불일치
up{job=~"node.*"}                   # 정규식 일치
up{job!~"kube.*"}                   # 정규식 불일치
up{job="node", instance=~"10\\.0.*"}  # 여러 조건 (AND)

http_requests_total[5m]             # 최근 5분 구간
http_requests_total offset 1h       # 1시간 전 시점
http_requests_total[5m] offset 1d   # 하루 전의 5분 구간 (전일 대비 비교)
```

| 단위 | s(초) m(분) h(시) d(일) w(주) y(년) |
|---|---|

---

## 카운터 — rate / irate / increase

카운터는 계속 증가만 하므로 **그대로 그리면 우상향 계단**만 보인다. 의미 있는 건 "얼마나 빨리 증가하는가"다.

```promql
rate(http_requests_total[5m])       # 구간 평균 증가율 (초당) ← 기본으로 쓴다
irate(http_requests_total[5m])      # 마지막 두 샘플만 본 순간 증가율
increase(http_requests_total[1h])   # 구간 동안의 총 증가량 (= rate × 구간초)
```

| 함수 | 특징 | 언제 |
|---|---|---|
| **`rate`** | 부드럽다, 노이즈에 강하다 | **알림·SLO·대시보드 대부분** |
| `irate` | 민감하다, 급변을 잘 잡는다 | 짧은 스파이크 조사 |
| `increase` | 사람이 읽기 쉬운 총량 | "지난 1시간 에러 몇 건" |

> **알림에는 `irate`를 쓰지 않는다.** 순간값이라 튀어서 오탐이 난다.  
> 세 함수 모두 **카운터 리셋(재시작으로 0이 되는 것)을 자동으로 보정**해준다. 그래서 카운터에는 직접 뺄셈을 하지 않는다.

### 구간은 스크랩 간격의 4배 이상

```
스크랩 간격 15s  →  rate(...[1m])   샘플 4개. 최소선
                 →  rate(...[5m])   샘플 20개. 안정적 ← 권장
                 →  rate(...[30s])  샘플 2개. 하나만 빠져도 값이 사라진다
```

---

## 집계 연산자

```promql
sum(rate(http_requests_total[5m]))                     # 전부 합쳐 하나로
sum by (pod) (rate(http_requests_total[5m]))           # pod 별로 (지정한 라벨만 남긴다)
sum without (instance) (rate(http_requests_total[5m])) # instance 라벨만 버린다
```

```
by      → 나열한 라벨만 남긴다        (화이트리스트)
without → 나열한 라벨만 버린다        (블랙리스트)
```

| 연산자 | 용도 |
|---|---|
| `sum` | 합계 — 여러 인스턴스의 요청 수 |
| `avg` | 평균 — CPU 사용률 |
| `max` / `min` | 최대·최소 |
| `count` | 시계열 개수 — `count(up == 0)` = 다운된 타겟 수 |
| `topk(k, ...)` / `bottomk` | 상위·하위 k개 |
| `stddev` / `quantile` | 분포 |

```promql
# CPU 상위 5개 파드
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))

# 다운된 타겟 수
count(up == 0)

# 네임스페이스별 파드 수
count by (namespace) (kube_pod_info)
```

> **집계 순서가 중요하다.** `sum(rate(...))`는 맞지만 `rate(sum(...))`은 틀렸다.  
> `sum`을 먼저 하면 파드 재시작 시 카운터 리셋이 뒤섞여 값이 튄다. **항상 `rate`를 먼저, `sum`을 나중에.**

---

## 실전 계산식

### CPU 사용률

```promql
# 노드
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 파드 (코어 수)
sum by (pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# 파드의 CPU limit 대비 사용률
sum by (pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
  / sum by (pod) (kube_pod_container_resource_limits{resource="cpu"}) * 100
```

> CPU 사용률을 직접 재는 메트릭은 없다. **idle 시간의 증가율을 빼서** 구한다.

### 메모리·디스크

```promql
# 메모리 사용률 (%)
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# 디스크 사용률 (%)
100 * (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}
         / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})

# 4시간 뒤 디스크가 찰 것인가 (선형 예측)
predict_linear(node_filesystem_avail_bytes[6h], 4*3600) < 0
```

> `predict_linear`는 **"이미 꽉 찼다"가 아니라 "이대로면 곧 찬다"** 를 알린다. 대응할 시간을 준다는 점에서 임계값 알림보다 낫다.

### RED — 에러율과 지연

```promql
# 에러율 (%)
100 * sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))

# p95 지연
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# 핸들러별 p99
histogram_quantile(0.99,
  sum by (le, handler) (rate(http_request_duration_seconds_bucket[5m]))
)
```

> **`histogram_quantile`에는 반드시 `le` 라벨이 남아 있어야 한다.** `sum by (le)`를 빼먹으면 결과가 NaN이 된다. 가장 흔한 실수다.  
> 순서도 고정이다: `rate` → `sum by (le)` → `histogram_quantile`.

### Kubernetes 상태

```promql
# 재시작한 파드
increase(kube_pod_container_status_restarts_total[1h]) > 0

# 원하는 레플리카 수를 못 채운 Deployment
kube_deployment_spec_replicas != kube_deployment_status_replicas_available

# Pending 파드
kube_pod_status_phase{phase="Pending"} == 1

# 노드 NotReady
kube_node_status_condition{condition="Ready", status="true"} == 0
```

---

## 벡터 매칭 (연산자 주의)

두 메트릭을 나누거나 더할 때, **라벨 집합이 같아야 매칭된다.**

```promql
# 라벨이 완전히 같아야 매칭 (기본)
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# 라벨이 다를 때 — 기준 라벨 지정
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))
  / on (pod) group_left
sum by (pod) (kube_pod_container_resource_limits{resource="cpu"})

# 특정 라벨만 무시하고 매칭
metric_a / ignoring(instance) metric_b
```

| 키워드 | 의미 |
|---|---|
| `on (라벨)` | 이 라벨들로만 매칭 |
| `ignoring (라벨)` | 이 라벨들을 무시하고 매칭 |
| `group_left` / `group_right` | 다대일 매칭에서 어느 쪽이 "다"인지 |

> 결과가 **비어 있으면 대부분 라벨 불일치**다. 양쪽 쿼리를 따로 실행해 라벨을 비교해본다.

---

## 레코딩 룰

무겁고 자주 쓰는 쿼리를 **미리 계산해 새 메트릭으로 저장**한다.

```yaml
groups:
  - name: platform.recording
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:ratio5m
        expr: |
          sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (job) (rate(http_requests_total[5m]))
```

### 명명 규칙

```
level:metric:operation

job:http_requests:rate5m
└집계 수준┘└─메트릭─┘└연산┘
```

| 왜 쓰나 | 효과 |
|---|---|
| 대시보드 로딩이 느릴 때 | 미리 계산해두면 즉시 응답 |
| 같은 복잡한 식을 여러 곳에서 쓸 때 | 한 곳에서 정의·수정 |
| 알림 룰이 무거울 때 | 평가 부담 감소 |

> 레코딩 룰은 **결과를 새 시계열로 저장**하므로 그만큼 저장 공간을 쓴다. 정말 반복해서 쓰는 것만 만든다.  
> 알림 룰(`alert:`)과 레코딩 룰(`record:`)은 같은 파일에 함께 둘 수 있다. → `05-alerting/`

---

## 디버깅 요령

```promql
# 1) 메트릭이 존재하는가
http_requests_total

# 2) 라벨 값이 뭐가 있는가
count by (status) (http_requests_total)

# 3) 값이 나오는가 (rate 를 벗겨본다)
http_requests_total[5m]

# 4) 라벨이 매칭되는가 (양쪽을 따로)
sum by (pod) (metric_a)
sum by (pod) (metric_b)
```

```bash
# 존재하는 메트릭 이름 전체
curl -s localhost:9090/api/v1/label/__name__/values | jq

# 특정 라벨의 값들
curl -s 'localhost:9090/api/v1/label/job/values' | jq
```

| 증상 | 원인 |
|---|---|
| `No data` | 메트릭 이름 오타, 라벨 값 불일치, 타겟이 DOWN |
| `histogram_quantile` 결과가 NaN | `sum by (le)`를 빼먹음 |
| 나눗셈 결과가 빈 값 | 양쪽 라벨 집합 불일치 → `on`/`ignoring` |
| 그래프가 계단 모양 | 카운터에 `rate()`를 안 씀 |
| 값이 튄다 | `irate` 사용, 또는 `rate` 구간이 너무 짧음 |

---

## 배운 점

- 결과 타입은 **순간 벡터**(그릴 수 있다)와 **구간 벡터**(함수를 거쳐야 한다) 두 가지
- **카운터는 반드시 `rate()`** — 그대로 그리면 계단만 보인다
- `rate`(기본) / `irate`(순간, 알림에는 금지) / `increase`(총량)
- rate 구간은 **스크랩 간격의 4배 이상** (15s 간격이면 `[5m]` 권장)
- 세 함수 모두 **카운터 리셋을 자동 보정**한다
- `by`는 남길 라벨, `without`은 버릴 라벨
- **`sum(rate(...))`이 맞고 `rate(sum(...))`은 틀렸다** — rate를 항상 먼저
- CPU 사용률은 **idle의 증가율을 100에서 뺀다**
- `predict_linear`로 "곧 찬다"를 미리 알릴 수 있다
- **`histogram_quantile`에는 `sum by (le)`가 필수** — 빼먹으면 NaN
- 나눗셈 결과가 비면 **라벨 불일치** → `on`/`ignoring`/`group_left`
- 반복되는 무거운 쿼리는 **레코딩 룰**로 미리 계산 (`level:metric:operation`)
- 디버깅은 쿼리를 안쪽부터 한 겹씩 벗겨가며 확인한다
