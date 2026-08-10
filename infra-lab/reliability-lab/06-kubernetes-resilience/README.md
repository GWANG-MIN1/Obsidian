# 06 Kubernetes 복원력

Kubernetes는 **선언형 + 셀프힐링**이라 기본적으로 복원력이 있다. 파드가 죽으면 다시 뜬다.  
문제는 **기본값이 안전하지 않다**는 것이다 — probe가 부정확하고, PDB가 없고, 파드가 한 노드에 몰려 있으면 셀프힐링은 작동하지 않는다.

---

## 복원력의 층위

```
컨테이너 죽음  → kubelet 이 재시작          (restartPolicy)
파드 죽음     → 컨트롤러가 재생성           (Deployment)
노드 죽음     → 다른 노드에 재스케줄        (스케줄러)
AZ 죽음       → 다른 AZ 로                 (토폴로지 분산)
리전 죽음     → 다른 리전으로              → 07-backup-dr/
```

> **각 층위마다 전제 조건이 있다.** 파드 재생성은 다른 노드에 여유가 있어야 하고, AZ 장애 대응은 파드가 여러 AZ에 흩어져 있어야 한다.  
> 전제를 안 갖추고 "Kubernetes니까 알아서 되겠지"가 가장 흔한 실패다.

---

## ⭐ Probe — 복원력의 출발점

**probe가 부정확하면 나머지 모든 장치가 무의미해진다.** 롤링 업데이트도, 셀프힐링도 probe를 믿고 동작하기 때문이다.

| Probe | 실패 시 | 목적 |
|---|---|---|
| **startup** | 재시작 | 느린 기동 보호 (다른 probe를 지연) |
| **readiness** | **엔드포인트에서 제외** (트래픽 차단) | 받을 준비가 됐는가 |
| **liveness** | **컨테이너 재시작** | 살아있는가 |

```yaml
startupProbe:                    # 기동이 느린 앱에 필수
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 10              # 최대 300초까지 기다린다

readinessProbe:
  httpGet: { path: /readyz, port: 8080 }
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3
```

### readiness와 liveness는 달라야 한다

```
/readyz  의존성 포함 검사 — DB 연결 가능한가, 캐시 워밍됐는가
/healthz 프로세스 자체만  — 이벤트 루프가 도는가
```

```
⚠️ liveness 에 의존성 검사를 넣으면
   DB 가 잠깐 느려짐 → liveness 실패 → 컨테이너 재시작
   → 재시작해도 DB 는 여전히 느림 → 무한 재시작
   → 전체 파드가 CrashLoop → DB 부하 가중 → 연쇄 장애  💥
```

> ⭐ **이게 Kubernetes 복원력에서 가장 흔하고 파괴적인 실수다.** liveness는 **"재시작하면 고쳐지는 문제"** 에만 반응해야 한다.  
> DB 장애는 재시작으로 안 고쳐진다 → liveness가 아니라 readiness의 몫이다.

### 껍데기 probe도 위험하다

```
/healthz 가 항상 200 을 반환하는 껍데기
      ↓
깨진 버전이 그대로 전량 배포된다 (롤링 업데이트가 못 막는다)
```

> **probe의 정확도가 배포 안전성을 결정한다.** → `../cicd-lab/08-deployment-strategies/`  
> 최소한 **"이 요청을 처리할 수 있는 상태인가"** 를 실제로 검사해야 한다.

---

## 우아한 종료

```
파드 종료 시작
  ├─ Endpoints 에서 제거 (전파에 시간이 걸린다)
  └─ SIGTERM 전송
        ↑ 이 둘이 동시에 일어난다
        → 이미 라우팅된 요청이 끊긴다
```

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]      # 엔드포인트 전파를 기다린다
```

```
앱이 SIGTERM 을 받으면
  ① 새 요청 수신 중단 (readiness 를 실패로)
  ② 진행 중인 요청 완료
  ③ 커넥션 정리 후 종료
```

> **배포 중 5xx의 1순위 원인이다.** Spot 중단과 Karpenter consolidation에서도 같은 상황이 반복된다. → `../cost-lab/05-spot-instances/` `../cost-lab/06-karpenter/`  
> `terminationGracePeriodSeconds`가 Spot의 2분보다 길면 강제 종료된다 — Spot 환경에서는 60초 이하가 안전하다.

---

## PodDisruptionBudget

```
자발적 중단(voluntary disruption) 시 최소 가용 파드 수를 보장
  노드 드레인, 클러스터 업그레이드, Karpenter consolidation
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2                # 또는 maxUnavailable: 1
  selector:
    matchLabels: { app: myapp }
