# 09 실습 — VPC + EKS 프로비저닝

앞의 8장을 하나로 합치는 실습이다. **VPC를 만들고 그 안에 EKS 클러스터를 세운 뒤, IRSA로 파드에 IAM 권한을 준다.**  
여기까지가 Terraform의 역할이고, 그 위에 앱을 배포하는 것은 GitOps의 몫이다. → `../../k8s-manifests/10-helm-gitops/`

---

## 목표 아키텍처

```
                        ┌─────────────── VPC 10.0.0.0/16 ───────────────┐
                        │                                                │
  인터넷 ──▶ IGW ──▶  퍼블릭 서브넷 (a, c)                                │
                        │    └─ NAT Gateway                              │
                        │           │                                    │
                        │           ▼                                    │
                        │   프라이빗 서브넷 (a, c)                         │
                        │    └─ EKS 관리형 노드그룹 (EC2)                  │
                        │                                                │
                        └────────────────────────────────────────────────┘
                                          ▲
                                          │ (관리형)
                                  EKS Control Plane
                                          │
                                     OIDC Provider ──▶ IRSA (파드별 IAM 역할)
```

- **노드는 프라이빗 서브넷에** 둔다. 외부에서 직접 접근할 수 없고, 아웃바운드는 NAT를 통한다
- **Control Plane은 AWS가 관리**한다. 우리가 만드는 건 노드그룹과 네트워크뿐이다
- **IRSA**로 파드가 IAM 역할을 맡는다 — 클러스터 안에 AWS 키를 두지 않는다

> 💸 **비용 주의:** EKS Control Plane은 시간당 과금(약 $0.10/h)이고 NAT Gateway도 시간당 + 데이터 처리 요금이 붙는다. **실습이 끝나면 반드시 `terraform destroy`** 한다. dev에서는 `single_nat_gateway = true`로 NAT를 1개만 쓴다.

---

## 0단계 — 백엔드 부트스트랩

state를 담을 S3 버킷은 Terraform으로 만들되, **그 자체는 로컬 state로 딱 한 번만** 만든다 (닭과 달걀 문제).

```
terraform/
├── bootstrap/          ← 로컬 state. S3 버킷 + 잠금 테이블만 만든다 (최초 1회)
├── modules/
│   ├── vpc/
│   └── eks/
└── environments/
    └── dev/            ← 원격 state. 실제 인프라
```

```hcl
# bootstrap/main.tf
resource "aws_s3_bucket" "tfstate" {
  bucket = "${var.project}-tfstate-${data.aws_caller_identity.current.account_id}"

  lifecycle {
    prevent_destroy = true      # 실수로 지우면 인프라 관리 권한을 잃는다
  }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

```bash
cd terraform/bootstrap
terraform init && terraform apply     # 최초 1회만
```

### 백엔드 값은 커밋하지 않는다

버킷 이름에 계정 ID가 들어가므로 코드에 박지 않고 init 시점에 주입한다. → `08-workspace-environment/`

```hcl
# environments/dev/backend.tf
terraform {
  backend "s3" {
    key     = "dev/terraform.tfstate"
    encrypt = true
    # bucket, region 등은 backend.hcl 에서 주입 (git-ignored)
  }
}
```

```bash
cd terraform/environments/dev
terraform init -backend-config=backend.hcl
```

> `backend.hcl.example`만 커밋하고 실제 `backend.hcl`은 `.gitignore`에 넣는다.

---

## 1단계 — VPC 모듈

```hcl
# modules/vpc/main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = var.name
  cidr = var.vpc_cidr
  azs  = var.azs

  private_subnets = [for i, az in var.azs : cidrsubnet(var.vpc_cidr, 8, i)]
  public_subnets  = [for i, az in var.azs : cidrsubnet(var.vpc_cidr, 8, i + 100)]

  enable_nat_gateway = true
  single_nat_gateway = var.single_nat_gateway   # dev=true(1개), prod=false(AZ별)
  enable_dns_hostnames = true

  # EKS가 서브넷 용도를 인식하기 위한 필수 태그
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
}
```

> **서브넷 태그가 없으면 로드밸런서가 안 만들어진다.** `Service type=LoadBalancer`나 Ingress를 만들었는데 pending에서 안 넘어가는 원인 1순위다.  
> 노드가 프라이빗에 있어도 퍼블릭 서브넷은 필요하다 — NAT과 외부용 로드밸런서가 거기 놓인다.

---

## 2단계 — EKS 모듈

```hcl
# modules/eks/main.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.24"

  cluster_name    = var.cluster_name
  cluster_version = var.kubernetes_version

  cluster_endpoint_public_access  = true    # 워크스테이션/CI에서 접근
  cluster_endpoint_private_access = true
  # 데모 밖이라면 cluster_endpoint_public_access_cidrs 로 반드시 좁힌다

  enable_cluster_creator_admin_permissions = true   # apply 한 주체에게 cluster-admin
  enable_irsa                              = true   # OIDC 프로바이더 생성

  cluster_addons = {
    coredns                = {}
    kube-proxy             = {}
    vpc-cni                = {}
    eks-pod-identity-agent = {}
  }

  vpc_id     = var.vpc_id
  subnet_ids = var.private_subnet_ids     # 노드는 프라이빗에

  eks_managed_node_group_defaults = {
    ami_type = "AL2023_x86_64_STANDARD"
  }

  eks_managed_node_groups = {
    default = {
      instance_types = var.node_instance_types
      capacity_type  = var.node_capacity_type   # dev는 SPOT 으로 비용 절감
      min_size       = var.node_min_size
      max_size       = var.node_max_size
      desired_size   = var.node_desired_size
    }
  }
}
```

### 인증 방식 — access entry (API 모드)

예전에는 `aws-auth` ConfigMap을 고쳐 클러스터 접근 권한을 줬다. 이 방식은 Terraform이 **Kubernetes 프로바이더까지 설정해야 해서** 순환 의존이 생겼다(클러스터가 있어야 프로바이더가 붙는데, 클러스터를 지금 만드는 중).

**access entry(API 모드)** 는 이걸 AWS API로 처리하므로, AWS 자격증명만으로 apply가 끝난다.

```
[예전] Terraform ──▶ EKS 생성 ──▶ Kubernetes provider ──▶ aws-auth ConfigMap 수정
                                    └─ 클러스터가 아직 없어서 plan 이 깨진다

