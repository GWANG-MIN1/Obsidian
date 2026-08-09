# 09 관측성 비용

관측성 스택은 **관측 대상보다 비싸질 수 있다.** t3.medium 2대 클러스터에 kube-prometheus-stack + Loki + promtail을 얹으면 전체 자원의 절반 가까이를 차지한다.  
"다 수집하고 보자"가 기본값인데, **수집·저장·조회 전부가 돈**이다.

---

## 비용이 발생하는 지점

```
메트릭   시계열 개수 × 스크랩 빈도 × 보존 기간
로그     수집량(GB) × 보존 기간 (+ 조회 비용)
트레이스  스팬 수 × 샘플링률 × 보존 기간
         │
      전부 '카디널리티 × 시간' 의 곱
```

| 항목 | 주 비용 요인 |
|---|---|
| Prometheus | **시계열 수(카디널리티)** → 메모리·디스크 |
| Loki | 수집량 → 오브젝트 스토리지 + 인덱스 |
| CloudWatch | **커스텀 메트릭 개수 + 로그 수집 GB** |
| 관리형 서비스(Datadog 등) | 호스트 수 + 커스텀 메트릭 + 로그 |

> **"곱"이라는 게 핵심이다.** 카디널리티가 2배가 되고 보존이 2배가 되면 비용은 4배가 된다.

---

## 카디널리티 — 메트릭 비용의 전부

```
http_requests_total{method, status, handler}
  method 4 × status 6 × handler 20 = 480 시계열      ✅

http_requests_total{method, status, handler, user_id}
  480 × 사용자 10만 = 4800만 시계열                   💥
```

> **카디널리티 폭발은 비용 문제이기 전에 장애 문제다.** Prometheus가 OOM으로 죽는다. → `../observability-lab/02-prometheus-architecture/`

```bash
# 시계열을 많이 쓰는 메트릭 상위
curl -s localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName'

# 라벨별 카디널리티
curl -s localhost:9090/api/v1/status/tsdb | jq '.data.labelValueCountByLabelName'
```

### 필요 없는 메트릭 버리기

```yaml
# 스크랩 후 저장 전에 걸러낸다
metric_relabel_configs:
  # Go 런타임 내부 메트릭 — 앱 디버깅에 거의 안 쓴다
  - source_labels: [__name__]
    regex: "go_gc_.*|go_memstats_.*"
    action: drop

  # 고카디널리티 히스토그램 버킷
  - source_labels: [__name__]
    regex: "apiserver_request_duration_seconds_bucket"
    action: drop

  # 특정 라벨만 제거 (시계열은 유지)
  - regex: "pod_template_hash|controller_revision_hash"
    action: labeldrop
```

> **`labeldrop`은 시계열을 합쳐준다.** `pod_template_hash` 같은 배포마다 바뀌는 라벨을 지우면, 배포할 때마다 새 시계열이 생기는 걸 막는다.  
> 기본 설정에서 kube-prometheus-stack은 **수만 개의 시계열**을 만든다. 실제로 대시보드·알림에 쓰는 건 그중 일부다.

```yaml
# 아예 안 쓰는 exporter·타겟은 끈다
kubeControllerManager: { enabled: false }   # EKS: 스크랩 불가
kubeScheduler:         { enabled: false }
kubeEtcd:              { enabled: false }
kubeProxy:             { enabled: false }
defaultRules:
  rules:
    etcd: false
    windows: false
```

> **EKS에서 스크랩 불가한 타겟을 끄는 건 알림 위생이자 비용 절감**이기도 하다. 실패하는 스크랩도 자원을 쓴다. → `../observability-lab/08-kube-prometheus-stack/`

---

## 보존 기간

```yaml
prometheus:
  prometheusSpec:
    retention: 7d           # 기간
    retentionSize: 40GB     # 용량 (먼저 도달하는 쪽)
```

```
질문: 30일 전 CPU 그래프를 실제로 몇 번 봤는가?
```

| 용도 | 필요 해상도 | 필요 기간 |
|---|---|---|
| 장애 대응 | 초·분 단위 | **최근 며칠** |
| 주간 리뷰 | 5분 | 2~4주 |
| 용량 계획·추세 | 1시간 | 6개월~1년 |

```
고해상도 × 장기 보존 = 가장 비싼 조합이고, 거의 필요 없다
      ↓
해법: 짧은 고해상도(로컬) + 긴 저해상도(다운샘플링)
      → Thanos·Mimir 의 compaction/downsampling
```

