# 07 리소스 라이트사이징

**노드 비용은 파드의 `requests` 합계가 결정한다.** 스케줄러는 requests를 기준으로 노드를 채우기 때문이다.  
실제 사용량이 0.1코어인데 2코어를 요청하면, 나머지 1.9코어는 **아무도 못 쓰는 채로 청구된다.**

---

## requests가 비용을 만든다

```
파드 A: requests.cpu = 2,  실사용 0.1
파드 B: requests.cpu = 2,  실사용 0.1
파드 C: requests.cpu = 2,  실사용 0.1
                            ↓
스케줄러: "6코어 필요" → m5.2xlarge(8코어) 한 대
실제 사용: 0.3코어 (3.75%)
                            ↓
청구서는 8코어 기준으로 나온다
```

| | requests | limits |
|---|---|---|
| 역할 | **스케줄링 기준 (예약)** | 상한 (강제) |
| 비용 영향 | **직접적** | 간접적 |
| 부족하면 | 노드가 과밀 → 성능 저하 | 스로틀링·OOMKill |
| 과하면 | **비용 낭비** | 문제 없음(메모리는 노드 위험) |

> **비용 최적화의 대상은 `limits`가 아니라 `requests`다.** limits를 줄여도 청구서는 안 바뀐다.  
> requests 기준으로 비용을 배분해야 이걸 고칠 동기가 생긴다. → `03-kubernetes-cost-allocation/`

---

## QoS 클래스

```
Guaranteed   requests == limits (모든 컨테이너)     → 축출 우선순위 최하 (가장 안전)
Burstable    requests < limits, 또는 일부만 설정     → 중간
BestEffort   requests·limits 둘 다 없음             → 축출 1순위 (위험)
```

```bash
kubectl get pod <POD> -o jsonpath='{.status.qosClass}'
```

> **노드에 메모리 압박이 오면 BestEffort → Burstable → Guaranteed 순으로 축출된다.**  
> requests를 안 쓰면 싸 보이지만, 실제로는 **가장 먼저 죽는 파드**가 된다. 비용이 아니라 안정성 문제로 돌아온다.

---

## CPU와 메모리는 다르게 다룬다

```
CPU     압축 가능(compressible)   한도를 넘으면 스로틀링 → 느려질 뿐 안 죽는다
메모리   압축 불가(incompressible) 한도를 넘으면 OOMKill → 죽는다
```

| | requests | limits |
|---|---|---|
| **CPU** | 실사용 p95 기준 | **설정하지 않는다** |
| **메모리** | 실사용 p95~p99 | **반드시 설정** (requests의 1.5~2배) |

```yaml
resources:
  requests:
    cpu: 100m           # 스케줄링 기준
    memory: 256Mi
  limits:
    memory: 512Mi       # OOM 으로 노드를 위협하지 않게
    # cpu limit 없음     ← 의도적
```

> ⚠️ **CPU limit은 스로틀링만 유발한다.** 노드에 여유 CPU가 있어도 limit에 걸려 못 쓴다. 버스티한 워크로드(요청 처리·GC·기동)가 특히 손해를 본다.  
> 이 판단은 관측성 스택과 Kyverno `require-resources` 정책에서 일관되게 적용돼 있다 — **"requests + 메모리 limit, CPU limit은 없음"**. → `../observability-lab/08-kube-prometheus-stack/` `../security-lab/04-kyverno/`

---

## 실측으로 값 정하기

```promql
# CPU: 최근 7일 p95 사용량
quantile_over_time(0.95,
  rate(container_cpu_usage_seconds_total{container!="", namespace="myapp"}[5m])[7d:5m]
)

# 메모리: 최근 7일 최댓값 (메모리는 최대치를 봐야 한다)
max_over_time(container_memory_working_set_bytes{container!="", namespace="myapp"}[7d])

# requests 대비 실사용 비율 — 1에 가까울수록 정확
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[7d]))
  / sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})
```

