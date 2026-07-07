# 01 클러스터 기초

Kubernetes(K8s)는 여러 호스트에 걸쳐 컨테이너를 배포·확장·복구하는 오케스트레이션 플랫폼이다.  
Docker Swarm에서 익힌 **선언형·서비스·복제·셀프힐링** 개념이 그대로 이어지되, 훨씬 강력하고 확장 가능하다.

---

## 클러스터 아키텍처

Kubernetes 클러스터는 **Control Plane(두뇌)** 과 **Node(일꾼)** 로 나뉜다.

```
┌────────────────────────────────────────────────────────────┐
│                     Control Plane                           │
│  ┌───────────┐ ┌──────────────┐ ┌──────────┐ ┌───────────┐ │
│  │ API Server│ │  Scheduler   │ │   etcd   │ │ Controller│ │
│  │  (관문)   │ │ (배치 결정)  │ │ (상태DB) │ │  Manager  │ │
│  └───────────┘ └──────────────┘ └──────────┘ └───────────┘ │
└──────────────────────────┬─────────────────────────────────┘
                           │ (kubelet ↔ API Server 통신)
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ Node 1  │        │ Node 2  │        │ Node 3  │
   │ kubelet │        │ kubelet │        │ kubelet │
   │ kube-   │        │ kube-   │        │ kube-   │
   │ proxy   │        │ proxy   │        │ proxy   │
   │ [Pods]  │        │ [Pods]  │        │ [Pods]  │
   └─────────┘        └─────────┘        └─────────┘
```

### Control Plane 구성요소

| 구성요소 | 역할 |
|----------|------|
| **kube-apiserver** | 모든 요청의 관문. `kubectl`·컨트롤러·kubelet이 여기로 통신. REST API 제공 |
| **etcd** | 클러스터의 모든 상태를 저장하는 분산 key-value DB. **유일한 진실의 원천(SSOT)** |
| **kube-scheduler** | 새 Pod를 **어느 Node에 배치**할지 결정 (리소스·제약 조건 고려) |
| **kube-controller-manager** | Deployment·ReplicaSet 등 컨트롤러 루프 실행. "원하는 상태"로 계속 조정 |
| **cloud-controller-manager** | 클라우드(로드밸런서·볼륨 등)와 연동 (관리형 클러스터) |

### Node 구성요소

| 구성요소 | 역할 |
|----------|------|
| **kubelet** | 각 Node의 에이전트. API Server 지시를 받아 Pod(컨테이너)를 실제로 띄우고 상태 보고 |
| **kube-proxy** | Service의 네트워크 규칙(iptables/IPVS) 관리, 트래픽 라우팅 |
| **컨테이너 런타임** | 실제 컨테이너 실행 (containerd, CRI-O 등) |

> 핵심: **kubectl → API Server → etcd에 원하는 상태 기록 → 컨트롤러가 현재 상태를 원하는 상태로 조정 → kubelet이 Node에서 실행.** 이 조정 루프(reconciliation loop)가 Kubernetes의 심장이다.

---

## 선언형(Declarative) 모델

Kubernetes의 가장 중요한 사고방식. **"어떻게(how)"가 아니라 "무엇을(what)"을 선언**한다.

```
명령형 (Imperative)          선언형 (Declarative)
"컨테이너 3개 실행해"    vs   "컨테이너는 항상 3개여야 해"
→ 하나 죽으면 수동 재실행     → 하나 죽으면 시스템이 자동으로 다시 3개로
```

매니페스트(YAML)에 원하는 상태를 기록하고 `kubectl apply`로 제출하면, Kubernetes가 그 상태를 **지속적으로 유지**한다. 이것이 셀프힐링·오토스케일링의 기반이다.

---

## 로컬 클러스터 구축

학습·개발용으로 노트북에 단일/소규모 클러스터를 띄운다.

| 도구 | 특징 |
|------|------|
| **minikube** | 가장 대중적. VM/Docker 드라이버, 애드온(ingress, dashboard) 풍부 |
| **kind** | Kubernetes IN Docker. 노드를 컨테이너로 실행, CI에 적합·빠름 |
| **Docker Desktop** | 내장 Kubernetes 토글로 즉시 사용 |
| **k3s** | 경량 배포판. IoT/엣지·저사양 환경 |

```bash
# minikube
minikube start --driver=docker
minikube status
kubectl get nodes            # Ready 상태 확인

# kind
kind create cluster --name lab
kubectl cluster-info --context kind-lab
```

---

## kubectl 기본

`kubectl`은 API Server와 통신하는 CLI다. 대부분의 명령은 `kubectl 동사 리소스 이름` 형태.

```bash
kubectl get nodes                    # 노드 목록
kubectl get pods -A                  # 전체 네임스페이스 파드
kubectl describe node <노드명>       # 노드 상세 (용량·조건·파드)
kubectl cluster-info                 # 클러스터 엔드포인트
kubectl api-resources                # 사용 가능한 리소스 종류 목록
kubectl explain pod.spec             # 필드 스펙 설명 (내장 문서)
```

### 명령형 vs 선언형

```bash
# 명령형: 빠른 실습·일회성
kubectl run nginx --image=nginx
kubectl create deployment web --image=nginx --replicas=3

# 선언형: 실무 표준 (Git으로 버전 관리)
kubectl apply -f deployment.yaml
```

> 실습은 명령형으로 빠르게 시작하되, **실무·GitOps는 반드시 선언형(YAML + apply)** 을 쓴다. `--dry-run=client -o yaml`로 명령형에서 YAML 뼈대를 뽑아 학습하면 좋다.

---

## kubeconfig

`kubectl`이 **어느 클러스터에, 어떤 사용자로** 접속할지는 `~/.kube/config`에 담긴다.

```
context = cluster(어디로) + user(누구로) + namespace(기본 작업 공간)
```

```bash
kubectl config get-contexts                       # 컨텍스트 목록
kubectl config use-context <이름>                 # 클러스터 전환
kubectl config set-context --current --namespace=dev  # 기본 ns 변경
```

---

## 배운 점

- Kubernetes = **Control Plane(결정)** + **Node(실행)**, etcd가 상태의 유일한 원천
- 조정 루프: 원하는 상태(선언) ↔ 현재 상태를 끊임없이 맞춘다 → 셀프힐링의 원리
- 실습은 minikube/kind로 시작, `kubectl get/describe`가 관찰의 기본
- 명령형은 빠른 실습용, **선언형(apply)이 실무 표준**
- 다음 장부터는 가장 작은 실행 단위인 **Pod**부터 쌓아 올린다
