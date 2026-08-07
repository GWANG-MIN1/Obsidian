# 05 RBAC 최소권한

RBAC은 **"누가(Subject) 무엇을(Resource) 어떻게(Verb) 할 수 있는가"** 를 정한다.  
개념은 단순한데 실무에서 거의 항상 과다 권한으로 끝난다 — **"일단 되게 하자"가 `cluster-admin`으로 귀결되기 때문**이다.

---

## 4가지 오브젝트

```
    권한 정의                    주체에 연결
┌──────────────┐            ┌──────────────────┐
│     Role     │◀───────────│   RoleBinding    │──▶ User / Group / ServiceAccount
│ (네임스페이스) │            │   (네임스페이스)   │
└──────────────┘            └──────────────────┘

┌──────────────┐            ┌──────────────────┐
│ ClusterRole  │◀───────────│ClusterRoleBinding│──▶ 클러스터 전체
│  (클러스터)   │            │    (클러스터)     │
└──────────────┘            └──────────────────┘
```

| 조합 | 범위 |
|---|---|
| Role + RoleBinding | 한 네임스페이스 |
| **ClusterRole + RoleBinding** | **정의는 재사용, 적용은 한 네임스페이스** ← 유용 |
| ClusterRole + ClusterRoleBinding | 클러스터 전체 |
| Role + ClusterRoleBinding | ❌ 불가능 |

> **ClusterRole + RoleBinding 조합이 실무에서 가장 유용하다.** 권한 정의를 한 번 쓰고 네임스페이스마다 바인딩만 만든다. 팀마다 Role을 복사할 필요가 없다.

---

## Role 작성

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: myapp
rules:
  - apiGroups: [""]                          # "" = core API 그룹
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]

  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "update", "patch"]

  - apiGroups: [""]
    resources: ["pods/exec"]                 # 서브리소스는 별도로 명시
    verbs: ["create"]
```

| verb | 의미 |
|---|---|
| `get` / `list` / `watch` | 조회 (list는 **전체 목록**을 의미 — get보다 강하다) |
| `create` / `update` / `patch` | 생성·수정 |
| `delete` / `deletecollection` | 삭제 |
| `*` | 전부 (**쓰지 않는다**) |

> **`apiGroups: [""]`가 core 그룹(Pod, Service, ConfigMap, Secret)이다.** Deployment는 `apps`, Ingress는 `networking.k8s.io`에 있다. 여기서 자주 틀린다.  
> `pods/log`, `pods/exec`, `pods/portforward` 같은 **서브리소스는 따로 써야 한다.** `pods`에 권한이 있어도 exec은 안 된다.

---

## 위험한 권한들

겉보기엔 평범한데 실질적으로 클러스터 장악으로 이어지는 것들이다.

| 권한 | 왜 위험한가 |
|---|---|
| `secrets: get/list` | **모든 시크릿 조회** — DB 비밀번호, 토큰 |
| `pods/exec`, `pods/attach` | 실행 중인 컨테이너 진입 → 그 파드의 권한 획득 |
| **`escalate`** on roles | 자기 권한을 넘는 Role을 만들 수 있다 |
| **`bind`** on roles | 자신에게 `cluster-admin`을 바인딩할 수 있다 |
| **`impersonate`** | 다른 사용자·SA로 행세 |
| `create pods` | **원하는 SA로 파드를 띄워** 그 권한을 얻는다 |
| `nodes/proxy` | kubelet API 직접 호출 |
| `create` on `serviceaccounts/token` | 임의 SA의 토큰 발급 |

```
"파드 생성 권한"의 실제 의미
  → 임의의 ServiceAccount 를 지정한 파드를 띄울 수 있다
  → 그 SA 가 cluster-admin 이면 → 클러스터 장악
  → hostPath: / 를 마운트하면 → 노드 장악
```

> **`create pods`는 사실상 그 네임스페이스의 모든 SA 권한을 갖는 것과 같다.** RBAC만으로는 이걸 막을 수 없고, **admission 정책(Kyverno·PSA)이 함께 있어야** 한다. → `04-kyverno/`  
> `escalate`와 `bind`는 RBAC의 자기 권한 상승 방지 장치를 무력화한다. 기본적으로 아무에게도 주지 않는다.

---

## ServiceAccount

파드가 API 서버에 접근할 때 쓰는 신원이다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp
automountServiceAccountToken: false      # 기본값 true — 필요 없으면 끈다
```