```
과대 요청 후보:  실사용/requests < 0.2      → requests 를 낮춘다
과소 요청 후보:  실사용/requests > 0.9      → 올린다 (스로틀링·OOM 위험)
```

| 기준 | CPU | 메모리 |
|---|---|---|
| requests | **p95** | **최댓값 × 1.1** |
| limits | 없음 | requests × 1.5~2 |

> **메모리는 평균이 아니라 최댓값을 본다.** 평균으로 잡으면 피크 때 OOMKill이 난다.  
> **관측 기간이 짧으면 안 된다.** 주간·월간 배치가 있으면 최소 7일, 가능하면 30일을 본다.

---

## VPA (Vertical Pod Autoscaler)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"          # ⭐ 추천값만 계산, 적용은 안 함
  resourcePolicy:
    containerPolicies:
      - containerName: app
        minAllowed: { cpu: 50m, memory: 128Mi }
        maxAllowed: { cpu: "2",  memory: 4Gi }
```

```bash
kubectl describe vpa myapp-vpa
# Recommendation:
#   Target:      cpu: 120m,  memory: 300Mi
#   Lower Bound: cpu: 80m,   memory: 250Mi
#   Upper Bound: cpu: 500m,  memory: 1Gi
```

| updateMode | 동작 |
|---|---|
| **`Off`** | 추천만 계산 (**권장 시작점**) |
| `Initial` | 파드 생성 시에만 적용 |
| `Auto` / `Recreate` | **파드를 재시작하며 적용** |

> **`updateMode: Off`로 추천값만 받아 사람이 매니페스트에 반영하는 방식이 실무에서 가장 안전하다.**  
> `Auto`는 파드를 재시작시킨다 — 예고 없이 재시작되는 걸 감당할 수 있는 워크로드만.  
> ⚠️ **VPA와 HPA(CPU 기준)를 같은 워크로드에 함께 쓰면 충돌한다.** HPA가 CPU로 스케일아웃하는데 VPA가 requests를 바꾸면 기준이 흔들린다. HPA는 커스텀 메트릭으로, VPA는 메모리만 쓰는 식으로 분리한다.

---

## HPA — 수평 확장

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # requests 대비 비율
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # 급격한 축소 방지
```

```
⚠️ HPA 의 CPU Utilization 은 'requests 대비 비율' 이다
   requests 를 과대하게 잡으면 → 사용률이 항상 낮게 나온다
   → HPA 가 스케일아웃하지 않는다 → 성능 문제
```

> **라이트사이징과 HPA는 함께 봐야 한다.** requests를 줄이면 사용률 계산이 바뀌어 HPA 동작도 바뀐다.  
> 매니페스트에 `replicas`를 두고 HPA도 쓰면 ArgoCD와 충돌한다 — `ignoreDifferences`로 빼거나 매니페스트에서 제거한다. → `../cicd-lab/05-argocd-advanced/`

---

## 노드 크기와 밀도

```
큰 노드                          작은 노드
├─ bin-packing 좋음               ├─ 세밀한 스케일 조정
├─ 시스템 오버헤드 비율 낮음        ├─ 장애 반경 작음
└─ 장애 반경 큼                   └─ 오버헤드 비율 높음
```

| 노드 크기 | 시스템 예약 비율 |
|---|---|
| t3.medium (2vCPU/4GB) | **상대적으로 크다** (kubelet·CNI·DaemonSet 고정 비용) |
| m5.2xlarge (8vCPU/32GB) | 작다 |

### ⚠️ 노드당 파드 수 한계

```
AWS VPC CNI 는 ENI 기반으로 파드 IP 를 할당한다
  t3.medium → max-pods 17
        ↓
DaemonSet(kube-proxy·CNI·promtail·node-exporter) 이 5~6개를 먹는다
        ↓
실제로 쓸 수 있는 건 10여 개
        ↓
CPU·메모리가 남아도 파드를 더 못 올린다  💥
```