```yaml
# 레코딩 룰로 미리 집계해두면 장기 보존 대상이 줄어든다
groups:
  - name: aggregate
    interval: 5m
    rules:
      - record: namespace:cpu_usage:rate5m
        expr: sum by (namespace) (rate(container_cpu_usage_seconds_total[5m]))
```

> **원본 시계열은 짧게, 집계된 시계열은 길게 보존**하는 게 정석이다. → `../observability-lab/03-promql/`  
> 이 프로젝트가 `retention: 7d` + PVC 없음을 고른 것은 **매일 파괴하는 dev 클러스터**라는 전제 때문이다. 장기 추세를 볼 이유가 없으니 저장 비용이 0에 가까운 구성이 정당하다. → `04-compute-purchasing/`

---

## 로그 — 가장 빨리 쌓인다

```
파드 100개 × 로그 1MB/분 = 시간당 6GB = 월 4.3TB
```

| 절감 수단 | 효과 |
|---|---|
| **로그 레벨 조정** | DEBUG를 끄는 것만으로 절반 이상 줄기도 한다 |
| **헬스체크 로그 제외** | `/healthz` 요청 로그가 전체의 30%인 경우가 흔하다 |
| 보존 기간 | 30일 → 7일 |
| 샘플링 | 반복되는 정상 로그를 일부만 |
| 구조화 | 파싱 비용↓, 필터 효율↑ |

```yaml
# promtail — 수집 단계에서 버린다 (가장 효과적)
scrape_configs:
  - job_name: kubernetes-pods
    pipeline_stages:
      - drop:
          expression: '.*"path":"/healthz".*'
      - drop:
          expression: ".*GET /metrics.*"
      # 레벨 기반
      - match:
          selector: '{namespace="myapp"} |= "DEBUG"'
          action: drop
```

```yaml
loki:
  limits_config:
    retention_period: 168h            # 7일
    ingestion_rate_mb: 10             # 폭주 방지
    ingestion_burst_size_mb: 20
  compactor:
    retention_enabled: true
```

> ⚠️ **`ingestion_rate_mb` 같은 상한이 없으면 앱 하나가 로그를 폭주시켜 스토리지를 먹어치운다.** 무한 재시도 루프에 에러 로그를 찍는 앱이 대표적이다.  
> **수집 단계에서 버리는 게 저장 후 지우는 것보다 항상 싸다.** → `../observability-lab/06-logging/`

### CloudWatch Logs

```
수집(GB) + 저장(GB·월) + 조회(스캔 GB)
      ↓
Insights 쿼리로 대량 스캔하면 조회 비용도 만만치 않다
```

```bash
# 로그 그룹 보존 기간 확인 — 기본이 '무기한' 이다
aws logs describe-log-groups \
  --query 'logGroups[?retentionInDays==`null`].[logGroupName,storedBytes]' --output table

aws logs put-retention-policy --log-group-name /aws/eks/my-cluster/cluster --retention-in-days 30
```

> ⚠️ **CloudWatch 로그 그룹의 기본 보존은 "무기한"이다.** 만들 때 보존 기간을 안 정하면 영원히 쌓인다. EKS 컨트롤 플레인 로그가 대표적이다. → `../security-lab/09-cluster-hardening/`

---

## 트레이스 — 샘플링이 전부

```
초당 1만 요청 전부 저장 = 트레이스 비용이 로그를 넘는다
```

```yaml
# Tail 샘플링 — 에러·느린 건 전부, 정상은 소량
processors:
  tail_sampling:
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow
        type: latency
        latency: { threshold_ms: 1000 }
      - name: baseline
        type: probabilistic
        probabilistic: { sampling_percentage: 1 }
```

> **Head 샘플링 1%는 싸지만 정작 보고 싶은 느린 요청을 놓친다.** Tail 샘플링이 비용 대비 가치가 훨씬 높다. → `../observability-lab/07-tracing/`

---

## 관측성 스택 자체의 자원

```yaml
prometheus:
  prometheusSpec:
    resources:
      requests: { cpu: 100m, memory: 400Mi }
      limits:   { memory: 1Gi }        # 메모리만. CPU limit 없음
```

> **스크래퍼에 CPU limit을 걸면 스로틀링으로 스크랩을 놓친다.** 놓친 스크랩은 데이터 구멍이 되고, 구멍이 오탐 알림을 만든다. → `07-rightsizing/`  
> 관측성 스택도 라이트사이징 대상이다. 다만 **여기서 아끼려다 관측성 자체를 잃으면 다른 모든 최적화의 근거가 사라진다.**

