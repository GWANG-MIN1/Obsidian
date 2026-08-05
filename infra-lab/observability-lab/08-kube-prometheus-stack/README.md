# 08 kube-prometheus-stack 실전

Prometheus·Alertmanager·Grafana·exporter를 하나씩 배포하는 대신, **Operator 기반 차트 하나로 전부 세운다.**  
이 장은 앞의 7장을 실제 EKS 클러스터에 올리면서 마주치는 결정들 — 무엇을 끄고, 무엇을 영속화하고, 무엇을 알릴지 — 를 다룬다.

---

## 차트에 들어 있는 것

```
kube-prometheus-stack
├── prometheus-operator          # CRD를 감시해 설정을 생성
├── Prometheus (CR)              # 메트릭 수집·저장
├── Alertmanager (CR)            # 알림 발송
├── Grafana                      # 대시보드 (kubernetes-mixin 포함)
├── node-exporter (DaemonSet)    # 노드 자원
├── kube-state-metrics           # K8s 오브젝트 상태
└── 기본 룰·대시보드 (kubernetes-mixin)
```

> 이걸 손으로 조립하면 며칠 걸린다. 차트 하나로 **"돌아가는 기본선"** 을 확보하고, 거기서 필요한 것만 조정하는 게 현실적인 순서다.

---

## Operator 패턴 — 왜 CRD인가

```
[ 직접 운영 ]
  prometheus.yml 을 손으로 수정 → ConfigMap 갱신 → 리로드
  새 서비스가 생길 때마다 scrape_configs 를 편집해야 한다

[ Operator ]
  ServiceMonitor CR 을 만든다
      ↓ Operator가 감시
  prometheus.yml 을 자동 생성·리로드
```

| CRD | 역할 |
|---|---|
| **ServiceMonitor** | Service를 대상으로 스크랩 (가장 많이 쓴다) |
| **PodMonitor** | Service 없이 Pod를 직접 스크랩 |
| **PrometheusRule** | 알림·레코딩 룰 |
| **Probe** | blackbox 프로브 대상 |
| **Prometheus / Alertmanager** | 인스턴스 자체의 스펙 |

> **핵심 이점:** 앱 팀이 자기 Helm 차트에 ServiceMonitor를 하나 넣으면, 플랫폼 팀이 Prometheus 설정을 손대지 않아도 수집이 시작된다. 관측성이 **셀프서비스**가 된다.

### ServiceMonitor 예시

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: my-app
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: my-app   # 이 라벨을 가진 Service 를 찾는다
  namespaceSelector:
    matchNames: ["my-app"]
  endpoints:
    - port: metrics                    # Service 의 '포트 이름' (번호 아님)
      path: /metrics
      interval: 30s
```

> **가장 흔한 실패 두 가지:**  
> ① `port`에 포트 번호를 썼다 → Service의 **포트 이름**이어야 한다  
> ② `selector`가 Service 라벨과 안 맞는다 → Deployment 라벨이 아니라 **Service 라벨**이다

```bash
# 진단 순서
kubectl get svc my-app -o jsonpath='{.spec.ports[*].name}'    # 포트 이름 확인
kubectl get svc my-app --show-labels                          # 라벨 확인
curl -s localhost:9090/api/v1/targets | jq '.data.activeTargets[].labels.job'
```

### ServiceMonitor를 못 찾는 문제

차트 기본값은 **자기 Helm 릴리스 라벨이 붙은 것만** 수집한다. 다른 네임스페이스의 ServiceMonitor가 무시되는 원인이다.

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false   # 릴리스 라벨 제약 해제
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
```

> 이 세 줄이 없으면 **"ServiceMonitor를 만들었는데 타겟에 안 잡힌다"** 를 겪는다. 원인 찾기 어려운 축에 속한다.

---

## EKS 현실 — 컨트롤 플레인은 스크랩할 수 없다

```yaml
kubeControllerManager: { enabled: false }
kubeScheduler:         { enabled: false }
kubeEtcd:              { enabled: false }
kubeProxy:             { enabled: false }   # EKS는 메트릭을 127.0.0.1에 바인딩

defaultRules:
  create: true
  rules:
    etcd: false
    kubeControllerManager: false
    kubeSchedulerAlerting: false
    kubeSchedulerRecording: false
    kubeProxy: false
    windows: false          # Windows 노드 없음
```

```
끄지 않으면:
  타겟 4개가 영원히 빨강 → 관련 알림이 계속 발화
  → "우리 모니터링은 원래 빨개요"
  → 진짜 장애 알림도 무시된다   💥
```

> **모니터링이 죽는 가장 흔한 경로다.** 스크랩 대상을 끄는 것으로 끝나지 않고, **그 대상에 걸린 기본 룰도 함께 꺼야** 한다. 룰만 남으면 "데이터 없음"으로 계속 발화한다. → `05-alerting/`

