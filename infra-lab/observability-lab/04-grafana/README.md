# 04 Grafana

Grafana는 데이터를 저장하지 않는다. **Prometheus·Loki 같은 데이터소스에 질의해 시각화하는 계층**일 뿐이다.  
그래서 Grafana를 잘 쓴다는 건 대시보드를 예쁘게 만드는 게 아니라, **장애 때 3분 안에 원인 범위를 좁힐 수 있게 배치하는 것**이다.

---

## 위치

```
Prometheus ─┐
Loki ───────┼──▶ Grafana ──▶ 대시보드 / Explore / 알림
Tempo ──────┘      │
                   └─ 저장 안 함. 매번 데이터소스에 질의한다
```

---

## 데이터소스

```yaml
# 프로비저닝 파일로 선언 (UI 클릭 대신)
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://kube-prometheus-stack-prometheus.observability.svc:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki.observability.svc.cluster.local:3100
```

| access 모드 | 동작 |
|---|---|
| **proxy** (권장) | Grafana 서버가 대신 질의 — 브라우저가 데이터소스에 직접 접근할 필요 없음 |
| direct | 브라우저에서 직접 — 클러스터 내부 주소는 쓸 수 없다 |

> **Prometheus와 Loki를 같은 Grafana에 붙이는 것이 핵심이다.** 메트릭 그래프에서 스파이크를 보고 그 시간대 로그로 바로 넘어갈 수 있어야 한다.  
> kube-prometheus-stack에서는 `grafana.additionalDataSources`에 Loki를 추가하면 된다. → `08-kube-prometheus-stack/`

---

## 패널 고르기

| 패널 | 언제 |
|---|---|
| **Time series** | 대부분의 경우. 추세를 본다 |
| **Stat** | 단일 값 강조 (현재 에러율, 가동 중인 파드 수) |
| **Gauge** | 임계값 대비 현재 위치 (디스크 사용률) |
| **Bar gauge** | 여러 항목 비교 (파드별 메모리) |
| **Table** | 라벨이 많은 목록 (타겟 상태) |
| **Heatmap** | 분포의 시간 변화 (지연 히스토그램) |
| **Logs** | Loki 로그 스트림 |
| **State timeline** | 상태 변화 (up/down 이력) |

```
❌ 파이 차트로 시계열 표현        ← 시간 정보가 사라진다
✅ Time series + 범례에 현재값
```

### 패널 설정에서 실제로 중요한 것

```
Unit          bytes, seconds, percent(0-100) ← 안 맞추면 "1500000000" 이 그대로 보인다
Legend        {{pod}} 처럼 라벨 템플릿으로 축약
Thresholds    경고선 (80% 노랑, 90% 빨강)
Min/Max       퍼센트 패널은 0~100 고정 (자동 스케일이면 3%도 꽉 차 보인다)
Null value    "connected" vs "null as zero" — 스크랩 실패를 0으로 그리면 거짓말이 된다
```

> **단위와 축 범위를 안 맞추면 그래프가 거짓말을 한다.** 자동 스케일된 CPU 그래프는 2%도 위험해 보인다.

---

## 변수 (템플릿)

대시보드를 재사용 가능하게 만드는 기능. 네임스페이스·파드마다 대시보드를 복제할 필요가 없다.

```
Variable: namespace
  Type   : Query
  Query  : label_values(kube_pod_info, namespace)

Variable: pod
  Type   : Query
  Query  : label_values(kube_pod_info{namespace="$namespace"}, pod)
  Multi-value: on
  Include All: on
```

```promql
# 패널 쿼리에서 사용
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="$namespace", pod=~"$pod"}[5m]))
```

| 변수 타입 | 용도 |
|---|---|
| **Query** | 데이터소스에서 라벨 값을 조회 (`label_values`) |
| Custom | 직접 나열한 목록 |
| Interval | `$__interval` 같은 시간 구간 선택 |
| Datasource | 데이터소스 자체를 전환 (dev/prod Prometheus) |
| Textbox | 자유 입력 |

### 내장 변수

```promql
$__interval      # 패널 폭에 맞춘 적절한 구간 → rate(metric[$__interval])
$__rate_interval # rate 전용. 스크랩 간격을 고려해 계산 (rate 에는 이걸 권장)
$__range         # 현재 선택한 시간 범위 전체
```

> **`rate()` 안에는 `$__rate_interval`을 쓴다.** `$__interval`은 스크랩 간격보다 짧아질 수 있어 그래프가 끊긴다.  
> 다중 선택 변수는 정규식으로 들어가므로 셀렉터에서 `=`가 아니라 **`=~`** 를 써야 한다.

---

## Explore — 대시보드보다 자주 쓴다

장애 때 실제로 쓰는 화면이다. 대시보드는 "정해진 질문", Explore는 "즉석 질문"이다.

```
Explore ─▶ Prometheus 로 에러율 확인
        ─▶ 데이터소스를 Loki 로 전환 (시간 범위 유지됨)
        ─▶ {namespace="sample-app"} |= "error"
        ─▶ Split view 로 메트릭·로그 나란히
```

> 시간 범위가 유지된 채 데이터소스를 바꿀 수 있다는 게 핵심이다. **"이 스파이크 시점의 로그"** 로 바로 갈 수 있다.

---

## 프로비저닝 — Dashboard as Code

UI에서 만든 대시보드는 **파드가 죽으면 사라진다**(PVC 없이 운영한다면). GitOps 환경에서는 코드로 관리한다.

