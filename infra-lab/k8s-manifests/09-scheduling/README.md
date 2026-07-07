# 09 스케줄링 & 오토스케일링

kube-scheduler는 새 Pod를 **어느 Node에 놓을지** 결정한다.  
레이블·affinity·taint로 배치를 제어하고, requests/limits로 자원을 관리하며, HPA로 부하에 따라 자동 확장한다.

---

## 레이블 & 셀렉터 (모든 것의 접착제)

레이블은 오브젝트에 붙이는 key-value 태그. Kubernetes의 거의 모든 연결(Service→Pod, Controller→Pod, 스케줄링)이 레이블로 이뤄진다.

```bash
kubectl label pod nginx env=prod tier=frontend
kubectl get pods -l env=prod              # 셀렉터로 필터
kubectl get pods -l 'env in (prod,stg)'   # 집합 셀렉터
kubectl get pods --show-labels
```

```yaml
metadata:
  labels:
    app: web
    env: prod
    version: v2
```

> 권장 표준 레이블: `app.kubernetes.io/name`, `app.kubernetes.io/version`, `app.kubernetes.io/component`, `app.kubernetes.io/managed-by`. 일관된 레이블 규칙이 대규모 운영·조회를 좌우한다.

---

## nodeSelector (가장 단순한 배치 제어)

특정 레이블을 가진 Node에만 Pod를 배치한다.

```bash
kubectl label node node-1 disktype=ssd
```

```yaml
spec:
  nodeSelector:
    disktype: ssd       # 이 레이블 있는 노드에만
```

---

## Affinity / Anti-Affinity (유연한 배치)

nodeSelector보다 표현력이 높다. "반드시(required)"와 "가급적(preferred)"을 구분.

### Node Affinity — 특정 노드 선호/강제

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # 강제
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution:  # 선호 (가중치)
        - weight: 50
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["ap-northeast-2a"]
```

### Pod Anti-Affinity — Pod 분산 (고가용성)

같은 앱의 복제본을 **서로 다른 노드**에 흩뿌려 단일 노드 장애에 대비.

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname   # 노드 단위로 분산
```

| 종류 | 의미 |
|------|------|
| **nodeAffinity** | 특정 노드에 배치 |
| **podAffinity** | 특정 Pod와 **같은** 위치에 (캐시 근접 등) |
| **podAntiAffinity** | 특정 Pod와 **다른** 위치에 (분산·HA) |

---

## Taint & Toleration (노드가 Pod를 밀어냄)

affinity가 "Pod가 노드를 고르는" 것이라면, **taint는 노드가 Pod를 거부**하는 반대 방향이다.

```bash
kubectl taint nodes gpu-node dedicated=gpu:NoSchedule
```

- **Taint** (노드에): "이 조건을 견디는(tolerate) Pod만 받겠다"
- **Toleration** (Pod에): "나는 그 taint를 견딜 수 있다"

```yaml
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
```

| effect | 동작 |
|--------|------|
| `NoSchedule` | toleration 없으면 배치 안 함 |
| `PreferNoSchedule` | 가급적 배치 안 함 (강제 아님) |
| `NoExecute` | 배치 거부 + 기존 Pod도 퇴거 |

> 활용: GPU 노드에 taint → GPU 워크로드(toleration 보유)만 그 비싼 노드에 배치. Control Plane 노드도 기본 taint가 걸려 일반 워크로드가 안 올라간다.

---

## Requests & Limits (자원 관리)

Pod가 요구/사용할 CPU·메모리를 지정한다. 스케줄링과 안정성의 근간.

```yaml
spec:
  containers:
    - name: app
      image: myapp
      resources:
        requests:          # 스케줄링 기준 (최소 보장)
          cpu: "250m"      # 0.25 vCPU
          memory: "256Mi"
        limits:            # 상한 (초과 시 제재)
          cpu: "500m"
          memory: "512Mi"
```

| 항목 | 의미 |
|------|------|
| **requests** | 이만큼은 **보장**받음. 스케줄러가 이 값으로 배치 결정 |
| **limits** | 이 이상 못 씀. CPU 초과=스로틀, **메모리 초과=OOMKill** |

- CPU 단위: `1` = 1 vCPU, `500m` = 0.5 vCPU
- 메모리 단위: `Mi`(2^20), `Gi`(2^30)

### QoS 클래스 (자원 압박 시 퇴거 우선순위)

| 클래스 | 조건 | 퇴거 우선순위 |
|--------|------|--------------|
| **Guaranteed** | requests == limits (전 컨테이너) | 가장 나중 (안전) |
| **Burstable** | requests < limits | 중간 |
| **BestEffort** | requests·limits 없음 | 가장 먼저 퇴거 |

> requests를 안 주면 스케줄러가 자원 계획을 못 세워 노드가 과밀해진다. **모든 프로덕션 Pod엔 requests/limits를 지정**하고, 중요한 워크로드는 Guaranteed로 둔다.

---

## HPA (Horizontal Pod Autoscaler)

부하(CPU·메모리·커스텀 지표)에 따라 **Pod 복제 수를 자동 조절**한다. metrics-server 필요.

```bash
kubectl autoscale deployment web --min=2 --max=10 --cpu-percent=70
kubectl get hpa
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70   # 평균 CPU 70% 목표
```

동작: 현재 사용률이 목표를 넘으면 Pod를 늘리고, 낮으면 줄인다. **HPA는 requests 기준 %로 계산**하므로 requests 설정이 필수.

### 스케일링 3종 비교

| 종류 | 대상 | 설명 |
|------|------|------|
| **HPA** | Pod **개수** | 부하에 따라 복제 수 가감 (가장 일반적) |
| **VPA** | Pod **크기** | requests/limits 자동 조정 |
| **Cluster Autoscaler** | **Node 수** | Pod가 배치될 자리가 없으면 노드 추가 (EKS) |

> 실무 조합: **HPA(파드 수) + Cluster Autoscaler(노드 수)**. 트래픽 급증 → HPA가 Pod를 늘림 → 자리가 모자라면 Cluster Autoscaler가 노드를 추가. EKS에선 Karpenter가 이 역할을 더 유연하게 수행.

---

## 배운 점

- **레이블·셀렉터**가 스케줄링·연결의 접착제 — 일관된 규칙이 중요
- 배치 제어: `nodeSelector`(단순) < affinity(유연) / **taint·toleration**(노드가 거부)
- podAntiAffinity로 복제본을 노드에 분산 → 고가용성
- **requests(보장·스케줄 기준) / limits(상한, 메모리 초과=OOMKill)**, QoS로 퇴거 순위 결정
- **HPA**(파드 수) + **Cluster Autoscaler/Karpenter**(노드 수)로 탄력적 확장
