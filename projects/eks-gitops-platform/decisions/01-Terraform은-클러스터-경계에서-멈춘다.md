# 01. Terraform 은 클러스터 경계에서 멈추고 안쪽은 GitOps 에 넘긴다

> 메모: VPC · EKS · IAM 까지는 Terraform, 클러스터 안의 모든 것은 ArgoCD. Helm 차트를 Terraform 으로 배포하지 않는다.
> 2026-07-12 `docs/architecture.md`, 07-13 EKS 모듈 커밋 `fc70de5`

---

## 상황

EKS 위에 ArgoCD · 관측성 · 보안 스택을 올려야 했다. Terraform 하나로도 전부 올릴 수 있다 —
`helm` · `kubernetes` 프로바이더가 있으니 `terraform apply` 한 번에 클러스터부터 Grafana 까지 뜬다.
한 도구로 끝낼지, 경계를 둘지를 정해야 했다.

## 결정

**AWS API 로 만드는 것은 Terraform, Kubernetes API 로 만드는 것은 GitOps.**

| Terraform | GitOps (ArgoCD) |
|---|---|
| VPC · 서브넷 · NAT | Application 8개 (root + 자식 7) |
| EKS 컨트롤플레인 · 노드 그룹 · 애드온 4개 | kube-prometheus-stack · Loki · promtail |
| OIDC provider · ESO 용 IAM 역할 | Kyverno · 정책 4개 · External Secrets |
| state 백엔드 (bootstrap, 별도) | sample-app |

경계를 지키려고 두 가지를 같이 골랐다.

- **클러스터 인증은 access entry(API 모드)** — `enable_cluster_creator_admin_permissions = true`.
  `aws-auth` ConfigMap 을 안 쓰니 **Terraform 에 kubernetes 프로바이더가 아예 없다.** AWS 자격증명만으로 apply 가 끝난다.
- **ArgoCD 설치도 Terraform 이 아니다** — `kubectl apply -k gitops/bootstrap/argocd` 한 번. → [결정 04](04-app-of-apps-루트-하나와-멀티소스.md)

## 왜

- **롤백 단위가 달라서.** architecture.md 의 논리 — 섞으면 느린 클라우드 상태와 빠른 앱 상태가 엉켜 롤백이 괴롭다.
  `git revert` 가 곧 롤백이 되려면 클러스터 안의 진실은 Git 하나여야 한다.
- **드리프트를 보는 방식이 달라서.** Terraform 은 `apply` 할 때만 실제 상태를 본다. ArgoCD 는 3분마다 보고
  `selfHeal` 로 되돌린다. 클러스터 안은 사람이 `kubectl` 로 건드리기 쉬운 곳이라 후자가 맞다.
- **순환 의존이 없어서.** 클러스터를 만드는 같은 설정에서 그 클러스터에 접속하는 프로바이더를 두면
  "아직 없는 클러스터에 접속하는" 순서 문제가 생긴다. access entry 는 AWS API 라 그 고리가 없다.
  → [terraform-lab/09](../../../infra-lab/terraform-lab/09-aws-vpc-eks/README.md) 의 "인증 방식 — access entry"

## 왜 다른 건 안 썼나

**Terraform `helm_release` 로 스택까지**

한 번의 apply 로 끝나는 게 매력이다. 안 쓴 이유는 위의 셋 전부다 — 차트 업그레이드가 `terraform plan` 에 섞이고,
클러스터 안 드리프트를 되돌리지 못하고, helm · kubernetes 프로바이더를 클러스터 생성과 같은 설정에 묶어야 한다.

**`aws-auth` ConfigMap 으로 권한 부여**

예전 방식이다. ConfigMap 을 Terraform 으로 관리하려면 kubernetes 프로바이더가 필요하고 위의 순환이 돌아온다.

**ArgoCD 하나만 Terraform 으로 설치**

흔한 절충이다 — 스택은 GitOps 에 두되 GitOps 엔진은 IaC 로 띄운다. **안 쓴 이유는 저장소에 적혀 있지 않다.**
결과적으로 부트스트랩이 `kubectl` 두 번이 됐다.

## 이 결정의 대가

경계 자체는 끝까지 지켜졌다(`kubectl edit` 0회). 그런데 **경계를 건너는 것들이 전부 사람 손에 남았다.**

| 건너는 것 | 방향 | 실제로 누가 |
|---|---|---|
| API 엔드포인트 (kubeconfig) | Terraform → 내 PC | `aws eks update-kubeconfig` — 재생성마다 → [트러블 05](../troubleshooting/05-재생성하자-kubectl이-no-such-host.md) |
| ArgoCD 설치 · root-app | 사람 → 클러스터 | `kubectl apply` 2번 |
| ESO 역할 ARN | Terraform output → ServiceAccount | `kubectl annotate` + `rollout restart` → [결정 10](10-시크릿은-SSM에-Git에는-참조만.md) |
| 노드 수 | 변수 → 노드 그룹 | 생성 땐 `-var`, 이후엔 AWS CLI → [트러블 01](../troubleshooting/01-노드-두-대가-파드-34개로-만석.md) |

Terraform output 이 `configure_kubectl` 과 `external_secrets_sa_annotation` 을
**"복사해서 쓰라"는 출력값으로** 내놓는 게 그 증거다. 인계점을 코드로 만들지 않고 **안내문**으로 만들었다.

그리고 클러스터에 `eks-pod-identity-agent` 애드온을 **설치해 놓고 안 썼다.**
Pod Identity 는 "어느 네임스페이스의 어느 SA 가 어느 역할을 쓴다"는 연결을 **AWS API 객체**로 만들기 때문에,
이 결정의 원칙대로라면 Terraform 이 소유할 수 있는 쪽이다. 왜 IRSA 를 골랐는지는 적혀 있지 않다. → [결정 10](10-시크릿은-SSM에-Git에는-참조만.md)

## 배운 점

**경계를 정하는 것보다 경계를 건너는 방법을 정하는 게 일이다.**
"AWS API 는 Terraform, Kubernetes API 는 GitOps" 한 줄은 쉽게 나왔고 12일 내내 지켜졌다.
그런데 두 세계가 만나는 네 군데는 전부 명령어 안내문으로 남았고, 그중 하나(kubeconfig)가 실제로 작업을 막았다.
경계를 긋는 순간 **인계점 목록**도 같이 만들어야 했다.