```yaml
# 파드에서 사용
spec:
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false    # 파드 수준에서도 끌 수 있다
```

### ⚠️ default ServiceAccount

```
파드에 serviceAccountName 을 안 쓰면 → 네임스페이스의 default SA 가 붙는다
  → 기본적으로 토큰이 /var/run/secrets/... 에 마운트된다
  → 파드가 뚫리면 그 토큰이 공격자 손에 들어간다
```

```bash
# 모든 네임스페이스의 default SA 토큰 자동 마운트 끄기
kubectl get ns -o name | while read ns; do
  kubectl patch sa default -n "${ns#namespace/}" \
    -p '{"automountServiceAccountToken": false}' 2>/dev/null
done
```

> **API 서버를 쓰지 않는 앱이 대부분이다.** 그런데 기본값 때문에 전부 토큰을 들고 있다.  
> `automountServiceAccountToken: false`는 비용 0에 가까운 방어인데 거의 아무도 안 한다. → `02-pod-container-security/`

### IRSA — AWS 권한 연결

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets
  namespace: external-secrets
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/dev-external-secrets
```

```
파드 ─(SA 토큰)─▶ EKS OIDC ─▶ STS AssumeRoleWithWebIdentity ─▶ 임시 IAM 자격증명
                                        │
                          신뢰 정책의 sub 조건으로 검증
                          "system:serviceaccount:external-secrets:external-secrets"
```

> **`sub` 조건을 빼면 그 클러스터의 아무 파드나 그 역할을 맡을 수 있다.** RBAC 최소권한과 IAM 최소권한이 SA에서 만나는 지점이다. → `../terraform-lab/09-aws-vpc-eks/` `06-secrets-management/`

> 🔧 **IRSA 어노테이션을 붙인 뒤에는 `rollout restart`가 필수다.** 자격증명은 파드가 뜰 때 주입되므로, 이미 떠 있는 파드는 어노테이션을 붙여도 역할을 못 맡는다. 실제로 여기서 막힌다.

```bash
kubectl -n external-secrets annotate sa external-secrets \
  eks.amazonaws.com/role-arn="$(terraform output -raw external_secrets_irsa_role_arn)"
kubectl -n external-secrets rollout restart deploy/external-secrets    # ← 필수
```

---

## 권한 감사

```bash
# 내가 할 수 있는 것 전부
kubectl auth can-i --list

# 특정 동작
kubectl auth can-i delete secrets --all-namespaces
kubectl auth can-i '*' '*'                         # 클러스터 관리자인가?

# 다른 주체 흉내 (가장 유용)
kubectl auth can-i --list --as system:serviceaccount:myapp:default -n myapp
kubectl auth can-i create pods --as system:serviceaccount:myapp:myapp-sa -n myapp

kubectl auth whoami                                # 현재 신원 (1.26+)
```

```bash
# cluster-admin 을 가진 주체 전부
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] | select(.roleRef.name=="cluster-admin")
           | {binding:.metadata.name, subjects:.subjects}'

# secrets 를 읽을 수 있는 ClusterRole 찾기
kubectl get clusterroles -o json \
  | jq -r '.items[] | select(.rules[]? | select((.resources[]? == "secrets")
           and (.verbs[]? | test("get|list|\\*")))) | .metadata.name'

# 와일드카드를 쓰는 Role 찾기
kubectl get clusterroles,roles -A -o json \
  | jq -r '.items[] | select(.rules[]? | (.verbs[]? == "*") or (.resources[]? == "*"))
           | "\(.kind)/\(.metadata.name)"'
