# 05 Spot 인스턴스

Spot은 **AWS의 남는 용량을 싸게 빌리는 것**이다. 60~90% 싸지만 **AWS가 필요해지면 2분 예고 후 회수**한다.  
"싸니까 쓴다"가 아니라 **"중단을 감당할 수 있게 설계했으니 쓴다"** 가 맞는 순서다.

---

## 중단 흐름

```
AWS 가 용량이 필요해짐
      │
      ▼
  중단 예고 (2분 전)  ── EC2 메타데이터 / EventBridge
      │
      ▼
  노드에 taint 부착 (관리형 노드그룹·Karpenter가 처리)
      │
      ▼
  파드 축출 (eviction) → 다른 노드로 재스케줄
      │
      ▼
  인스턴스 종료
```

> **2분은 생각보다 짧다.** 그 안에 파드가 다른 노드에서 뜨고 Ready가 되어야 무중단이 된다.  
> 이미지 pull에 1분이 걸리면 이미 늦는다. **작은 이미지와 빠른 기동이 Spot 적합성의 실질적 조건**이다. → `../cicd-lab/04-container-image-pipeline/`

---

## 적합성 판단

| 적합 | 부적합 |
|---|---|
| 스테이트리스 웹·API (레플리카 다수) | **싱글톤 스테이트풀** (마스터 노드) |
| 배치·큐 워커 (재시도 가능) | 진행 상황을 못 잃는 장기 작업 |
| CI 러너 | 라이선스가 노드에 묶인 워크로드 |
| dev·stg 전체 | 로컬 디스크에 상태를 쓰는 앱 |
| 재스케줄이 빠른 워크로드 | 기동에 5분 걸리는 앱 |

```
판단 질문
  □ 이 파드가 갑자기 죽어도 되는가
  □ 다른 노드에서 2분 안에 뜰 수 있는가
  □ 레플리카가 2개 이상인가
  □ 상태를 외부(DB·S3)에 두는가
```

> **레플리카 1개짜리를 Spot에 올리면 중단 = 다운타임이다.** Spot의 전제는 항상 복수 레플리카다.  
> 반대로 **매일 파괴하는 dev 클러스터라면 중단 비용이 사실상 0**이다. 그래서 `node_capacity_type = "SPOT"`이 정당하다.

---

## 다양화 (Diversification)

**한 인스턴스 타입만 쓰면 그 타입의 용량이 부족할 때 전멸한다.**

```
❌ instance_types = ["t3.medium"]
      → t3.medium 풀이 마르면 전체 노드가 회수된다

✅ instance_types = ["t3.medium", "t3a.medium", "t2.medium",
                     "m5.large", "m5a.large", "m6i.large"]
      → 한 풀이 말라도 다른 풀에서 확보
```

| 다양화 축 | 효과 |
|---|---|
| **인스턴스 타입** | 풀이 분산된다 (가장 중요) |
| **가용영역** | AZ별 용량 상황이 다르다 |
| 세대·제조사 | t3/t3a, m5/m5a/m6i |

```hcl
# 관리형 노드 그룹 — 여러 타입 지정
node_instance_types = ["t3.medium", "t3a.medium", "m5.large"]
node_capacity_type  = "SPOT"
```

> **다양화가 Spot 안정성의 90%다.** 중단 자체를 막을 수는 없지만, "동시에 전부 사라지는" 상황은 막을 수 있다.  
> Karpenter는 이걸 자동으로 한다 — 요구 조건만 주면 사용 가능한 타입 중 알아서 고른다. → `06-karpenter/`

```bash
# 중단 위험·절감률 확인
aws ec2 describe-spot-price-history --instance-types t3.medium \
  --product-descriptions "Linux/UNIX" --max-items 5

aws ec2 get-spot-placement-scores --instance-types t3.medium m5.large \
  --target-capacity 10 --region-names ap-northeast-2
```

---

## 중단을 견디게 만들기

### ① PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: myapp
spec:
  minAvailable: 2              # 또는 maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

> **PDB는 축출 시 최소 가용 파드 수를 보장한다.** 노드가 여러 개 동시에 회수될 때 전부 죽는 걸 막는다.  
> ⚠️ **PDB는 자발적 축출(drain)만 막는다.** Spot 회수 자체를 지연시키지는 못한다 — 2분 뒤 인스턴스는 사라진다. PDB의 역할은 **"동시에 여러 노드를 비우는 과정"** 을 순차화하는 것이다.  
> ⚠️ `minAvailable`을 레플리카 수와 같게 잡으면 **아무 파드도 축출 못 해 노드 드레인이 영원히 멈춘다.**

