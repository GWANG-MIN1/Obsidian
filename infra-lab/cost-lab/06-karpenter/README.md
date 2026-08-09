# 06 Karpenter

노드 그룹 방식은 **인스턴스 타입을 사람이 미리 정한다.** t3.medium 2대로 고정해두면, 메모리만 필요한 파드가 와도 t3.medium이 뜬다.  
Karpenter는 **Pending 파드의 요구사항을 보고 그때그때 적합한 인스턴스를 고른다.** 그리고 필요 없어지면 통합·정리한다.

---

## Cluster Autoscaler와의 차이

```
[ Cluster Autoscaler ]
파드 Pending → 어느 노드그룹을 키울지 결정 → ASG desired 증가 → 노드 생성
                    │
              노드그룹이 미리 정의돼 있어야 한다 (타입 고정)

[ Karpenter ]
파드 Pending → 요구사항(cpu·mem·아키텍처·zone) 분석 → 적합한 인스턴스 직접 선택 → 생성
                    │
              노드그룹 개념이 없다. EC2 를 직접 만든다.
```

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| 노드 그룹 | **필요** (타입별로 미리 정의) | 불필요 |
| 인스턴스 선택 | 정의된 그룹 중에서 | **요구사항에 맞춰 자동** |
| 프로비저닝 속도 | 느림 (ASG 경유) | **빠름 (EC2 직접)** |
| bin-packing | 그룹 단위 | **파드 단위 최적화** |
| 통합(consolidation) | 제한적 | **적극적** |
| Spot 다양화 | 그룹 설정에 의존 | **자동** |
| 성숙도 | 오래됨, 안정 | 상대적으로 새것 |

> **비용 관점에서 Karpenter의 핵심 이점은 consolidation이다.** 파드가 줄면 노드를 통합해 더 작은 인스턴스로 바꾼다 — CA는 노드를 지우기만 하고 크기를 바꾸진 않는다.

---

## NodePool과 EC2NodeClass

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default

      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]      # Spot 우선 시도
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["t", "m", "c", "r"]       # 넓게 열어야 다양화가 된다
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]                      # 5세대 이상
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["ap-northeast-2a", "ap-northeast-2c"]

      expireAfter: 720h                      # 노드 최대 수명 30일 (패치 강제)

  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
    budgets:
      - nodes: "10%"                         # 한 번에 최대 10% 만 교체

  limits:
    cpu: "100"                               # ⚠️ 폭주 방지 상한
    memory: 400Gi
```

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  amiSelectorTerms:
    - alias: al2023@latest
  role: KarpenterNodeRole-my-cluster
  subnetSelectorTerms:
    - tags: { "karpenter.sh/discovery": "my-cluster" }
  securityGroupSelectorTerms:
    - tags: { "karpenter.sh/discovery": "my-cluster" }

  metadataOptions:
    httpEndpoint: enabled
    httpTokens: required                     # IMDSv2 강제
    httpPutResponseHopLimit: 1               # 파드에서 IMDS 접근 차단

  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 50Gi
        volumeType: gp3                      # gp2 보다 싸고 빠르다
        encrypted: true
        deleteOnTermination: true
```

> ⚠️ **`limits`를 반드시 설정한다.** 설정을 안 하면 잘못된 워크로드(무한 재시도하는 Job 등)가 노드를 무한정 띄워 청구서를 폭발시킨다. **Karpenter 도입 사고의 1순위다.**  
> `metadataOptions`의 IMDSv2·hop limit 1은 보안 하드닝의 핵심 항목이기도 하다. → `../security-lab/09-cluster-hardening/`

### requirements를 넓게 여는 것이 중요하다

```
❌ values: ["t3.medium"]                    선택지 1개 → Spot 다양화 불가
✅ instance-category: ["t","m","c","r"]     수십 개 타입 중 자동 선택
```

> **Karpenter의 가치는 선택지가 많을 때 나온다.** 좁게 제한하면 노드 그룹과 다를 게 없어진다.  
> 제한은 **정말 필요한 조건**(아키텍처, AZ, GPU 유무)만 건다. → `05-spot-instances/`