---

## 영속성 — 켤 것인가 끌 것인가

```yaml
prometheus:
  prometheusSpec:
    retention: 7d
    # storageSpec 없음 — 의도적
```

| | PVC 켬 | PVC 끔 |
|---|---|---|
| 파드 재시작 시 | 데이터 유지 | **전부 소실** |
| 요구사항 | CSI 드라이버 필요 | 없음 |
| dev 클러스터 | 과할 수 있다 | 적합 |
| 운영 | 필수 | 불가 |

> **EBS CSI 드라이버가 없는 클러스터에서 PVC를 켜면 파드가 `Pending`에서 영원히 멈춘다.** 매일 파괴하는 dev 클러스터라면 끄는 게 맞다.  
> 다만 이건 "플래그 하나"가 아니라 **결정**이다. 영속화하려면 EBS CSI 애드온을 설치하거나(메트릭), S3 + IRSA를 붙여야 한다(Loki). → `../terraform-lab/09-aws-vpc-eks/`

### 리소스 설정

```yaml
prometheus:
  prometheusSpec:
    resources:
      requests: { cpu: 100m, memory: 400Mi }
      limits:   { memory: 1Gi }      # 메모리만. CPU limit은 두지 않는다
```

> **스크래퍼에 CPU limit을 걸면 부하 시 스로틀링으로 스크랩을 놓친다.** 놓친 스크랩은 데이터 구멍이 되고, 그 구멍이 오탐 알림을 만든다.  
> 메모리는 OOM으로 노드 전체를 위협하므로 limit을 둔다. **CPU는 request만, 메모리는 request + limit** 이 일반적인 패턴이다.

---

## 관측 대상 확장 — ArgoCD 예시

플랫폼 자체도 관측 대상이다. GitOps 컨트롤러가 죽으면 배포가 조용히 멈춘다.

```yaml
prometheus:
  prometheusSpec:
    additionalServiceMonitors:
      - name: argocd-application-controller
        namespaceSelector:
          matchNames: ["argocd"]
        selector:
          matchLabels:
            app.kubernetes.io/name: argocd-metrics
        endpoints:
          - port: metrics
      - name: argocd-server
        namespaceSelector:
          matchNames: ["argocd"]
        selector:
          matchLabels:
            app.kubernetes.io/name: argocd-server-metrics
        endpoints:
          - port: metrics
```

```yaml
# 그 위에 얹는 알림 — 상위 차트가 알 수 없는 '이 플랫폼의 약속'
additionalPrometheusRulesMap:
  platform-rules:
    groups:
      - name: platform.gitops
        rules:
          - alert: ArgoCDAppNotSynced
            expr: argocd_app_info{sync_status!="Synced"} == 1
            for: 15m
            labels: { severity: warning }
          - alert: ArgoCDAppUnhealthy
            expr: argocd_app_info{health_status!="Healthy"} == 1
            for: 15m
            labels: { severity: warning }
```

> **values.yaml("스택을 어떻게 튜닝하는가")과 alerts.yaml("무엇을 문제로 보는가")을 파일로 분리**하면 리뷰가 쉬워진다. Helm은 `-f`를 여러 번 받으므로 병합된다.

---

## 메트릭 + 로그 한 화면

```yaml
grafana:
  enabled: true
  defaultDashboardsEnabled: true      # kubernetes-mixin 대시보드

  # adminPassword 는 여기 쓰지 않는다 — Git에 평문 비밀번호를 넣지 않기 위해

  additionalDataSources:
    - name: Loki
      type: loki
      access: proxy
      url: http://loki.observability.svc.cluster.local:3100
      isDefault: false
```

> 이 한 블록이 **메트릭 스파이크 → 그 시점 로그** 이동을 가능하게 한다. 관측성 스택에서 체감 효과가 가장 큰 설정이다. → `04-grafana/`

---

## GitOps로 배포하기

