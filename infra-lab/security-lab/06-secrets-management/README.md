# 06 시크릿 관리

GitOps의 전제는 **"Git이 진실의 원천"** 인데, 비밀번호는 Git에 넣을 수 없다. 이 모순이 GitOps의 유일한 진짜 난제다.  
해법의 공통 원리는 하나다 — **Git에는 참조만, 값은 밖에.**

---

## Kubernetes Secret의 한계

```
Secret 은 '암호화'가 아니다.
  data: 필드는 base64 인코딩일 뿐 → echo ... | base64 -d 하면 끝
```

| 오해 | 실제 |
|---|---|
| Secret은 암호화된다 | **base64 인코딩**일 뿐 |
| etcd에 안전하게 저장된다 | 기본은 평문 (EncryptionConfiguration 필요) |
| Secret은 아무나 못 본다 | RBAC에서 `secrets: get`이 있으면 전부 조회 가능 |
| ConfigMap보다 안전하다 | 저장 방식은 거의 같다 |

```bash
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
# supersecret123     ← 이게 전부다
```

> **Secret이 주는 것은 "암호화"가 아니라 "분리와 접근 제어의 단위"** 다. 실제 보호는 RBAC·etcd 암호화·감사 로그에서 온다.  
> EKS에서는 **KMS를 이용한 etcd 봉투 암호화**를 켤 수 있다. 클러스터 생성 시 설정한다.

---

## 방법 비교

| 방법 | Git에 들어가는 것 | 장점 | 단점 |
|---|---|---|---|
| **External Secrets (ESO)** | 참조(경로)만 | 값이 Git에 전혀 없다, 로테이션 자동 | 외부 저장소·IRSA 필요 |
| **Sealed Secrets** | 암호화된 값 | 외부 의존 없음 | **클러스터 키 분실 시 복호화 불가** |
| **SOPS + KMS/age** | 암호화된 값 | 파일 단위, Git 친화 | 복호화 플러그인 설정 필요 |
| ❌ 평문 Secret | 값 그대로 | — | 절대 금지 |

```
클러스터를 자주 재생성한다        →  ESO (외부에 값이 있으므로 무관)
외부 시크릿 저장소가 없다          →  Sealed Secrets / SOPS
이미 AWS 를 쓴다                  →  ESO + SSM / Secrets Manager
```

> **Sealed Secrets는 클러스터의 복호화 키가 사라지면 시크릿을 못 푼다.** 매일 클러스터를 지우는 환경에서는 키 백업·복원이 추가 부담이 된다.  
> 그 점에서 **값이 처음부터 클러스터 밖에 있는 ESO가 재생성이 잦은 환경에 맞는다.**

---

## External Secrets Operator

```
   Git                      클러스터                        AWS
┌──────────┐          ┌──────────────────┐          ┌──────────────┐
│External  │          │  ESO 컨트롤러     │──IRSA──▶ │ SSM Parameter│
│Secret    │─감시────▶│                  │◀─값───── │ Store        │
│(참조만)   │          │        ↓         │          └──────────────┘
└──────────┘          │  K8s Secret 생성  │
                      └──────────────────┘
```

### ① SecretStore — 어디서 가져올지

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore          # 네임스페이스 범위면 SecretStore
metadata:
  name: aws-ssm
spec:
  provider:
    aws:
      service: ParameterStore     # 또는 SecretsManager
      region: ap-northeast-2
      # auth 블록 없음 — 컨트롤러가 자기 파드의 IRSA 자격증명을 쓴다
```

> **`auth` 블록이 없는 게 정상이다.** IRSA를 쓰면 컨트롤러 파드가 이미 AWS 자격증명을 갖고 있다. 여기에 액세스 키를 넣으면 IRSA를 쓰는 의미가 없어진다.

```bash
kubectl get clustersecretstore aws-ssm
# STATUS 가 Valid / READY 가 True 여야 AWS 인증에 성공한 것
```

### ② ExternalSecret — 무엇을 어디로

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: grafana-admin
  namespace: observability
spec:
  refreshInterval: 1h                 # 주기적으로 다시 가져온다 (로테이션 반영)
  secretStoreRef:
    name: aws-ssm
    kind: ClusterSecretStore
  target:
    name: grafana-admin               # 생성될 K8s Secret 이름
    creationPolicy: Owner             # ESO 가 소유·관리 (삭제 시 함께 삭제)
  data:
    - secretKey: admin-user           # Secret 안의 키
      remoteRef:
        key: /eks-gitops/dev/grafana/admin-user
    - secretKey: admin-password
      remoteRef:
        key: /eks-gitops/dev/grafana/admin-password
```

