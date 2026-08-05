# 06 로깅 (Loki)

메트릭이 "얼마나"를 알려준다면 로그는 **"정확히 무슨 일이 있었는지"** 를 알려준다.  
Loki는 "Prometheus를 로그에 적용한" 설계다 — **로그 본문은 인덱싱하지 않고 라벨만 인덱싱**해서, 훨씬 싸게 로그를 다룬다.

---

## PLG 스택

```
  각 노드                          중앙
┌──────────────┐
│ 파드 stdout   │
│    ↓         │
│ /var/log/    │──▶ promtail ──push──▶ Loki ──query──▶ Grafana
│ containers/  │   (DaemonSet)       (저장·조회)      (Explore)
└──────────────┘
```

| 구성요소 | 역할 |
|---|---|
| **promtail** | 각 노드에서 컨테이너 로그를 tail하고 K8s 라벨을 붙여 push (DaemonSet) |
| **Loki** | 로그 저장·조회. 라벨만 인덱싱 |
| **Grafana** | LogQL 질의 UI. 메트릭과 같은 화면 |

> 앱은 **stdout/stderr로만 출력하면 된다.** 파일에 쓰거나 로그 수집 라이브러리를 붙일 필요가 없다 — 컨테이너 로그 수집은 플랫폼의 책임이다.  
> promtail 대신 **Grafana Alloy**(구 Grafana Agent)를 쓰는 흐름도 있다. 메트릭·로그·트레이스를 한 에이전트로 처리한다.

---

## Loki의 핵심 아이디어

```
[ Elasticsearch ]   로그 전문(full-text)을 인덱싱
                    → 검색이 강력하지만 인덱스가 원본만큼 커진다

[ Loki ]            라벨만 인덱싱, 로그 본문은 압축해 통째로 저장
                    → 인덱스가 아주 작다. 대신 본문 검색은 grep 방식(brute force)
```

| | Loki | Elasticsearch (EFK) |
|---|---|---|
| 인덱싱 | **라벨만** | 전문 인덱싱 |
| 저장 비용 | 낮다 | 높다 (인덱스가 크다) |
| 검색 | 라벨로 좁힌 뒤 스캔 | 임의 전문 검색이 빠르다 |
| 운영 난이도 | 낮다 | 높다 (샤드·힙 튜닝) |
| 메트릭 연계 | Grafana에서 자연스럽다 | Kibana 별도 |

> **로그를 라벨로 먼저 좁힐 수 있다면 Loki가 훨씬 싸다.** Kubernetes는 네임스페이스·파드·컨테이너라는 좋은 라벨이 이미 있으므로 궁합이 좋다.  
> 반대로 "전체 로그에서 특정 문자열을 찾겠다"는 사용 패턴이면 Loki는 느리다.

---

## ⚠️ 라벨 설계 — 카디널리티

Loki에서 가장 중요한 주제다. Prometheus와 똑같은 함정이 여기도 있다.

```
스트림(stream) = 라벨 조합 하나 = 별도의 청크(chunk)

{namespace="app", pod="app-1", container="app"}   ← 스트림 1개
```

```
❌ {trace_id="abc-123"}     ← 요청마다 새 스트림 = 스트림 수백만 개  💥
❌ {user_id="..."}          ← 사용자 수만큼
❌ {timestamp="..."}        ← 무한

✅ {namespace="sample-app"}
✅ {pod="app-xxx", container="app"}
✅ {level="error"}          ← 값이 몇 개뿐이면 OK
```

> **높은 카디널리티 정보는 라벨이 아니라 로그 본문에 둔다.** 본문에 있으면 `| json | trace_id="abc"`로 필터할 수 있다 — 인덱스를 늘리지 않고도 찾을 수 있다.  
> 이게 Loki 설계의 핵심이다: **"인덱스는 좁히기용, 검색은 스캔으로."**

---

## LogQL

PromQL을 닮았다. **라벨 셀렉터(필수) → 라인 필터 → 파서 → 라벨 필터** 순서로 파이프라인을 쌓는다.

```logql
{namespace="sample-app"}                          # 1. 스트림 선택 (필수)
{namespace="sample-app"} |= "error"               # 2. 라인 필터
{namespace="sample-app"} |= "error" != "healthz"
{namespace="sample-app"} |~ "(?i)timeout|refused" # 정규식
```

