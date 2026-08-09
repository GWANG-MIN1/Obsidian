# 03 Kubernetes 비용 배분

AWS 청구서는 **노드(EC2) 단위**로 온다. 그런데 노드 하나에는 여러 팀의 파드 수십 개가 섞여 있다.  
"이 노드 $50이 어느 팀 것인가"에 답하는 것이 Kubernetes 비용 배분의 문제다.

---

## 왜 어려운가

```
AWS 청구서
  i-0abc123 (t3.medium, SPOT)  $12.40
        │
        ▼  이 노드 위에는
  team-a/api      × 3 파드
  team-b/worker   × 2 파드
  platform/argocd × 1 파드
  kube-system/*   × 8 파드     ← 공통 오버헤드
  (유휴 CPU 40%)               ← 아무도 안 쓰는 부분
```

| 배분해야 할 것 | 어려운 이유 |
|---|---|
| 파드가 쓴 자원 | 노드 비용을 파드 비율로 나눠야 한다 |
| **유휴 자원** | 아무도 안 쓴 40%는 누구 몫인가 |
| 시스템 파드 | kube-system·CNI·CSI는 공통 비용 |
| 파드 수명 | 5분 산 파드와 30일 산 파드 |
| 노드 종류 | Spot과 온디맨드의 단가가 다르다 |

> **"AWS 태그로는 노드까지밖에 못 간다"** 는 게 핵심 제약이다. 파드 단위 배분은 Kubernetes 쪽 데이터가 필요하다.

---

## 배분의 기준 — requests vs usage

```
① requests 기준   파드가 '요청한' 자원으로 나눈다
② usage 기준      파드가 '실제로 쓴' 자원으로 나눈다
③ max(둘 중 큰 값) 실무에서 흔히 쓰는 절충
```

| 기준 | 장점 | 단점 |
|---|---|---|
| **requests** | 스케줄러가 실제로 예약한 몫 = 다른 파드가 못 씀 | 과대 요청해도 그만큼 청구 |
| **usage** | 실사용 반영, 공정해 보인다 | **과대 요청의 비용이 아무에게도 안 간다** |
| max(req, usage) | 낭비에 책임을 지운다 | 계산이 복잡 |

```
requests 로 배분해야 하는 이유
  파드가 CPU 2코어를 요청하고 0.1 만 쓰면
  → 나머지 1.9 는 다른 파드가 쓸 수 없다 (스케줄러가 예약해뒀다)
  → 실제로 노드를 점유한 것이므로 비용도 그 팀 몫이다
```

> **requests 기준 배분이 라이트사이징의 동기를 만든다.** 실사용 기준으로 청구하면 과대 요청을 고칠 이유가 없어진다.  
> 이게 requests를 제대로 설정하게 만드는 가장 강력한 장치다. → `07-rightsizing/`

---

## 유휴 비용 (Idle Cost)

```
노드 비용 $100
├── 파드 requests 합계 60%  → $60 (팀별 배분 가능)
└── 유휴 40%                → $40 ← 누구 몫인가?
```

| 배분 방식 | 의미 |
|---|---|
| **플랫폼 팀 부담** | 클러스터 효율은 플랫폼 책임 (권장 시작점) |
| **팀별 비례 배분** | 사용 비율대로 나눠 부담 |
| 별도 표시 | 배분하지 않고 지표로만 관리 |

> **유휴 비용을 팀에 나눠 부담시키면 "내가 왜 남의 낭비를"이 된다.** 반면 플랫폼 팀이 전부 지면 개선 동기가 플랫폼에만 생긴다.  
> 실무에서는 **유휴율 자체를 플랫폼 팀의 KPI로 삼고**, 배분은 requests 기준으로 하는 조합이 많다.

```promql
# 클러스터 유휴율 — 플랫폼 팀 지표
1 - (
  sum(kube_pod_container_resource_requests{resource="cpu"})
  / sum(kube_node_status_allocatable{resource="cpu"})
)
```

> 유휴율이 높으면 **노드가 너무 크거나, 너무 많거나, bin-packing이 나쁜 것**이다. Karpenter의 consolidation이 직접 겨냥하는 지표다. → `06-karpenter/`

---

## OpenCost / Kubecost

```
OpenCost   CNCF 오픈소스, 배분 엔진 (Kubecost 의 코어)
Kubecost   OpenCost + UI + 추천 + 알림 (무료 티어 + 상용)
```

```
Prometheus 메트릭 ──┐
(requests·usage)    ├──▶ OpenCost ──▶ 네임스페이스·라벨별 비용
클라우드 가격 정보 ──┘                   (API / Prometheus 메트릭)
```

```bash
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost -n opencost --create-namespace

kubectl -n opencost port-forward svc/opencost 9003:9003

# 네임스페이스별 7일 비용
curl -s "http://localhost:9003/allocation/compute?window=7d&aggregate=namespace" | jq

# 라벨(team)별 배분
curl -s "http://localhost:9003/allocation/compute?window=1d&aggregate=label:team" | jq
```

| 집계 기준 | 용도 |
|---|---|
| `namespace` | 가장 흔함 — 팀별로 네임스페이스를 나눴다면 |
| `label:team` | 네임스페이스가 팀과 일치하지 않을 때 |
| `controller` | Deployment 단위 |
| `pod` | 개별 파드 (디버깅) |