```

| | 막는 것 | 못 막는 것 |
|---|---|---|
| PDB | 노드 드레인, 업그레이드, consolidation | **노드 하드웨어 장애, Spot 회수, OOM** |

> ⚠️ **PDB는 자발적 중단만 막는다.** 노드가 갑자기 죽는 건 못 막는다 — 그건 토폴로지 분산의 몫이다.  
> ⚠️ **`minAvailable`을 레플리카 수와 같게 잡으면 드레인이 영원히 멈춘다.** 클러스터 업그레이드가 진행되지 않는다.

```bash
kubectl -n myapp describe pdb myapp-pdb      # ALLOWED DISRUPTIONS 확인
kubectl get pdb -A
```

```promql
# 축출이 불가능한 상태 — 업그레이드가 막힌다
kube_poddisruptionbudget_status_pod_disruptions_allowed == 0
```

---

## 토폴로지 분산

```
레플리카 3개가 전부 같은 노드에 있으면
  → 노드 하나 죽음 = 전멸  (레플리카가 3개인 의미가 없다)
```

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname          # 노드별로 고르게
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels: { app: myapp }
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone     # AZ 별로도
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: { app: myapp }
```

| `whenUnsatisfiable` | 동작 |
|---|---|
| `ScheduleAnyway` | 조건을 못 맞춰도 스케줄 (soft) |
| `DoNotSchedule` | 조건 못 맞추면 **Pending** (hard) |

> **노드 레벨은 `ScheduleAnyway`, AZ 레벨은 `DoNotSchedule`** 조합이 실무에서 무난하다.  
> ⚠️ `DoNotSchedule`을 남발하면 노드가 부족할 때 파드가 Pending에 갇힌다. Spot·consolidation 환경에서 특히 위험하다.

```bash
# 실제로 흩어져 있는지 확인
kubectl -n myapp get pods -o wide
kubectl -n myapp get pods -o custom-columns=\
NAME:.metadata.name,NODE:.spec.nodeName,ZONE:.metadata.labels.topology\\.kubernetes\\.io/zone
```

> **선언만 하고 확인 안 하면 의미가 없다.** 실제로 몇 개 노드에 흩어져 있는지 눈으로 본다.

---

## 우선순위와 선점

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical
value: 1000000
preemptionPolicy: PreemptLowerPriority
description: "결제 등 핵심 워크로드"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch
value: 100
preemptionPolicy: Never
```

```
자원이 부족할 때
  높은 우선순위 파드가 스케줄되기 위해
  낮은 우선순위 파드를 축출(선점)한다
```

> **배치 잡 때문에 결제 서비스가 스케줄 안 되는 상황을 막는다.** 벌크헤드 패턴의 Kubernetes 구현이다. → `05-resilience-patterns/`  
> 시스템 컴포넌트에는 `system-cluster-critical`·`system-node-critical`이 이미 붙어 있다.

---

## 리소스 설정과 축출

```
노드 메모리 압박 시 축출 순서
  BestEffort (requests 없음)  →  Burstable  →  Guaranteed
```

| QoS | 조건 | 축출 |
|---|---|---|
| **Guaranteed** | requests == limits (전 컨테이너) | 마지막 |
| **Burstable** | requests < limits 또는 일부만 | 중간 |
| **BestEffort** | requests·limits 없음 | **1순위** |

> ⚠️ **requests를 안 쓰면 싸 보이지만 가장 먼저 죽는다.** 비용 문제가 아니라 신뢰성 문제로 돌아온다. → `../cost-lab/07-rightsizing/`  
> **메모리 limit은 반드시 설정**한다 — 없으면 한 파드가 노드 전체를 OOM으로 몰고 간다.  
> **CPU limit은 설정하지 않는다** — 스로틀링으로 지연이 늘어 오히려 신뢰성을 해친다.

```promql
# 축출 발생 추적
increase(kube_pod_status_reason{reason="Evicted"}[1h])

# OOMKilled
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}