| 연산자 | 의미 |
|---|---|
| `\|=` | 포함 |
| `!=` | 미포함 |
| `\|~` | 정규식 일치 |
| `!~` | 정규식 불일치 |

### 파서 — 구조화된 로그 다루기

```logql
# JSON 로그
{namespace="app"} | json | level="error" | duration > 1s

# logfmt (level=error msg="..." duration=1.2s)
{namespace="app"} | logfmt | status_code >= 500

# 패턴 추출
{job="nginx"} | pattern `<ip> - - <_> "<method> <path> <_>" <status> <_>` | status="500"

# 정규식 추출
{job="app"} | regexp `duration=(?P<duration>[0-9.]+)` | duration > 2
```

> **라벨 셀렉터로 먼저 좁힌 뒤 파싱한다.** 파서는 스캔한 모든 라인에 대해 동작하므로, 좁히지 않으면 느리다.  
> `{namespace=~".+"}` 같이 전체를 스캔하는 쿼리는 클러스터가 커지면 타임아웃 난다.

### 로그를 메트릭으로

```logql
# 초당 에러 로그 수
rate({namespace="sample-app"} |= "error" [5m])

# 파드별 로그 건수
sum by (pod) (count_over_time({namespace="sample-app"}[5m]))

# 로그에서 뽑은 값의 분위수
quantile_over_time(0.95,
  {namespace="app"} | json | unwrap duration [5m]
) by (pod)
```

> 이걸로 **로그 기반 알림**도 만들 수 있다. 다만 메트릭으로 낼 수 있는 건 메트릭으로 내는 게 항상 싸다. 로그 알림은 앱을 계측할 수 없을 때의 차선책이다.

---

## 실전 — SingleBinary 구성

Loki 차트의 기본 배포 모드는 `SimpleScalable`인데, **read/write/backend로 쪼개고 오브젝트 스토리지를 요구한다.** 소규모 클러스터에는 과하다.

```yaml
deploymentMode: SingleBinary        # 모든 컴포넌트를 한 프로세스로

loki:
  auth_enabled: false               # 멀티테넌시 없음 (X-Scope-OrgID 불필요)

  commonConfig:
    replication_factor: 1           # 복제할 대상이 없다

  storage:
    type: filesystem

  # 차트 v6는 스키마를 명시하지 않으면 뜨지 않는다
  schemaConfig:
    configs:
      - from: "2024-04-01"
        store: tsdb                 # tsdb + v13 이 현재 권장 조합
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h

singleBinary:
  replicas: 1
  persistence:
    enabled: false

# SingleBinary 모드에서는 쓰지 않는 스케일아웃 컴포넌트
write:  { replicas: 0 }
read:   { replicas: 0 }
backend: { replicas: 0 }

gateway:      { enabled: false }
chunksCache:  { enabled: false }
resultsCache: { enabled: false }
minio:        { enabled: false }
```

| 배포 모드 | 언제 |
|---|---|
| **SingleBinary** | 소규모 (~수십 GB/일). 학습·dev |
| SimpleScalable | 중규모. 오브젝트 스토리지 필요 |
| Distributed (microservices) | 대규모 (~TB/일) |

> 규모에 안 맞는 아키텍처를 고르는 것 자체가 흔한 실수다. **1TB/일을 위한 구성을 t3.medium 2대에 올리면 뜨지도 않는다.**

### 🔧 실제로 겪은 것 — read-only 파일시스템

```
persistence: enabled: false 로 두면
  → /var/loki 에 아무것도 마운트되지 않는다
  → 컨테이너 루트 파일시스템은 read-only
  → Loki 기동 실패: "mkdir /var/loki: read-only file system"
```

```yaml
singleBinary:
  persistence:
    enabled: false
  # 대신 쓰기 가능한 임시 볼륨을 그 경로에 준다
  extraVolumes:
    - name: loki-data
      emptyDir: {}
  extraVolumeMounts:
    - name: loki-data
      mountPath: /var/loki
```