[지금] Terraform ──▶ EKS 생성 + access entry (AWS API)
                     └─ Kubernetes provider 불필요
```

---

## 3단계 — 조립

```hcl
# environments/dev/main.tf
module "vpc" {
  source = "../../modules/vpc"

  name               = "${var.cluster_name}-vpc"
  vpc_cidr           = var.vpc_cidr
  azs                = var.azs
  single_nat_gateway = var.single_nat_gateway
}

module "eks" {
  source = "../../modules/eks"

  cluster_name       = var.cluster_name
  kubernetes_version = var.kubernetes_version

  vpc_id             = module.vpc.vpc_id           # ← 모듈 출력 → 모듈 입력
  private_subnet_ids = module.vpc.private_subnet_ids

  node_instance_types = var.node_instance_types
  node_capacity_type  = var.node_capacity_type
  node_desired_size   = var.node_desired_size
  node_min_size       = var.node_min_size
  node_max_size       = var.node_max_size
}
```

```hcl
# environments/dev/outputs.tf
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "oidc_provider_arn" {
  value = module.eks.oidc_provider_arn
}
```

```bash
terraform init -backend-config=backend.hcl
terraform plan
terraform apply          # EKS는 15~20분 걸린다. 정상이다
```

---

## 4단계 — kubeconfig 연결

```bash
aws eks update-kubeconfig \
  --region ap-northeast-2 \
  --name "$(terraform output -raw cluster_name)"

kubectl get nodes
kubectl get pods -A
```

> 클러스터를 지웠다 다시 만들면 `~/.kube/config`에 **옛 클러스터 정보가 남아** 인증 오류가 난다. `update-kubeconfig`를 다시 실행하거나 해당 컨텍스트를 지운다.  
> `terraform output -raw`를 쓰면 따옴표 없이 나와 그대로 셸에 넘길 수 있다. → `04-variables-outputs/`

---

## 5단계 — IRSA (파드에 IAM 권한 주기)

IRSA는 **파드의 ServiceAccount 토큰으로 IAM 역할을 맡는** 구조다. 클러스터 안에 AWS 액세스 키를 두지 않아도 된다.

```
파드 ─(ServiceAccount 토큰)─▶ EKS OIDC Provider ─▶ STS AssumeRoleWithWebIdentity
                                                         │
                                                         ▼
                                                   임시 자격증명 (IAM 역할)
```

```hcl
# 신뢰 정책 — "이 클러스터의, 이 네임스페이스의, 이 ServiceAccount만" 맡을 수 있다
data "aws_iam_policy_document" "external_secrets_assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [module.eks.oidc_provider_arn]
    }

    condition {
      test     = "StringEquals"
      variable = "${module.eks.oidc_provider}:sub"
      values   = ["system:serviceaccount:external-secrets:external-secrets"]
    }
  }
}

