# 09 Argo Rollouts

Kubernetes의 `Deployment`는 RollingUpdate와 Recreate 두 가지만 안다. **Blue-Green과 Canary는 직접 만들어야 한다.**  
Argo Rollouts는 `Deployment`를 대체하는 `Rollout` CRD를 제공해 이 전략들을 선언형으로 쓰게 하고, **메트릭 기반 자동 판정·자동 롤백**을 붙인다.

---

## 무엇이 달라지나

```
[ Deployment ]
  kubectl set image → 롤링 → 끝
  문제가 있어도 알아서 멈추지 않는다 (probe 를 통과하는 한)

[ Rollout ]
  set image → 20% → 분석(Prometheus 질의) → 정상? → 50% → 분석 → 100%
                         │
                    실패 → 자동 중단·롤백
```

| | Deployment | Rollout |
|---|---|---|
| 지원 전략 | RollingUpdate, Recreate | + **Blue-Green, Canary** |
| 단계적 진행 | ❌ | ✅ (`steps`) |
| 메트릭 기반 판정 | ❌ | ✅ (`AnalysisTemplate`) |
| 자동 롤백 | probe 실패 시만 | **지표 악화 시** |
| 수동 승인 게이트 | ❌ | ✅ (`pause`) |
| 트래픽 제어 | 레플리카 비율 | Ingress·서비스메시 연동 |

> **핵심 차이는 "판단"이다.** Deployment는 파드가 Ready면 성공으로 본다. Rollout은 **에러율·지연 같은 실제 지표**로 성공을 정의할 수 있다.

---

## 설치

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/download/v1.7.2/install.yaml

# kubectl 플러그인
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

> GitOps 환경이라면 이것도 Application으로 배포한다. 컨트롤러는 앱보다 먼저 떠야 하므로 **낮은 sync-wave**를 준다. → `06-app-of-apps-applicationset/`

---

## Rollout 기본 구조

`Deployment`와 거의 같다. `kind`와 `strategy`만 다르다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout                       # Deployment 대신
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 5
  selector:
    matchLabels: { app: myapp }
  template:                         # 파드 템플릿은 Deployment 와 동일
    metadata:
      labels: { app: myapp }
    spec:
      containers:
        - name: app
          image: myapp:1.4.2
          ports: [{ containerPort: 8080 }]
          readinessProbe:
            httpGet: { path: /readyz, port: 8080 }

  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

```
setWeight: 20      →  트래픽 20% 를 새 버전으로
pause: {duration}  →  이만큼 기다린다
pause: {}          →  무기한 대기 (사람이 promote 해야 진행) ← 수동 승인 게이트
```

> **기존 Deployment를 Rollout으로 옮길 때**는 `workloadRef`로 참조하는 방법도 있다. 파드 템플릿을 복사하지 않아도 된다.

```yaml
spec:
  workloadRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
```

---

## Canary + 트래픽 라우팅

`setWeight`가 정확히 동작하려면 **트래픽 라우터**가 필요하다. 없으면 레플리카 비율로 근사한다.

```yaml
strategy:
  canary:
    canaryService: myapp-canary        # 새 버전용 Service
    stableService: myapp-stable        # 안정 버전용 Service
    trafficRouting:
      nginx:
        stableIngress: myapp-ingress
    steps:
      - setWeight: 10
      - pause: { duration: 2m }
      - setWeight: 30
      - pause: { duration: 2m }
      - setWeight: 60
      - pause: { duration: 2m }
```

| 라우터 | 비고 |
|---|---|
| **NGINX Ingress** | 카나리 어노테이션 활용, 가장 쉽다 |
| **ALB (AWS)** | 가중 타겟그룹 — EKS에서 자연스럽다 |
| **Istio / Linkerd** | VirtualService 가중치, 가장 정밀 |
| 없음 | 레플리카 비율로 근사 (부정확) |

```yaml
# ALB 사용 시
trafficRouting:
  alb:
    ingress: myapp-ingress
    servicePort: 80
```

> **EKS + ALB Ingress Controller 조합이면 `alb` 라우팅이 가장 손이 적게 간다.** 서비스메시를 도입하지 않고도 정확한 가중치 제어가 된다. → `../terraform-lab/09-aws-vpc-eks/`

---

## AnalysisTemplate — 자동 판정의 핵심