| `creationPolicy` | 동작 |
|---|---|
| `Owner` | ESO가 Secret을 만들고 소유 (ExternalSecret 삭제 시 Secret도 삭제) |
| `Merge` | 기존 Secret에 키만 추가 |
| `Orphan` | 만들지만 소유하지 않는다 |

```bash
kubectl -n observability describe externalsecret grafana-admin
# STATUS: SecretSynced 여야 정상
kubectl -n observability get secret grafana-admin        # Opaque / DATA 2
```

### ③ SSM에 값 넣기 (한 번만)

```bash
aws ssm put-parameter --type SecureString \
  --name /eks-gitops/dev/grafana/admin-user --value admin
aws ssm put-parameter --type SecureString \
  --name /eks-gitops/dev/grafana/admin-password --value "$(openssl rand -base64 24)"
```

### ④ 소비하는 쪽

```yaml
# Grafana Helm values — 만들어진 Secret 을 참조
grafana:
  admin:
    existingSecret: grafana-admin
    userKey: admin-user
    passwordKey: admin-password
```

> **이 체인이 완성되면 Git 어디에도 비밀번호가 없다.** Git에는 `/eks-gitops/dev/grafana/admin-password`라는 **경로**만 있다.

---

## IRSA 체인

```
Terraform                          GitOps                    AWS
─────────                          ──────                    ───
IAM 역할 생성                    SA 어노테이션              SSM 파라미터
(SSM read, /eks-gitops/dev/*)    (역할 ARN)                (SecureString)
     │                                │                         │
     └────────── 역할 ARN 이 인계점 ───┘                         │
                        │                                       │
                   ESO 컨트롤러 ──AssumeRoleWithWebIdentity──────┘
```

```hcl
# Terraform — 신뢰 정책의 sub 조건이 핵심
condition {
  test     = "StringEquals"
  variable = "${module.eks.oidc_provider}:sub"
  values   = ["system:serviceaccount:external-secrets:external-secrets"]
}
```

```bash
kubectl -n external-secrets annotate sa external-secrets \
  eks.amazonaws.com/role-arn="$(terraform output -raw external_secrets_irsa_role_arn)"

kubectl -n external-secrets rollout restart deploy/external-secrets   # ← 필수
```

> 🔧 **어노테이션만 붙이고 끝내면 동작하지 않는다.** IRSA 자격증명은 **파드가 뜰 때 주입**되므로, 이미 떠 있는 파드는 어노테이션을 알아채지 못한다. `rollout restart`가 반드시 필요하다.  
> **IAM 역할은 Terraform(클라우드 상태), SA 어노테이션은 GitOps(클러스터 상태)** — 역할 ARN이 두 세계의 인계점이다. → `../terraform-lab/09-aws-vpc-eks/`

### IAM 정책은 경로로 좁힌다

```
❌ ssm:GetParameter on *                        모든 파라미터
✅ ssm:GetParameter on /eks-gitops/dev/*        이 환경만
```

> **경로 접두사로 스코프를 나누면 dev의 ESO가 prod 시크릿을 못 읽는다.** SSM 파라미터 이름을 `/프로젝트/환경/앱/키` 형태로 설계하는 이유다.

---

## 로테이션

```yaml
refreshInterval: 1h
```

```
SSM 값을 바꾼다
      ↓
최대 1시간 뒤 ESO 가 K8s Secret 을 갱신
      ↓
⚠️ 하지만 파드는 갱신을 모른다
```

| 주입 방식 | 값 변경 시 |
|---|---|
| `env` (환경변수) | **파드 재시작 전까지 옛 값** |
| 볼륨 마운트 | kubelet이 파일을 갱신 (수십 초~분) — **앱이 다시 읽어야 한다** |

> **Secret을 갱신해도 앱이 자동으로 새 값을 쓰지는 않는다.** 환경변수로 주입했다면 반드시 `rollout restart`가 필요하다.  
> `stakater/Reloader` 같은 컨트롤러가 Secret/ConfigMap 변경 시 자동으로 롤아웃을 걸어준다.