> **OpenCost는 이미 있는 Prometheus를 재사용한다.** `kube-state-metrics`와 `cAdvisor` 메트릭이 필요하므로 kube-prometheus-stack이 깔려 있으면 그대로 붙는다. → `../observability-lab/08-kube-prometheus-stack/`  
> 클라우드 실제 청구액과 맞추려면 **CUR 연동**이 필요하다. 안 하면 공시 요금 기준 추정치다 — Spot·SP 할인이 반영되지 않아 실제보다 비싸게 나온다.

---

## Prometheus만으로 근사하기

도구를 안 깔고도 대략적인 배분은 가능하다.

```promql
# 네임스페이스별 CPU requests 비중
sum by (namespace) (kube_pod_container_resource_requests{resource="cpu"})
  / scalar(sum(kube_node_status_allocatable{resource="cpu"}))

# 네임스페이스별 메모리 requests 비중
sum by (namespace) (kube_pod_container_resource_requests{resource="memory"})
  / scalar(sum(kube_node_status_allocatable{resource="memory"}))
```

```
네임스페이스 비용 ≈ 클러스터 월 비용 × CPU 비중·메모리 비중의 가중 평균
```

> **정확하진 않지만 "어느 팀이 큰가"는 충분히 보인다.** 배분 논의를 시작하기에는 이것으로 족하다.  
> 도구를 먼저 깔고 문화를 만들려 하지 말고, **거친 숫자라도 먼저 보여주는 게 순서다.** → `01-finops-basics/`

---

## 배분을 가능하게 만드는 전제

```
① 모든 파드에 requests 가 있다     ← 없으면 배분 자체가 불가능
② 네임스페이스·라벨이 팀과 매핑된다
③ 시스템 파드가 구분된다
```

```bash
# requests 없는 파드 찾기 (배분 불가 대상)
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.containers[].resources.requests == null)
  | "\(.metadata.namespace)/\(.metadata.name)"'
```

```yaml
# Kyverno 로 강제 — 실제 정책
# require-resources: CPU/메모리 requests + 메모리 limit 필수
```

> **`require-resources` 정책은 보안 정책이 아니라 사실상 비용·안정성 정책이다.** requests가 없으면 스케줄러가 노드를 제대로 채우지 못하고, 비용 배분도 불가능해진다.  
> 그래서 카테고리가 Reliability로 붙어 있다 — 같은 admission 게이트에서 막는 게 효율적이기 때문이다. → `../security-lab/04-kyverno/`

---

## 공유 비용 처리

```
클러스터 공통 비용
├── EKS 컨트롤 플레인 (시간당 고정)
├── kube-system (CNI, CoreDNS, kube-proxy)
├── 플랫폼 스택 (ArgoCD, 관측성, Kyverno)
├── NAT Gateway
└── 유휴 자원
```

| 방식 | 설명 |
|---|---|
| **플랫폼 팀 예산** | 공통 비용은 플랫폼이 부담 (단순·권장) |
| 균등 분배 | 팀 수로 나눈다 |
| 비례 분배 | 사용 비중대로 |

> ⚠️ **관측성 스택이 관측 대상보다 비쌀 수 있다.** Prometheus + Grafana + Loki + promtail + node-exporter가 t3.medium 2대 클러스터에서는 전체의 절반 가까이를 차지한다.  
> 공유 비용을 별도로 집계해두면 "플랫폼 오버헤드가 몇 %인가"를 관리할 수 있다. → `09-observability-cost/`

---

## 멀티테넌시와 쿼터

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.memory: 80Gi
    persistentvolumeclaims: "10"
    count/deployments.apps: "50"
```

```yaml
# 요청을 안 쓴 컨테이너에 기본값 부여
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
    - type: Container
      default:            { memory: 256Mi }      # limits 기본값
      defaultRequest:     { cpu: 100m, memory: 128Mi }
      max:                { cpu: "4", memory: 8Gi }
```

> **ResourceQuota는 비용 상한을 강제하는 유일한 클러스터 내장 수단이다.** 팀이 무한정 파드를 띄우는 걸 막는다.  
> **LimitRange는 requests 누락을 자동으로 메워준다** — 배분 가능성을 보장하는 안전망이다. → `../k8s-manifests/08-namespace-rbac/`

---

## 배운 점

- AWS 청구서는 **노드 단위**로 온다 — 파드 단위 배분은 Kubernetes 데이터가 필요
- 배분 기준은 **requests / usage / max(둘)**
- **requests 기준으로 배분해야 라이트사이징 동기가 생긴다** — 예약한 자원은 남이 못 쓴다
- usage 기준은 공정해 보이지만 **과대 요청의 비용이 아무에게도 안 간다**
- **유휴 비용은 플랫폼 팀 KPI로 삼고**, 배분은 requests 기준이 실무적 조합
- 유휴율이 높으면 노드가 크거나·많거나·bin-packing이 나쁜 것
- **OpenCost는 기존 Prometheus를 재사용**한다 (kube-state-metrics + cAdvisor)
- CUR 연동 없이는 **공시 요금 기준 추정치** — Spot·SP 할인이 반영 안 된다
- 도구 없이 **PromQL로 requests 비중만 봐도** 어느 팀이 큰지는 충분히 보인다
- 도구를 먼저 깔지 말고 **거친 숫자라도 먼저 보여준다**
- 배분의 전제는 **모든 파드에 requests가 있는 것** — 없으면 배분 자체가 불가능
- `require-resources` 정책은 사실상 **비용·안정성 정책**
- ⚠️ **관측성 스택이 관측 대상보다 비쌀 수 있다** — 공유 비용을 별도 집계한다
- **ResourceQuota가 비용 상한을 강제하는 유일한 내장 수단**
- **LimitRange가 requests 누락을 메워** 배분 가능성을 보장한다