**Prometheus에 질의해 성공/실패를 판정**한다. 여기가 Rollouts를 쓰는 진짜 이유다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: myapp
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m                 # 1분마다 측정
      count: 5                     # 5회 측정
      successCondition: result[0] >= 0.99
      failureLimit: 2              # 2회 실패하면 롤백
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.observability:9090
          query: |
            sum(rate(http_requests_total{
              service="{{args.service-name}}", status!~"5.."
            }[2m]))
            /
            sum(rate(http_requests_total{
              service="{{args.service-name}}"
            }[2m]))

    - name: p95-latency
      interval: 1m
      count: 5
      successCondition: result[0] <= 0.5
      failureLimit: 2
      provider:
        prometheus:
          address: http://kube-prometheus-stack-prometheus.observability:9090
          query: |
            histogram_quantile(0.95,
              sum by (le) (rate(http_request_duration_seconds_bucket{
                service="{{args.service-name}}"
              }[2m]))
            )
```

```yaml
# Rollout 에서 사용
strategy:
  canary:
    analysis:
      templates:
        - templateName: success-rate
      args:
        - name: service-name
          value: myapp-canary
      startingStep: 1              # 1단계부터 분석 시작
    steps:
      - setWeight: 10
      - pause: { duration: 5m }
      - setWeight: 50
      - pause: { duration: 5m }
      - setWeight: 100
```

| 필드 | 의미 |
|---|---|
| `interval` | 측정 주기 |
| `count` | 총 측정 횟수 |
| `successCondition` | 이 조건이 참이면 성공 |
| `failureLimit` | 실패 허용 횟수 (넘으면 롤백) |
| `inconclusiveLimit` | 판정 불가 허용 횟수 (데이터 없음 등) |

### ⚠️ 데이터가 없으면 어떻게 되나

```
새 버전에 트래픽이 10% 밖에 안 가는데 요청이 적으면
      ↓
분모가 0 → 쿼리 결과가 NaN/빈 값
      ↓
successCondition 평가 불가 → Inconclusive
```

> **트래픽이 적은 서비스에서는 카나리 분석이 성립하지 않는다.** 통계적으로 의미 있는 표본이 모이려면 시간이나 비중이 충분해야 한다.  
> `count`와 `interval`을 늘리거나, 초기 `setWeight`를 높이거나, 애초에 카나리를 쓰지 않는 게 답일 수 있다.

### 다른 프로바이더

```yaml
# 잡 실행 결과로 판정 (스모크 테스트)
provider:
  job:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: test
              image: curlimages/curl
              command: ["sh", "-c", "curl -f http://myapp-canary/healthz"]

# 웹 요청
provider:
  web:
    url: https://api.example.com/check
    jsonPath: "{$.status}"
```

| 프로바이더 | 용도 |
|---|---|
| **prometheus** | 에러율·지연 (가장 흔함) |
| **job** | 스모크·통합 테스트 |
| web | 외부 시스템 판정 |
| datadog / newrelic / cloudwatch | 해당 APM 사용 시 |

---

## Blue-Green

```yaml
strategy:
  blueGreen:
    activeService: myapp-active         # 실제 트래픽
    previewService: myapp-preview       # 검증용 (새 버전에 연결)
    autoPromotionEnabled: false         # 수동 승인 필요
    scaleDownDelaySeconds: 300          # 전환 후 구버전 유지 시간 (롤백 대비)
    prePromotionAnalysis:               # 전환 '전' 검증
      templates:
        - templateName: smoke-test
    postPromotionAnalysis:              # 전환 '후' 검증
      templates:
        - templateName: success-rate
```

```
새 버전 배포 → preview Service 에 연결 (트래픽 0%)
      ↓
prePromotionAnalysis (스모크 테스트)
      ↓
autoPromotionEnabled: false → 사람이 promote
      ↓
active Service 를 새 버전으로 전환 (즉시)
      ↓
postPromotionAnalysis → 실패 시 즉시 롤백
      ↓
scaleDownDelaySeconds 후 구버전 축소
```

> **`scaleDownDelaySeconds`가 롤백 창이다.** 이 시간 안에는 구버전 파드가 살아 있어 되돌리기가 즉시 된다. 너무 짧게 잡으면 Blue-Green의 이점이 사라진다.

---

## 운영 명령

```bash
# 진행 상황 실시간 확인 (가장 많이 쓴다)
kubectl argo rollouts get rollout myapp -n myapp --watch