### ② 안티어피니티 — 파드를 흩는다

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname     # 노드별로 고르게
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels: { app: myapp }
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone  # AZ 별로도
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels: { app: myapp }
```

> **레플리카 3개가 전부 같은 노드에 있으면 노드 하나 회수로 전멸한다.** `topologySpreadConstraints`가 이를 막는다.  
> `whenUnsatisfiable: DoNotSchedule`로 강제하면 노드가 부족할 때 파드가 Pending에 갇힌다 — Spot 환경에서는 `ScheduleAnyway`가 안전하다. → `../k8s-manifests/09-scheduling/`

### ③ 우아한 종료

```yaml
spec:
  terminationGracePeriodSeconds: 60      # 2분 안에 끝나야 한다
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]      # 엔드포인트 전파 대기
```

> 배포 시 5xx가 나는 원인과 동일하다. **Spot 중단은 그 상황이 예고 없이 반복되는 것**이다. → `../cicd-lab/08-deployment-strategies/`

### ④ 중단 핸들러

```
관리형 노드 그룹  → AWS 가 중단 예고를 받아 노드를 cordon·drain 한다 (자동)
Karpenter        → 중단 이벤트를 구독해 사전에 대체 노드를 띄운다
자체 관리 ASG    → aws-node-termination-handler 를 직접 설치
```

> **EKS 관리형 노드 그룹이나 Karpenter를 쓰면 중단 처리가 기본 제공된다.** 별도 핸들러를 깔 필요가 없다.  
> Karpenter는 **중단 예고를 받으면 대체 노드를 먼저 띄우고 나서** 파드를 옮기므로 공백이 더 짧다.

---

## 혼합 전략

```
안정성이 필요한 최소 용량 → 온디맨드 (또는 Savings Plans)
그 위의 변동 용량         → Spot
```

```yaml
# Karpenter — 온디맨드 우선, Spot 확장
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: mixed
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]      # 둘 다 허용 (Spot 우선 시도)
```

```yaml
# 중요한 워크로드는 온디맨드에만
spec:
  nodeSelector:
    karpenter.sh/capacity-type: on-demand
```

```yaml
# Spot 노드에 taint 를 두고, 감당 가능한 워크로드만 toleration
tolerations:
  - key: "spot"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

> **taint/toleration으로 "Spot에 올라가도 되는 워크로드"를 명시적으로 표시**하는 게 안전하다. 기본은 온디맨드, 감당 가능한 것만 옵트인.  
> 반대 방향(기본 Spot, 중요한 것만 온디맨드)은 실수했을 때 대가가 크다.

---

## 검증 — 실제로 견디는지

```bash
# 노드의 capacity type 확인
kubectl get nodes -L eks.amazonaws.com/capacityType
kubectl get nodes -L karpenter.sh/capacity-type

# Spot 노드 비율 (PromQL)
count(kube_node_labels{label_eks_amazonaws_com_capacity_type="SPOT"})
  / count(kube_node_info)
```

```bash
# 중단을 시뮬레이션한다 — 실제로 노드를 하나 비워본다
kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -A -w
```

> **"Spot을 켰다"와 "중단을 견딘다"는 다르다.** Kyverno 정책의 대조 실험, NetworkPolicy 차단 테스트와 같은 문제다 — **실제로 노드를 비워보지 않으면 검증된 게 아니다.** → `../security-lab/04-kyverno/`  
> drain으로 파드가 안 옮겨지거나 서비스가 끊기면, 실제 Spot 회수 때도 똑같이 끊긴다.

---

## 자주 겪는 문제

| 증상 | 원인 |
|---|---|
| 노드가 계속 회수됨 | 인스턴스 타입 다양화 부족 → 타입 추가 |
| 파드가 Pending에 갇힘 | Spot 용량 부족 → 온디맨드 폴백 구성 |
| drain이 안 끝남 | PDB `minAvailable`이 너무 빡빡 |
| 중단 시 5xx | preStop·readinessProbe 미비 |
| 특정 AZ만 회수 | AZ 다양화 필요 |
| 재스케줄이 느림 | 이미지가 큼 → 이미지 최적화 |

> **"Spot 용량 부족"은 Spot 자체의 문제가 아니라 다양화 설계의 문제인 경우가 대부분이다.**

---

## 배운 점

- Spot은 **"싸니까"가 아니라 "중단을 감당할 수 있게 설계했으니"** 쓰는 것
- 중단은 **2분 예고 후 회수** — 그 안에 다른 노드에서 Ready가 되어야 한다
- **작은 이미지와 빠른 기동이 Spot 적합성의 실질적 조건**
- **레플리카 1개짜리를 Spot에 올리면 중단 = 다운타임**
- 매일 파괴하는 dev 클러스터는 **중단 비용이 사실상 0** — Spot이 정당하다
- ⭐ **다양화가 Spot 안정성의 90%** — 한 타입만 쓰면 그 풀이 마를 때 전멸
- 다양화 축: 인스턴스 타입 > 가용영역 > 세대·제조사
- **PDB는 자발적 축출만 막는다** — Spot 회수 자체를 지연시키지 못한다
- ⚠️ `minAvailable`을 레플리카 수와 같게 잡으면 **드레인이 영원히 멈춘다**
- `topologySpreadConstraints`로 파드를 노드·AZ에 흩는다
- Spot에서는 `whenUnsatisfiable: ScheduleAnyway`가 안전하다
- 관리형 노드 그룹·Karpenter는 **중단 처리가 기본 제공**된다
- Karpenter는 **대체 노드를 먼저 띄우고** 옮기므로 공백이 짧다
- 혼합은 **taint/toleration으로 옵트인** — 기본 온디맨드, 감당 가능한 것만 Spot
- ⭐ **"Spot을 켰다"와 "중단을 견딘다"는 다르다** — `kubectl drain`으로 실제 검증한다
- "Spot 용량 부족"의 대부분은 **다양화 설계의 문제**