---

## Consolidation — 비용 절감의 핵심

```
[ 시간 t1 ]  파드 10개
  m5.large × 3  (사용률 70%)

[ 시간 t2 ]  파드 4개로 감소
  m5.large × 3  (사용률 25%)  ← CA 라면 빈 노드만 지우고 끝
      │
      ▼ Karpenter consolidation
  m5.large × 1  (사용률 75%)  ← 통합하고 남은 노드 제거
      또는
  t3.medium × 1               ← 더 작은 타입으로 교체
```

| 정책 | 동작 |
|---|---|
| `WhenEmpty` | **완전히 빈 노드만** 제거 (보수적) |
| `WhenEmptyOrUnderutilized` | 빈 노드 + **저사용 노드 통합·교체** (권장) |

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m
  budgets:
    - nodes: "10%"                  # 동시 교체 상한
    - nodes: "0"                    # 업무 시간에는 교체 금지
      schedule: "0 9 * * mon-fri"
      duration: 9h
```

> **consolidation은 파드를 옮긴다.** 즉 **정기적으로 파드가 재스케줄된다** — Spot 중단을 견디는 설계가 여기서도 그대로 필요하다.  
> `budgets`로 동시 교체 수를 제한하고, 민감한 시간대에는 아예 금지할 수 있다. **PDB도 함께 있어야 안전하다.** → `05-spot-instances/`

### disruption을 막고 싶은 파드

```yaml
metadata:
  annotations:
    karpenter.sh/do-not-disrupt: "true"      # 이 파드가 있는 노드는 통합 대상에서 제외
```

> 마이그레이션 Job처럼 **중단되면 안 되는 작업**에 붙인다. 남발하면 consolidation이 무력화된다.

---

## 노드 수명 제한

```yaml
spec:
  template:
    spec:
      expireAfter: 720h        # 30일 뒤 노드를 교체한다
```

> **노드 교체가 곧 패치다.** 수명을 제한하면 최신 AMI가 자연스럽게 적용되고, 침해가 지속되지 못한다.  
> 보안 하드닝의 "노드를 패치하지 말고 교체하라"와 같은 발상이다. → `../security-lab/09-cluster-hardening/`

---

## 비용 관점 설정 요약

| 설정 | 효과 |
|---|---|
| `capacity-type: ["spot", "on-demand"]` | Spot 우선, 없으면 온디맨드 |
| `instance-category` 넓게 | 다양화 → Spot 안정성·가격 최적 |
| `consolidationPolicy: WhenEmptyOrUnderutilized` | **유휴 자원 자동 제거** |
| `volumeType: gp3` | gp2 대비 저렴 |
| `arch: ["amd64","arm64"]` | Graviton 자동 활용 → `04-compute-purchasing/` |
| **`limits`** | 폭주 방지 (필수) |

```yaml
# Graviton 을 섞으려면 멀티아키 이미지가 전제
requirements:
  - key: kubernetes.io/arch
    operator: In
    values: ["amd64", "arm64"]
```

> ⚠️ **멀티아키 이미지가 없는데 arm64를 열면 파드가 `CrashLoopBackOff`로 죽는다.** 이미지 파이프라인이 먼저다. → `../cicd-lab/04-container-image-pipeline/`

---

## 운영 확인

```bash
kubectl get nodepool
kubectl get ec2nodeclass
kubectl get nodeclaim                      # Karpenter 가 만든 노드 요청
kubectl describe nodeclaim <NAME>

kubectl get nodes -L karpenter.sh/nodepool,node.kubernetes.io/instance-type,karpenter.sh/capacity-type

kubectl -n karpenter logs deploy/karpenter -f
kubectl -n karpenter logs deploy/karpenter | grep -i "consolidat\|disrupt\|launched"
```

```promql
# 노드 유휴율 — consolidation 이 잘 도는지의 지표
1 - (sum(kube_pod_container_resource_requests{resource="cpu"})
     / sum(kube_node_status_allocatable{resource="cpu"}))

