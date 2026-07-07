# 08 Namespace & RBAC

하나의 클러스터를 여러 팀·환경이 공유하려면 **격리**와 **권한 통제**가 필요하다.  
**Namespace**로 리소스를 논리적으로 나누고, **RBAC**로 "누가 무엇을 할 수 있는지"를 통제한다.

---

## Namespace (논리적 격리)

클러스터를 여러 가상 공간으로 나눈다. 팀·환경·프로젝트 단위 분리에 사용.

```bash
kubectl get namespaces
kubectl create namespace dev
kubectl get pods -n dev                     # 특정 ns 조회
kubectl config set-context --current --namespace=dev   # 기본 ns 변경
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### 기본 네임스페이스

| Namespace | 용도 |
|-----------|------|
| `default` | 지정 안 하면 여기 |
| `kube-system` | 클러스터 핵심 컴포넌트 (CoreDNS, kube-proxy 등) |
| `kube-public` | 모두가 읽을 수 있는 공용 정보 |
| `kube-node-lease` | 노드 하트비트(lease) |

### 격리의 범위

| 격리됨 (네임스페이스별) | 격리 안 됨 (클러스터 전역) |
|------------------------|---------------------------|
| Pod, Deployment, Service, ConfigMap, Secret, PVC | Node, PersistentVolume, StorageClass |
| Role, RoleBinding, ResourceQuota | ClusterRole, ClusterRoleBinding, Namespace 자체 |

> Namespace는 **논리적 분리이지 강한 보안 경계가 아니다.** 네트워크 격리는 NetworkPolicy, 강한 격리는 별도 클러스터가 필요하다. DNS는 네임스페이스를 넘어 접근 가능(`svc.다른ns`).

---

## RBAC (역할 기반 접근 제어)

"누가(주체) + 무엇을(리소스) + 어떻게(동사)" 할 수 있는지를 정의한다.

```
주체(Subject)      역할(Role)              바인딩(Binding)
User / Group   ──▶ verbs + resources  ◀── 주체와 역할을 연결
ServiceAccount     (권한의 집합)
```

### 4가지 오브젝트

| 오브젝트 | 범위 | 역할 |
|----------|------|------|
| **Role** | 네임스페이스 | 해당 ns 안의 권한 정의 |
| **ClusterRole** | 클러스터 전역 | 전역 리소스·모든 ns 권한 정의 |
| **RoleBinding** | 네임스페이스 | 주체 ↔ Role 연결 |
| **ClusterRoleBinding** | 클러스터 전역 | 주체 ↔ ClusterRole 연결 |

### Role — 권한 묶음 정의

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
  - apiGroups: [""]                 # "" = core API 그룹
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"] # 읽기 전용
```

동사(verb): `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `*`

### RoleBinding — 주체에 역할 부여

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: dev
  name: read-pods
subjects:
  - kind: ServiceAccount
    name: ci-bot
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# 권한 확인 (매우 유용)
kubectl auth can-i create pods
kubectl auth can-i list secrets --as=system:serviceaccount:dev:ci-bot -n dev
```

---

## ServiceAccount (Pod의 신원)

사람은 User로 인증하지만, **Pod 안 앱은 ServiceAccount로** API Server에 인증한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-bot
  namespace: dev
---
apiVersion: v1
kind: Pod
metadata:
  name: deployer
spec:
  serviceAccountName: ci-bot     # 이 SA의 권한으로 API 접근
  containers:
    - name: kubectl
      image: bitnami/kubectl
```

> Pod가 명시 안 하면 `default` ServiceAccount가 붙는다. **CI/CD·오퍼레이터처럼 API를 호출하는 Pod에는 최소 권한 전용 SA를 만들어 부여**한다. `default`에 과한 권한을 주지 말 것.

---

## ResourceQuota & LimitRange

네임스페이스별로 자원 사용 총량과 개별 상한을 통제한다.

### ResourceQuota — 네임스페이스 총량

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    persistentvolumeclaims: "10"
```

### LimitRange — Pod/컨테이너 개별 기본값·상한

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:              # limits 미지정 시 기본
        cpu: 500m
        memory: 256Mi
      defaultRequest:       # requests 미지정 시 기본
        cpu: 100m
        memory: 128Mi
      max:                  # 컨테이너당 최대
        cpu: "2"
        memory: 2Gi
```

| 오브젝트 | 통제 대상 |
|----------|-----------|
| **ResourceQuota** | 네임스페이스 **전체 합계** (총 CPU·메모리·오브젝트 수) |
| **LimitRange** | **개별** Pod/컨테이너의 기본값·최소·최대 |

---

## 최소 권한 원칙 (Least Privilege)

```
❌ cluster-admin을 여기저기 부여 → 사고 시 전체 클러스터 위험
✅ 필요한 ns·리소스·동사만 정확히 부여, 전역 권한은 최소화
```

- 사람·앱마다 **필요한 만큼만** 권한 부여
- 읽기만 필요하면 `get/list/watch`만
- 전역(ClusterRole)은 꼭 필요할 때만, 대부분 네임스페이스 Role로 충분
- `default` ServiceAccount에 권한 부여 금지

---

## 배운 점

- **Namespace** = 논리적 분리(팀·환경), 단 강한 보안 경계는 아님
- 일부 리소스(Node·PV·StorageClass)는 네임스페이스에 속하지 않는 전역 자원
- **RBAC** = Role(권한 묶음) + Binding(주체 연결), ns용/전역용 2쌍
- Pod의 신원은 **ServiceAccount** → CI/오퍼레이터엔 최소 권한 전용 SA
- **ResourceQuota(총량)** + **LimitRange(개별)** 로 자원 남용 방지
- 관통 원칙: **최소 권한(Least Privilege)**