> **PVC를 끄는 것과 "쓸 공간이 없는 것"은 다르다.** 영속성이 필요 없어도 프로세스는 여전히 디스크에 써야 한다.  
> EBS CSI 드라이버가 없는 클러스터에서 PVC를 켜면 파드가 `Pending`에서 영원히 멈춘다. 그래서 `emptyDir`가 답이 된다. → `../k8s-manifests/06-storage/`

### promtail

```yaml
config:
  clients:
    # 차트 기본값은 loki-gateway 를 가리킨다.
    # gateway 를 껐으므로 Loki 서비스에 직접 push 한다.
    - url: http://loki.observability.svc.cluster.local:3100/loki/api/v1/push

resources:
  requests: { cpu: 20m, memory: 64Mi }
  limits:   { memory: 128Mi }
```

> promtail 차트의 기본 스크랩 설정이 이미 파드를 자동 발견하고 namespace·pod·container 라벨을 붙여준다. **실제로 바꿔야 하는 건 push 주소 하나뿐이다.**

---

## 로그를 잘 남기는 법 (앱 쪽)

```json
{"level":"error","ts":"2026-08-05T10:23:45Z","msg":"db query failed",
 "trace_id":"abc123","user_id":"u-99","duration_ms":1523}
```

| 원칙 | 이유 |
|---|---|
| **stdout/stderr로만** | 파일 로테이션·수집은 플랫폼이 처리 |
| **구조화(JSON/logfmt)** | `\| json`으로 필드 필터가 가능해진다 |
| **`trace_id` 포함** | 트레이스 ↔ 로그 연결 → `07-tracing/` |
| 심각도를 정확히 | error가 남발되면 error 필터가 무의미 |
| 비밀 정보 금지 | 토큰·비밀번호·주민번호는 로그에도 남기지 않는다 |
| 고카디널리티는 본문에 | 라벨이 아니라 필드로 |

---

## 보존과 비용

```yaml
loki:
  limits_config:
    retention_period: 168h          # 7일
    ingestion_rate_mb: 10           # 테넌트당 초당 수집 제한
    ingestion_burst_size_mb: 20
    max_query_series: 500
  compactor:
    retention_enabled: true
```

> 로그는 **메트릭보다 훨씬 빨리 쌓인다.** 보존 기간과 수집 제한을 처음부터 정해두지 않으면 디스크가 먼저 죽는다.  
> 운영에서는 로컬 파일시스템 대신 **S3 + IRSA**로 넘긴다. 이건 플래그 하나가 아니라 별도의 작업이다. → `../terraform-lab/09-aws-vpc-eks/`

---

## 배운 점

- Loki는 **라벨만 인덱싱하고 본문은 스캔**한다 — 그래서 EFK보다 훨씬 싸다
- 전문 검색이 주 사용 패턴이면 Loki는 느리다. **라벨로 좁힐 수 있을 때** 유리하다
- 앱은 **stdout/stderr로만** 출력하면 된다 — 수집은 promtail(DaemonSet)의 몫
- **라벨 카디널리티가 Loki를 죽인다** — trace_id·user_id를 라벨로 쓰지 않는다
- 고카디널리티 정보는 **로그 본문에 두고 `| json`으로 필터**한다
- LogQL 순서: **라벨 셀렉터(필수) → 라인 필터 → 파서 → 라벨 필터**
- 셀렉터로 먼저 좁히지 않으면 파서가 전체를 스캔해 느려진다
- `rate({...} |= "error" [5m])`로 로그를 메트릭처럼 다룰 수 있다 (단, 계측이 항상 더 싸다)
- 배포 모드를 규모에 맞게: **SingleBinary(소규모) / SimpleScalable / Distributed**
- 차트 v6는 `schemaConfig`를 명시하지 않으면 기동하지 않는다 (tsdb + v13)
- **PVC를 끄는 것과 쓸 공간이 없는 것은 다르다** — read-only FS 때문에 `emptyDir` 마운트가 필요했다
- gateway를 끄면 promtail의 push 주소를 Loki 서비스로 직접 바꿔야 한다
- 로그에는 `trace_id`를 포함시켜 트레이스와 연결한다
- **보존 기간·수집 제한을 처음부터** 정하지 않으면 디스크가 먼저 죽는다