---

## Sealed Secrets와 SOPS

### Sealed Secrets

```bash
kubectl create secret generic db --from-literal=password=xxx --dry-run=client -o yaml \
  | kubeseal --format yaml > sealed-db.yaml     # 이걸 Git 에 커밋
```

```
클러스터의 컨트롤러가 개인키로 복호화 → 일반 Secret 생성
  ✅ 외부 저장소 불필요
  ❌ 클러스터 재생성 시 키가 사라지면 기존 SealedSecret 을 못 푼다
  ❌ 로테이션이 수동
```

### SOPS

```bash
sops --encrypt --kms arn:aws:kms:...:key/xxx secrets.yaml > secrets.enc.yaml
sops --decrypt secrets.enc.yaml
```

```
파일 단위 암호화 — 값만 암호화되고 키 이름은 평문 → diff 가 읽힌다
ArgoCD 에서 쓰려면 복호화 플러그인(ksops 등) 설정이 필요하다
```

| | ESO | Sealed Secrets | SOPS |
|---|---|---|---|
| Git에 암호문 | ❌ (참조만) | ✅ | ✅ |
| 외부 저장소 | 필요 | 불필요 | KMS(선택) |
| 클러스터 재생성 | 무관 | **키 백업 필수** | 무관 |
| 로테이션 | 자동(`refreshInterval`) | 수동 | 수동 |

---

## 절대 하지 말 것

```
❌ Git 에 평문 Secret 커밋              base64 는 암호화가 아니다
❌ Helm values 에 비밀번호               차트와 함께 Git 에 남는다
❌ 컨테이너 이미지에 시크릿 굽기          레이어에 영구히 남는다
❌ -e PASSWORD=xxx 로 넘기기            docker inspect·프로세스 목록에 노출
❌ 로그에 시크릿 출력                    로그 수집 시스템에 영구 저장
❌ terraform.tfvars 커밋                → ../terraform-lab/03-state/
```

> ⚠️ **한 번 Git에 커밋된 시크릿은 유출된 것으로 간주하고 즉시 로테이션한다.** 커밋을 되돌려도 히스토리와 포크·클론에 남는다.  
> `git filter-repo`로 히스토리를 지우는 것은 사후 정리일 뿐, **값 자체를 바꾸는 게 유일한 대응**이다.

```yaml
# CI 에서 시크릿 커밋 탐지
- uses: gitleaks/gitleaks-action@v2
# 또는
- run: trivy fs --scanners secret .
```

---

## 배운 점

- **Kubernetes Secret은 암호화가 아니라 base64 인코딩**이다
- 실제 보호는 **RBAC·etcd 암호화(KMS)·감사 로그**에서 온다
- 공통 원리는 하나 — **Git에는 참조만, 값은 밖에**
- **ESO**: 값이 처음부터 클러스터 밖 → 클러스터 재생성이 잦은 환경에 적합
- **Sealed Secrets**: 외부 의존이 없지만 **복호화 키 분실 시 복구 불가**
- SecretStore에 **`auth` 블록이 없는 게 정상** — IRSA로 컨트롤러 파드가 인증한다
- `ClusterSecretStore`의 STATUS가 **`Valid`** 여야 AWS 인증 성공
- ExternalSecret의 STATUS는 **`SecretSynced`** 가 정상
- 🔧 **IRSA 어노테이션 후 `rollout restart` 필수** — 자격증명은 파드 기동 시 주입
- IAM 역할=Terraform, SA 어노테이션=GitOps — **역할 ARN이 인계점**
- IAM 정책은 **SSM 경로 접두사로 환경을 분리**한다 (`/프로젝트/환경/앱/키`)
- ⚠️ **Secret이 갱신돼도 파드는 모른다** — env 주입은 `rollout restart` 필요
- 볼륨 마운트도 앱이 파일을 다시 읽어야 반영된다 (Reloader 같은 도구 활용)
- **한 번 커밋된 시크릿은 유출로 간주하고 즉시 로테이션** — 히스토리 삭제는 사후 정리일 뿐
- CI에 **gitleaks·`trivy fs --scanners secret`** 으로 커밋 탐지를 건다