resource "aws_iam_role" "external_secrets" {
  name               = "${var.cluster_name}-external-secrets"
  assume_role_policy = data.aws_iam_policy_document.external_secrets_assume.json
}
```

```yaml
# 클러스터 쪽 — ServiceAccount에 역할 ARN을 어노테이션으로 연결
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets
  namespace: external-secrets
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/tf-lab-external-secrets
```

> **`sub` 조건을 빼면 안 된다.** 조건 없이 OIDC 프로바이더만 신뢰하면 그 클러스터의 **아무 파드나** 이 역할을 맡을 수 있다.  
> IAM 역할은 Terraform(클라우드 상태), ServiceAccount 어노테이션은 GitOps(클러스터 상태)에 둔다. **역할 ARN이 두 세계의 인계점**이다.

---

## 6단계 — 정리

```bash
terraform destroy       # 15분 정도 걸린다

# LoadBalancer 타입 Service 를 만들었다면 먼저 지운다
kubectl delete svc --all -A
```

> **클러스터 안에서 만든 AWS 리소스(ELB, EBS 볼륨)는 Terraform state에 없다.** 그래서 `destroy`가 VPC를 지우려다 "dependency violation"으로 실패한다.  
> 순서는 **① 클러스터 안 리소스 삭제 → ② terraform destroy**.

---

## 자주 막히는 지점

| 증상 | 원인 | 해결 |
|---|---|---|
| Ingress/LB가 pending에서 안 넘어감 | 서브넷에 `kubernetes.io/role/elb` 태그 없음 | VPC 모듈에 태그 추가 |
| `destroy`가 VPC에서 실패 | 클러스터가 만든 ELB·ENI가 남아있음 | K8s 리소스 먼저 삭제 |
| `kubectl` 인증 오류 | 옛 클러스터의 kubeconfig 잔존 | `aws eks update-kubeconfig` 재실행 |
| 노드가 NotReady | 프라이빗 서브넷에 NAT 경로 없음 | 라우트 테이블 확인 |
| apply가 EKS에서 20분 | 정상 | 기다린다 |
| 파드가 AWS API 403 | IRSA `sub` 조건의 네임스페이스/SA 이름 불일치 | 신뢰 정책의 `sub` 값 확인 |

---

## 여기서부터가 GitOps

Terraform이 만드는 것과 GitOps가 만드는 것의 경계를 정하는 게 중요하다.

| Terraform (클라우드 상태) | GitOps / Helm (클러스터 상태) |
|---|---|
| VPC, 서브넷, NAT | Deployment, Service, Ingress |
| EKS 클러스터, 노드그룹 | ArgoCD Application |
| IAM 역할·정책, OIDC | ServiceAccount(역할 ARN 어노테이션) |
| RDS, S3, SSM 파라미터 | ExternalSecret, ConfigMap |

> 경계가 흐려지면 "누가 이걸 관리하지"가 애매해진다. **AWS API로 만드는 건 Terraform, Kubernetes API로 만드는 건 GitOps**를 원칙으로 잡는다.  
> 실제 적용 사례는 [eks-gitops-platform](https://github.com/GWANG-MIN1/eks-gitops-platform) 저장소에 Phase 1~4로 정리되어 있다.

---

## 배운 점

- state 버킷은 **부트스트랩 디렉터리에서 로컬 state로 딱 한 번** 만든다 (닭과 달걀)
- 백엔드의 버킷 이름은 계정마다 다르므로 커밋하지 않고 `-backend-config`로 주입
- **노드는 프라이빗 서브넷, 퍼블릭 서브넷은 NAT·로드밸런서용**으로 여전히 필요하다
- `kubernetes.io/role/elb` · `internal-elb` **서브넷 태그가 없으면 LB가 안 만들어진다**
- dev는 `single_nat_gateway = true` + SPOT 인스턴스로 비용을 줄인다
- **access entry(API 모드)** 를 쓰면 Kubernetes 프로바이더 없이 apply가 끝난다 (순환 의존 해소)
- `enable_irsa = true`가 만드는 OIDC 프로바이더가 IRSA의 출발점
- IRSA 신뢰 정책에서 **`sub` 조건을 빼면 클러스터의 아무 파드나 역할을 맡을 수 있다**
- IAM 역할은 Terraform, ServiceAccount 어노테이션은 GitOps — **역할 ARN이 인계점**
- **클러스터가 만든 ELB·EBS는 state에 없다** → K8s 리소스를 먼저 지우고 destroy
- EKS 생성·삭제는 각각 15~20분 걸린다. 정상이다
- 경계 원칙: **AWS API로 만드는 건 Terraform, Kubernetes API로 만드는 건 GitOps**