# 스로틀링 (CPU limit 을 걸었다면)
sum by (namespace,pod) (rate(container_cpu_cfs_throttled_seconds_total[5m]))
```

---

## 컨트롤러 선택

| 컨트롤러 | 복원력 특성 |
|---|---|
| **Deployment** | 파드가 대체 가능, 순서 무관 — 스테이트리스에 적합 |
| **StatefulSet** | 안정적 이름·순서·스토리지 — 복구가 느리다 |
| **DaemonSet** | 노드마다 하나 — 노드 장애 시 함께 사라짐 |
| **Job/CronJob** | `backoffLimit`, `activeDeadlineSeconds` 설정 필수 |

```yaml
# Job 이 무한 재시도하지 않게
spec:
  backoffLimit: 4
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600     # 완료된 Job 자동 정리
```

> ⚠️ **`backoffLimit` 없이 실패하는 Job은 무한히 파드를 만든다.** 클러스터 자원과 클라우드 비용을 동시에 태운다.  
> `ttlSecondsAfterFinished`가 없으면 완료된 Job이 계속 쌓인다.

---

## 노드 장애 시 실제 흐름

```
노드 다운
  ↓ (약 40초) 노드 컨트롤러가 NotReady 표시
  ↓ (약 5분)  파드에 NoExecute taint → 축출 시작
  ↓          다른 노드에 재스케줄
  ↓          이미지 pull + 기동 + readiness
  ────────────────────────────────────────
  총 5~7분 동안 그 파드들은 서비스 불가
```

```yaml
# 중요한 워크로드는 축출을 앞당긴다
spec:
  tolerations:
    - key: node.kubernetes.io/not-ready
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 30        # 기본 300초 → 30초
    - key: node.kubernetes.io/unreachable
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 30
```

> ⭐ **"파드가 알아서 옮겨진다"는 5분 뒤의 이야기다.** 그동안 서비스가 유지되려면 **다른 노드에 이미 떠 있는 레플리카**가 있어야 한다.  
> 즉 **토폴로지 분산이 노드 장애 대응의 본체**이고, 재스케줄은 사후 복구일 뿐이다.

---

## 점검 체크리스트

```
□ readiness 와 liveness 가 다른 엔드포인트인가
□ liveness 에 의존성 검사가 들어있지 않은가        ← 가장 위험
□ startupProbe 가 있는가 (기동이 느린 앱)
□ preStop + terminationGracePeriodSeconds 설정
□ 레플리카 2개 이상인가
□ topologySpreadConstraints 로 흩어져 있는가
□ 실제로 몇 개 노드에 있는지 확인했는가
□ PDB 가 있고 ALLOWED DISRUPTIONS > 0 인가
□ requests 가 설정돼 있는가 (BestEffort 회피)
□ 메모리 limit 있음 / CPU limit 없음
□ Job 에 backoffLimit·ttlSecondsAfterFinished 있는가
```

```bash
# 한 번에 훑기
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.containers[].readinessProbe == null)
  | "readiness 없음: \(.metadata.namespace)/\(.metadata.name)"'

kubectl get deploy -A -o json | jq -r '.items[]
  | select(.spec.replicas < 2)
  | "레플리카 1개: \(.metadata.namespace)/\(.metadata.name)"'
```

---

## 배운 점

- Kubernetes는 셀프힐링이지만 **기본값이 안전하지는 않다**
- 복원력은 층위별로 전제 조건이 있다 — 전제 없이 "알아서 되겠지"가 가장 흔한 실패
- ⭐ **probe가 부정확하면 나머지 장치가 전부 무의미**해진다
- **readiness는 의존성 포함, liveness는 프로세스 자체만**
- ⭐ ⚠️ **liveness에 의존성 검사를 넣으면 무한 재시작 → 연쇄 장애**
- liveness는 **"재시작하면 고쳐지는 문제"** 에만 반응해야 한다
- 껍데기 probe는 **깨진 버전을 전량 배포**시킨다
- 배포 중 5xx의 1순위는 **엔드포인트 전파와 SIGTERM의 경합** → `preStop`
- Spot 환경에서는 `terminationGracePeriodSeconds`를 **2분보다 짧게**
- ⚠️ **PDB는 자발적 중단만 막는다** — 노드 하드웨어 장애·Spot 회수는 못 막는다
- ⚠️ `minAvailable`을 레플리카 수와 같게 잡으면 **드레인이 영원히 멈춘다**
- **레플리카 3개가 같은 노드에 있으면 레플리카가 3개인 의미가 없다**
- 노드 레벨 `ScheduleAnyway` + AZ 레벨 `DoNotSchedule` 조합이 무난
- **선언만 하고 실제 분산을 확인 안 하면 의미 없다**
- PriorityClass가 **벌크헤드의 Kubernetes 구현**
- ⚠️ **requests 없음(BestEffort)은 축출 1순위** — 비용이 아니라 신뢰성 문제
- **메모리 limit은 필수, CPU limit은 금지**
- ⚠️ **`backoffLimit` 없는 Job은 무한히 파드를 만든다**
- ⭐ **노드 장애 시 재스케줄까지 5~7분** — 그동안은 **이미 떠 있는 레플리카**가 서비스한다
- 즉 **토폴로지 분산이 본체이고 재스케줄은 사후 복구**다