# 이미지 변경 = 롤아웃 시작
kubectl argo rollouts set image myapp app=myapp:1.4.3 -n myapp

# pause: {} 에서 다음 단계로
kubectl argo rollouts promote myapp -n myapp
kubectl argo rollouts promote myapp -n myapp --full     # 남은 단계 전부 건너뛰기

# 중단 (안정 버전 유지)
kubectl argo rollouts abort myapp -n myapp

# 되돌리기
kubectl argo rollouts undo myapp -n myapp
kubectl argo rollouts undo myapp -n myapp --to-revision=3

# 분석 결과 확인
kubectl -n myapp get analysisrun
kubectl -n myapp describe analysisrun <NAME>

# 로컬 대시보드
kubectl argo rollouts dashboard      # localhost:3100
```

```
Rollout 상태
  Progressing   진행 중
  Paused        대기 (수동 승인 또는 duration)
  Healthy       완료
  Degraded      실패 (분석 실패·롤백됨)
```

---

## ArgoCD와 함께 쓸 때

```yaml
# Rollout 리소스도 Git 으로 관리된다
# 이미지 태그 갱신 커밋 → ArgoCD 동기화 → Rollout 시작 → 자동 분석
```

| 주의 | 설명 |
|---|---|
| **`kubectl argo rollouts promote`는 클러스터 조작** | Git에 안 남는다. selfHeal과 충돌하지 않는지 확인 |
| Health 판정 | ArgoCD는 Rollout이 `Paused`면 Progressing으로 본다 |
| `abort` 후 상태 | Git은 새 이미지, 클러스터는 구버전 → **OutOfSync** |

> **abort하면 Git과 클러스터가 어긋난다.** 진짜 롤백은 **Git에서 이미지 태그를 되돌리는 커밋**(`git revert`)이다. `abort`는 응급 정지이지 롤백의 완결이 아니다. → `07-gitops-repo-strategy/`

---

## 도입 판단

```
관측성이 없다        →  Rollouts 를 써도 판정할 지표가 없다. 먼저 계측한다.
트래픽이 적다        →  카나리 분석이 통계적으로 성립하지 않는다.
배포가 드물다        →  RollingUpdate 로 충분하다.
장애 비용이 크다     →  ✅ Rollouts 의 값어치가 나온다.
배포가 잦다          →  ✅ 자동 판정 없이는 사람이 못 따라간다.
```

> **Rollouts는 관측성 위에 세우는 것이다.** RED 지표(요청률·에러율·지연)가 이미 수집되고 있어야 AnalysisTemplate을 쓸 수 있다. → `../observability-lab/01-observability-basics/`  
> 순서는 **계측 → 대시보드 → 알림 → 자동 판정**이다. 건너뛰면 "지표가 없어서 분석이 항상 Inconclusive"가 된다.

---

## 배운 점

- Deployment는 **RollingUpdate/Recreate만** — Blue-Green·Canary는 Rollout이 제공
- 핵심 차이는 **"판단"** — Deployment는 Ready면 성공, Rollout은 **지표로 성공을 정의**
- `steps`의 `setWeight`/`pause`로 단계적 진행, `pause: {}`는 **수동 승인 게이트**
- 기존 Deployment는 **`workloadRef`** 로 참조해 옮길 수 있다
- `setWeight`가 정확하려면 **트래픽 라우터**(NGINX·ALB·Istio)가 필요하다
- EKS라면 **ALB 가중 타겟그룹**이 서비스메시 없이 정확한 제어를 준다
- **AnalysisTemplate이 Rollouts를 쓰는 진짜 이유** — Prometheus 질의로 자동 판정
- `failureLimit`을 넘으면 자동 중단·롤백
- ⚠️ **트래픽이 적으면 분석이 성립하지 않는다** (분모 0 → Inconclusive)
- 프로바이더는 prometheus 외에 **job(스모크 테스트)**, web 등이 있다
- Blue-Green의 `scaleDownDelaySeconds`가 **롤백 창** — 너무 짧으면 이점이 사라진다
- `prePromotionAnalysis`(전환 전)와 `postPromotionAnalysis`(전환 후)를 나눠 건다
- **`abort`는 응급 정지이지 롤백의 완결이 아니다** — Git과 어긋난 채 OutOfSync
- 진짜 롤백은 **Git에서 태그를 되돌리는 커밋**
- **Rollouts는 관측성 위에 세운다** — 계측 → 대시보드 → 알림 → 자동 판정 순서