```yaml
# ConfigMap 으로 대시보드 배포 (sidecar 가 자동 로드)
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-dashboard
  namespace: observability
  labels:
    grafana_dashboard: "1"        # 이 라벨을 sidecar 가 감시한다
data:
  my-dashboard.json: |
    { "title": "Platform Overview", "panels": [ ... ] }
```

```yaml
# values.yaml — 공개 대시보드를 ID로 가져오기
grafana:
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: default
          folder: ""
          type: file
          options:
            path: /var/lib/grafana/dashboards/default
  dashboards:
    default:
      node-exporter:
        gnetId: 1860           # grafana.com 대시보드 ID
        revision: 37
        datasource: Prometheus
```

| 자주 쓰는 공개 대시보드 | ID |
|---|---|
| Node Exporter Full | 1860 |
| Kubernetes Cluster Monitoring | 315 |
| Loki Logs | 13639 |
| ArgoCD | 14584 |

> **직접 만들기 전에 grafana.com에서 찾아본다.** kube-prometheus-stack은 kubernetes-mixin 대시보드를 기본 포함하므로, 클러스터 기본 대시보드는 이미 갖춰진 상태로 시작한다.  
> 대시보드 JSON을 손으로 편집하지 말고, **UI에서 만든 뒤 Export → JSON을 Git에 커밋**하는 흐름이 현실적이다.

---

## 대시보드 설계 원칙

```
┌─────────────────────────────────────────┐
│  1행: 요약 (Stat)                        │  ← 3초 안에 "정상인가?"
│  에러율 0.02% | p95 120ms | 파드 12/12   │
├─────────────────────────────────────────┤
│  2행: RED (Time series)                  │  ← 30초 안에 "무엇이?"
│  요청률 | 에러율 | 지연 분포               │
├─────────────────────────────────────────┤
│  3행: 자원 (USE)                          │  ← 3분 안에 "왜?"
│  CPU | 메모리 | 네트워크 | 디스크           │
├─────────────────────────────────────────┤
│  4행: 로그 (Logs 패널)                     │
└─────────────────────────────────────────┘
```

| 원칙 | 이유 |
|---|---|
| **위에서 아래로 좁혀지게** | 요약 → 서비스 → 자원 → 로그 |
| **한 화면에 들어오게** | 스크롤해야 보이는 패널은 안 본다 |
| 패널당 시계열은 10개 이하 | 20줄짜리 그래프는 아무것도 안 보인다 → `topk()` |
| 대시보드는 목적별로 | "전부 다 보는 대시보드"는 아무 질문에도 답 못 한다 |
| 임계선 표시 | 숫자만 보면 그게 높은 건지 모른다 |

> **"이 대시보드로 어떤 질문에 답하는가"** 를 먼저 정한다. 답이 없으면 그 패널은 지운다.

---

## 실전 — 이 스택의 구성

```bash
# 포트포워딩으로 접속 (LoadBalancer 를 만들지 않는 dev 구성)
kubectl -n observability port-forward svc/kube-prometheus-stack-grafana 3000:80
# http://localhost:3000  (기본: admin / prom-operator)
```

```yaml
# values.yaml
grafana:
  enabled: true
  defaultDashboardsEnabled: true     # kubernetes-mixin 대시보드 포함

  # adminPassword 는 여기 쓰지 않는다 — Git에 비밀번호를 넣지 않기 위해.
  # 실제 운영에서는 External Secrets 로 주입한다.

  additionalDataSources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki.observability.svc.cluster.local:3100
      isDefault: false
```

> **비밀번호를 values.yaml에 쓰면 Git에 평문으로 올라간다.** 포트포워딩 전용 dev 스택이면 차트 기본값을 쓰고, 운영에서는 External Secrets로 주입한다.  
> PVC 없이 운영하면 파드 재시작 시 UI에서 만든 대시보드가 사라진다. 그래서 **프로비저닝이 선택이 아니라 필수**가 된다.

---

## 배운 점

- Grafana는 **저장하지 않는다** — 데이터소스에 매번 질의하는 시각화 계층
- 데이터소스 access는 **proxy**를 쓴다 (클러스터 내부 주소를 쓸 수 있다)
- **Prometheus와 Loki를 같은 Grafana에** 붙여야 메트릭 → 로그 이동이 가능하다
- 시계열에 파이 차트를 쓰지 않는다 — 시간 정보가 사라진다
- **단위와 축 범위를 안 맞추면 그래프가 거짓말을 한다** (퍼센트는 0~100 고정)
- 스크랩 실패를 `null as zero`로 그리면 "0이었다"는 거짓 정보가 된다
- 변수(`label_values`)로 대시보드를 재사용 — 네임스페이스마다 복제하지 않는다
- **`rate()` 안에는 `$__interval`이 아니라 `$__rate_interval`**
- 다중 선택 변수는 정규식이므로 셀렉터에 `=~`를 쓴다
- 장애 때 실제로 쓰는 건 대시보드보다 **Explore** (시간 범위 유지한 채 데이터소스 전환)
- UI에서 만든 대시보드는 파드가 죽으면 사라진다 → **ConfigMap 프로비저닝**
- 직접 만들기 전에 grafana.com의 공개 대시보드(1860, 315…)를 먼저 찾는다
- 대시보드는 **요약 → RED → USE → 로그** 순으로 좁혀지게 배치한다
- "이 대시보드로 어떤 질문에 답하는가"에 답이 없으면 그 패널은 지운다