```
observability/                     ← Helm values 만 (무엇을 어떻게 설정할지)
├── kube-prometheus-stack/
│   ├── values.yaml
│   └── alerts.yaml
├── loki/values.yaml
└── promtail/values.yaml

gitops/apps/                       ← ArgoCD Application (어디에 배포할지)
├── kube-prometheus-stack.yaml
├── loki.yaml
└── promtail.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 87.16.1              # 버전 고정
    helm:
      valueFiles:
        - $values/observability/kube-prometheus-stack/values.yaml
        - $values/observability/kube-prometheus-stack/alerts.yaml
  destination:
    namespace: observability
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

> 차트는 업스트림 리포에서 당기고 **버전을 고정**하되, Git에는 오버라이드만 둔다. 관측성 스택 자체가 재현 가능해진다. → `../k8s-manifests/10-helm-gitops/`  
> ⚠️ CRD가 큰 차트는 ArgoCD에서 `OutOfSync`가 영구적으로 남는 경우가 있다 (어노테이션 크기 제한). `ServerSideApply=true` 옵션으로 해결한다.

---

## 용량 — 관측성 스택이 클러스터를 먹는다

```
관측 대상 워크로드보다 관측 스택이 더 많은 자원을 쓰는 상황이 흔하다.
t3.medium 2대 기준: Prometheus + Grafana + Alertmanager + Loki + promtail
                    + node-exporter + kube-state-metrics
                    → 여유가 거의 없다
```

| 파드가 `Pending`이면 | 확인 |
|---|---|
| 자원 부족 | `kubectl describe pod` → Insufficient cpu/memory |
| 노드당 파드 수 한계 | t3.medium은 ENI 기준 **최대 17 파드** |
| PVC | CSI 드라이버 없이 PVC 요청 |

> **t3.medium의 max-pods는 17이다.** 관측성 스택을 얹으면 여기에 먼저 걸린다. 노드 수를 늘리거나 t3.large로 올린다.

---

## 검증 순서

```bash
# 1) 파드가 다 떴는가
kubectl -n observability get pods

# 2) 타겟이 UP 인가 (여기서 대부분의 문제가 드러난다)
kubectl -n observability port-forward svc/kube-prometheus-stack-prometheus 9090:9090
curl -s localhost:9090/api/v1/targets \
  | jq '.data.activeTargets[] | select(.health!="up") | {job:.labels.job, lastError}'

# 3) 룰이 로드됐는가
curl -s localhost:9090/api/v1/rules | jq '.data.groups[].name'

# 4) Grafana 대시보드
kubectl -n observability port-forward svc/kube-prometheus-stack-grafana 3000:80

# 5) 로그가 들어오는가
# Grafana → Explore → Loki → {namespace="sample-app"}
```

| 증상 | 원인 |
|---|---|
| ServiceMonitor가 무시됨 | `serviceMonitorSelectorNilUsesHelmValues: false` 누락 |
| 타겟 DOWN | 포트 **이름** 불일치, NetworkPolicy, 앱이 `/metrics`를 안 냄 |
| 파드 Pending | 자원 부족, max-pods 한계, PVC |
| Loki 기동 실패 | read-only FS에 쓰기 시도 → `emptyDir` 필요 → `06-logging/` |
| 알림이 안 옴 | Prometheus firing인지 / Alertmanager 라우팅인지 구분 → `05-alerting/` |
| ArgoCD 영구 OutOfSync | CRD 어노테이션 크기 → `ServerSideApply=true` |

---

## 배운 점

- kube-prometheus-stack 하나로 **돌아가는 기본선**을 확보하고 거기서 조정한다
- Operator는 **CRD(ServiceMonitor·PrometheusRule)를 감시해 설정을 자동 생성**한다
- 덕분에 앱 팀이 ServiceMonitor를 넣기만 하면 수집이 시작된다 — **셀프서비스 관측성**
- ServiceMonitor의 `port`는 **포트 번호가 아니라 Service의 포트 이름**
- `selector`는 Deployment가 아니라 **Service의 라벨**을 본다
- **`serviceMonitorSelectorNilUsesHelmValues: false`** 가 없으면 다른 릴리스의 모니터가 무시된다
- **EKS는 컨트롤 플레인을 스크랩할 수 없다** — 타겟과 **기본 룰을 함께** 꺼야 한다
- 룰만 남기면 "데이터 없음"으로 영원히 발화한다
- PVC를 끄면 재시작 시 데이터가 사라진다 — CSI 드라이버 없이 켜면 `Pending`
- **스크래퍼에 CPU limit을 걸지 않는다** (스로틀링 → 스크랩 누락 → 오탐)
- 메모리는 limit을 건다 (OOM이 노드를 위협하므로)
- 플랫폼 자체(ArgoCD)도 관측 대상 — 배포 경로가 조용히 멈추는 걸 막는다
- values.yaml(튜닝)과 alerts.yaml(문제 정의)을 **파일로 분리**하면 리뷰가 쉽다
- Grafana에 Loki 데이터소스를 추가하는 것이 **체감 효과가 가장 큰 설정**
- 차트 버전을 고정하고 Git에는 오버라이드만 두면 스택이 재현 가능해진다
- **관측성 스택이 클러스터 자원을 크게 먹는다** — t3.medium은 max-pods 17이 먼저 걸린다
- 검증은 **타겟 상태 확인**부터 — 대부분의 문제가 거기서 드러난다