### PVC를 안 붙이는 선택

```
장점: EBS 비용 0, CSI 드라이버 불필요
단점: 파드 재시작 시 데이터 소실
      → 매일 파괴하는 dev 클러스터에서만 정당하다
```

> **이건 비용 결정이자 제약 조건이다.** 운영에서는 반드시 영속화해야 하고, 그때 EBS·S3 비용이 새로 생긴다.

---

## 관측성으로 비용을 관측하기

역설적이지만 **관측성 비용을 줄이려면 관측성이 필요하다.**

```promql
# 시계열 총량 추이
prometheus_tsdb_head_series

# 초당 수집 샘플 수
rate(prometheus_tsdb_head_samples_appended_total[5m])

# Loki 수집량
sum(rate(loki_distributor_bytes_received_total[5m]))

# 스크랩 대상 수
count(up)
```

> **`prometheus_tsdb_head_series`를 대시보드에 띄워두면 카디널리티 폭발을 즉시 안다.** 새 서비스를 배포한 뒤 이 값이 튀면 그 서비스의 라벨을 본다.  
> 이 값에 알림을 걸어두는 것도 실용적이다 — 장애로 번지기 전에 잡는다. → `../observability-lab/05-alerting/`

---

## 균형 잡기

```
❌ 다 수집한다            → 비싸고, 카디널리티 폭발로 죽는다
❌ 최소만 수집한다        → 장애 때 아무것도 못 본다
✅ 질문에 답하는 것만 수집  → "이 메트릭으로 어떤 판단을 하는가?"
```

| 판단 기준 | |
|---|---|
| 이 메트릭으로 **알림을 걸거나 대시보드에 쓰는가** | 아니면 버린다 |
| 이 로그를 지난 3개월간 **조회한 적 있는가** | 없으면 보존 단축 |
| 이 라벨로 **필터·집계를 하는가** | 아니면 `labeldrop` |

> **대시보드에 안 쓰고 알림에도 안 쓰는 메트릭은 "혹시 몰라서" 저장하는 것이다.** 그 "혹시"의 비용을 계산해본다.  
> 다만 ⚠️ **장애 때 필요한 메트릭을 미리 알 수는 없다.** 그래서 무작정 줄이는 게 아니라 **카디널리티가 큰 것부터** 손댄다 — 상위 10개 메트릭이 전체의 대부분을 차지하는 게 보통이다.

---

## 배운 점

- **관측성 스택이 관측 대상보다 비쌀 수 있다**
- 비용은 전부 **"카디널리티 × 시간"의 곱** — 둘 다 2배면 비용은 4배
- **카디널리티 폭발은 비용 문제이기 전에 장애 문제** (Prometheus OOM)
- `api/v1/status/tsdb`로 **시계열을 많이 쓰는 메트릭 상위**를 확인한다
- `metric_relabel_configs`의 `drop`·**`labeldrop`** 으로 저장 전에 걸러낸다
- `pod_template_hash` 같은 라벨은 **배포마다 새 시계열**을 만든다
- EKS에서 스크랩 불가한 타겟을 끄는 건 **알림 위생이자 비용 절감**
- **고해상도 × 장기 보존이 가장 비싸고 거의 필요 없다**
- **원본은 짧게, 레코딩 룰로 집계한 것은 길게** 보존한다
- 로그는 **수집 단계에서 버리는 게 저장 후 지우는 것보다 항상 싸다**
- 헬스체크·`/metrics` 로그가 전체의 상당 비율인 경우가 흔하다
- ⚠️ **`ingestion_rate_mb` 상한이 없으면 앱 하나가 스토리지를 먹어치운다**
- ⚠️ **CloudWatch 로그 그룹의 기본 보존은 "무기한"** — 반드시 설정한다
- 트레이스는 **tail 샘플링**이 비용 대비 가치가 높다 (head 1%는 느린 걸 놓친다)
- **스크래퍼에 CPU limit을 걸지 않는다** — 스크랩 누락 → 오탐
- PVC 미사용은 **매일 파괴하는 클러스터에서만 정당**한 비용 결정
- `prometheus_tsdb_head_series`를 대시보드에 띄우면 **카디널리티 폭발을 즉시** 안다
- 판단 기준은 **"이 메트릭으로 어떤 판단을 하는가"**
- ⚠️ 장애 때 필요한 메트릭을 미리 알 수는 없다 — **카디널리티가 큰 것부터** 손댄다
- 여기서 아끼려다 관측성을 잃으면 **다른 모든 최적화의 근거가 사라진다**