# Spot 비율
count(kube_node_labels{label_karpenter_sh_capacity_type="spot"}) / count(kube_node_info)
```

> **유휴율이 consolidation의 성적표다.** 도입 전후를 비교하면 효과가 숫자로 보인다. → `03-kubernetes-cost-allocation/`

---

## 도입 시 주의

| 항목 | 주의 |
|---|---|
| **`limits` 미설정** | **청구서 폭발** — 가장 흔한 사고 |
| requirements 과도 제한 | 노드 그룹과 다를 게 없어진다 |
| PDB 없음 | consolidation 때 서비스 중단 |
| 서브넷·SG 태그 누락 | 노드가 안 뜬다 (`karpenter.sh/discovery`) |
| 노드 그룹과 공존 | 관리형 노드그룹 1개는 남겨 Karpenter 자신을 띄운다 |
| 멀티아키 미준비 | arm64를 열면 파드가 죽는다 |

```
Karpenter 자신은 어디서 도는가?
  → Karpenter 가 만든 노드에서 돌면 자기 자신을 지울 수 있다
  → 작은 관리형 노드 그룹(또는 Fargate)에 Karpenter·시스템 컴포넌트를 둔다
```

> **닭과 달걀 문제다.** Terraform 백엔드 부트스트랩과 같은 구조 — 최소한의 것은 다른 방식으로 먼저 세운다. → `../terraform-lab/09-aws-vpc-eks/`

---

## 언제 도입할 것인가

```
노드가 몇 대뿐이고 부하가 일정하다      →  관리형 노드 그룹으로 충분
부하 변동이 크다 / 워크로드가 다양하다   →  ✅ Karpenter
GPU·ARM·큰 메모리 등 요구가 섞인다      →  ✅ Karpenter
클러스터가 크고 유휴율이 높다            →  ✅ Karpenter (consolidation)
```

> **t3.medium 2대짜리 dev 클러스터에는 Karpenter가 과하다.** 관리 대상이 하나 늘 뿐 절감할 유휴가 별로 없다.  
> 이 프로젝트가 관리형 노드 그룹 + SPOT을 쓰는 것도 그 이유다 — **규모에 맞는 도구를 고르는 게 먼저**다. → `../observability-lab/06-logging/`(Loki 배포 모드 선택과 같은 판단)

---

## 배운 점

- 노드 그룹은 **타입을 사람이 미리 정하고**, Karpenter는 **파드 요구사항을 보고 고른다**
- 비용 관점의 핵심 이점은 **consolidation** — CA는 노드를 지우기만 하고 크기를 못 바꾼다
- ⚠️ **`limits`를 반드시 설정한다** — 미설정이 Karpenter 도입 사고 1순위(청구서 폭발)
- **requirements를 넓게 열어야** Karpenter의 가치가 나온다 (좁히면 노드 그룹과 동일)
- `WhenEmptyOrUnderutilized`가 유휴 제거의 핵심 설정
- **consolidation은 파드를 옮긴다** — Spot 중단을 견디는 설계가 그대로 필요
- `budgets`로 동시 교체 수·시간대를 제한하고 **PDB를 함께** 둔다
- `karpenter.sh/do-not-disrupt`로 중단되면 안 되는 파드를 보호 (남발 금지)
- **`expireAfter`로 노드 수명을 제한하면 교체가 곧 패치**가 된다
- `metadataOptions`의 IMDSv2 + hop limit 1은 보안 필수 항목
- ⚠️ **멀티아키 이미지 없이 arm64를 열면 파드가 죽는다** — 이미지 파이프라인이 먼저
- 서브넷·SG에 **`karpenter.sh/discovery` 태그**가 없으면 노드가 안 뜬다
- **Karpenter 자신은 별도 노드에서 돌아야 한다** (닭과 달걀)
- **유휴율이 consolidation의 성적표** — 도입 전후 비교로 효과를 증명한다
- **작은 클러스터에는 Karpenter가 과하다** — 규모에 맞는 도구를 고른다