> **작은 인스턴스에서는 CPU·메모리보다 max-pods가 먼저 걸린다.** 자원이 남는데 Pending이 나면 이걸 의심한다.  
> 관측성 스택처럼 DaemonSet이 많은 환경에서 특히 두드러진다. **비용 효율만 보고 작은 노드를 고르면 오히려 노드 수가 늘어난다.**  
> `ENABLE_PREFIX_DELEGATION=true`로 완화할 수 있다.

---

## 정리 대상 찾기

```bash
# requests 없는 파드 (BestEffort — 배분 불가 + 축출 1순위)
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.containers[].resources.requests == null)
  | "\(.metadata.namespace)/\(.metadata.name)"'

# 노드별 requests 총량 (오버커밋 확인)
kubectl describe node <NODE> | grep -A8 "Allocated resources"

kubectl top pods -A --sort-by=memory
kubectl top nodes
```

```yaml
# LimitRange 로 기본값 보장 — requests 누락 방지
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: myapp
spec:
  limits:
    - type: Container
      defaultRequest: { cpu: 100m, memory: 128Mi }
      default:        { memory: 256Mi }
      max:            { cpu: "4", memory: 8Gi }
```

> **LimitRange는 "누가 requests를 빼먹어도 최소한 배분과 스케줄링은 되게" 하는 안전망**이다.  
> Kyverno `require-resources`가 거부한다면, LimitRange는 채워준다. 둘은 상호 보완적이다. → `../security-lab/04-kyverno/`

---

## 라이트사이징 절차

```
1. 실측      7~30일 사용량 수집 (Prometheus)
2. 계산      CPU p95, 메모리 max × 1.1
3. 검증      VPA 추천값과 대조
4. 적용      비운영부터, 한 번에 하나씩
5. 관찰      스로틀링·OOMKill·지연 지표
6. 반복      분기 1회
```

```promql
# 적용 후 확인 — 이게 늘면 너무 줄인 것
sum by (namespace, pod) (rate(container_cpu_cfs_throttled_seconds_total[5m]))
sum by (namespace, pod) (increase(kube_pod_container_status_restarts_total[1h]))
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
```

> ⚠️ **너무 줄이면 OOMKill과 스로틀링으로 돌아온다.** 절감액보다 장애 비용이 크다.  
> **한 번에 하나씩, 비운영부터.** 전체를 일괄 조정하면 문제가 생겼을 때 원인을 못 찾는다. → `01-finops-basics/`

---

## 배운 점

- **노드 비용은 `requests` 합계가 결정한다** — 스케줄러가 그 기준으로 노드를 채운다
- 최적화 대상은 **`limits`가 아니라 `requests`**
- requests를 안 쓰면 **BestEffort가 되어 축출 1순위**가 된다 (싼 게 아니라 위험한 것)
- **CPU는 압축 가능(스로틀링), 메모리는 압축 불가(OOMKill)** — 다르게 다룬다
- ⚠️ **CPU limit은 설정하지 않는다** — 여유가 있어도 못 쓰게 만든다
- **메모리 limit은 반드시** 설정한다 (노드 전체를 위협하므로)
- requests는 **CPU p95 / 메모리 최댓값 × 1.1**
- **메모리는 평균이 아니라 최댓값**을 본다, 관측 기간은 최소 7일
- **VPA는 `updateMode: Off`로 추천만 받는 게 가장 안전**
- ⚠️ **VPA와 CPU 기반 HPA는 충돌한다**
- **HPA의 사용률은 requests 대비 비율** — requests가 과대하면 스케일아웃이 안 된다
- 작은 노드는 **시스템 오버헤드 비율이 높다**
- ⚠️ **max-pods가 CPU·메모리보다 먼저 걸린다** (t3.medium = 17) — 자원이 남는데 Pending이면 이것
- 비용만 보고 작은 노드를 고르면 **오히려 노드 수가 늘어난다**
- **LimitRange가 requests 누락의 안전망**, Kyverno는 거부 — 상호 보완적
- ⚠️ **너무 줄이면 OOMKill·스로틀링** — 절감액보다 장애 비용이 크다
- **한 번에 하나씩, 비운영부터** 적용하고 스로틀링·재시작 지표로 검증