```

> **`--as`로 흉내내는 것이 RBAC 디버깅의 핵심 도구다.** "이 SA가 정말 이것만 할 수 있는가"를 직접 확인할 수 있다.  
> 도구를 쓴다면 `rbac-tool`, `kubectl-who-can`, `rakkess`가 감사를 편하게 해준다.

---

## 최소권한 설계 순서

```
1. 무엇이 필요한지 모른다        →  일단 아무 권한 없이 배포한다
2. Forbidden 에러가 난다        →  로그에 어떤 리소스·verb 인지 찍힌다
3. 그것만 추가한다              →  반복
4. 안정화되면 감사              →  안 쓰는 권한 제거
```

```
Error from server (Forbidden): pods is forbidden:
  User "system:serviceaccount:myapp:myapp-sa" cannot list resource "pods"
  in API group "" in the namespace "myapp"
        │                    │                │
      verb                리소스           apiGroup
```

> **에러 메시지가 정확히 필요한 권한을 알려준다.** 넓게 주고 좁히는 것보다 이 방식이 빠르고 정확하다.  
> "일단 `cluster-admin` 주고 나중에 좁히자"는 영원히 안 좁혀진다. → `01-security-basics/`

### 기본 제공 ClusterRole

| 이름 | 권한 |
|---|---|
| `cluster-admin` | 전부 (**사람에게 상시 부여하지 않는다**) |
| `admin` | 네임스페이스 내 전부 (쿼터·네임스페이스 제외) |
| `edit` | 리소스 수정 가능, RBAC은 불가 |
| `view` | 읽기 전용 (**단, Secret은 제외**) |

```yaml
# 대부분의 개발자에게 적절한 수준
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-edit
  namespace: myapp
subjects:
  - kind: Group
    name: dev-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit                # ClusterRole 을 RoleBinding 으로 → 이 네임스페이스만
  apiGroup: rbac.authorization.k8s.io
```

> **`view`는 Secret을 못 읽는다.** 이걸 모르고 "읽기 권한인데 왜 안 보이지"로 헤매는 경우가 많다. 의도된 설계다.

---

## EKS 접근 제어

```
예전: aws-auth ConfigMap 을 편집해 IAM 주체를 K8s 그룹에 매핑
      → Terraform 이 Kubernetes provider 를 필요로 해서 순환 의존 발생

지금: EKS Access Entry (API 모드)
      → AWS API 로 처리, Terraform 이 AWS 자격증명만으로 완결
```

```hcl
enable_cluster_creator_admin_permissions = true   # apply 주체에게 cluster-admin
```

> 데모·개인 클러스터에서는 편하지만, **팀 환경에서는 access entry로 역할별 권한을 나눈다.** → `../terraform-lab/09-aws-vpc-eks/`

---

## 배운 점

- **Role+RoleBinding(네임스페이스)** / **ClusterRole+ClusterRoleBinding(전체)**
- **ClusterRole + RoleBinding 조합**이 가장 유용하다 (정의 재사용, 적용은 한 네임스페이스)
- `apiGroups: [""]`가 core 그룹 — Deployment는 `apps`에 있다
- **서브리소스(`pods/log`, `pods/exec`)는 따로 명시**해야 한다
- **`create pods`는 사실상 그 네임스페이스의 모든 SA 권한**을 얻는 것 — admission 정책이 함께 필요
- `escalate`·`bind`·`impersonate`는 자기 권한 상승 경로 — 아무에게도 주지 않는다
- `list`는 전체 목록을 의미해 `get`보다 강하다
- **파드에 SA를 안 쓰면 default SA 토큰이 자동 마운트**된다
- **`automountServiceAccountToken: false`** 는 비용 0에 가까운 방어인데 대부분 안 한다
- IRSA의 `sub` 조건을 빼면 **클러스터의 아무 파드나** 그 역할을 맡는다
- 🔧 **IRSA 어노테이션 후 `rollout restart`가 필수** — 자격증명은 파드 기동 시 주입된다
- **`kubectl auth can-i --as`가 RBAC 디버깅의 핵심 도구**
- 설계는 **권한 0에서 시작해 Forbidden 에러를 보고 추가**하는 게 빠르고 정확하다
- 에러 메시지가 필요한 **verb·리소스·apiGroup을 정확히** 알려준다
- 기본 ClusterRole `view`는 **Secret을 못 읽는다** (의도된 설계)
- EKS는 `aws-auth` ConfigMap 대신 **access entry(API 모드)** 를 쓴다
