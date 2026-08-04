# 08 환경 분리 전략

dev·stg·prod를 같은 코드로 만들되, **state와 폭발 반경(blast radius)은 분리해야 한다.**  
"dev에서 `terraform destroy` 했는데 prod가 지워졌다"는 사고가 나오는 지점이 바로 여기다.

---

## 무엇을 분리해야 하나

| 분리 대상 | 이유 |
|---|---|
| **state** | 환경별로 독립적으로 apply·destroy 되어야 한다 |
| **변수 값** | 인스턴스 타입·용량·도메인이 다르다 |
| **권한** | dev 자격증명으로 prod를 건드릴 수 없어야 한다 |
| **적용 시점** | dev는 자동 apply, prod는 승인 후 apply |

> 핵심은 state 분리다. **하나의 state에 dev와 prod가 함께 있으면 그 순간 사고 반경이 전체가 된다.**

---

## 방법 1 — Workspace

Terraform 내장 기능. 하나의 백엔드 안에서 state를 여러 개로 나눈다.

```bash
terraform workspace list          # 목록 (* 가 현재)
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform workspace show
terraform workspace delete dev
```

state는 백엔드 안에서 이렇게 나뉜다.

```
s3://my-tfstate-bucket/
├── infra-lab/terraform.tfstate                    ← default workspace
└── env:/
    ├── dev/infra-lab/terraform.tfstate
    └── prod/infra-lab/terraform.tfstate
```

```hcl
locals {
  environment   = terraform.workspace                     # 현재 workspace 이름
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
  replica_count = terraform.workspace == "prod" ? 3 : 1
}

resource "aws_instance" "web" {
  instance_type = local.instance_type
  tags          = { Name = "web-${terraform.workspace}" }
}
```

### workspace의 한계

| 한계 | 설명 |
|---|---|
| **같은 백엔드·같은 자격증명** | dev와 prod가 같은 계정·같은 버킷을 쓴다. 권한 분리가 안 된다 |
| **전환을 잊는다** | `select`를 안 하고 apply해서 엉뚱한 환경에 적용하는 사고가 잦다 |
| **환경 차이 표현이 어렵다** | prod에만 있는 리소스를 만들려면 `count = terraform.workspace == "prod" ? 1 : 0` 이 코드에 번진다 |
| **눈에 안 보인다** | 코드만 봐서는 지금 어느 환경인지 알 수 없다 |

> 공식 문서도 workspace를 **"같은 구성의 짧은 수명 변형"** (기능 브랜치 테스트, PR 미리보기 환경)에 쓰라고 안내한다.  
> **dev/stg/prod 같은 장기 환경 분리에는 권장되지 않는다.**

---

## 방법 2 — 디렉터리 분리 (권장)

환경마다 디렉터리를 두고, 공통 로직은 모듈로 뺀다.

```
infra/
├── modules/                    # 공통 로직 (환경 무관)
│   ├── network/
│   ├── eks/
│   └── app/
└── envs/
    ├── dev/
    │   ├── main.tf             # 모듈 호출
    │   ├── backend.tf          # dev 전용 state key
    │   └── terraform.tfvars    # dev 값
    ├── stg/
    └── prod/
        ├── main.tf
        ├── backend.tf          # prod 전용 state key
        └── terraform.tfvars
```

```hcl
# envs/prod/backend.tf
terraform {
  backend "s3" {
    bucket       = "my-tfstate-bucket"
    key          = "infra-lab/prod/terraform.tfstate"   # ← 환경별로 다른 key
    region       = "ap-northeast-2"
    encrypt      = true
    use_lockfile = true
  }
}
```

```hcl
# envs/prod/main.tf
module "network" {
  source = "../../modules/network"

  name_prefix = "tf-lab-prod"
  vpc_cidr    = "10.10.0.0/16"
  azs         = ["ap-northeast-2a", "ap-northeast-2c"]
  enable_nat  = true
}

module "eks" {
  source = "../../modules/eks"

  cluster_name  = "tf-lab-prod"
  vpc_id        = module.network.vpc_id
  subnet_ids    = module.network.private_subnet_ids
  desired_size  = 3
}
```

```bash
cd envs/prod
terraform init
terraform plan
terraform apply       # prod 디렉터리에서만 prod가 바뀐다
```

### 장점

- **어느 환경인지 경로로 명확하다** — `cd envs/prod`
- state·백엔드·자격증명을 환경마다 완전히 분리할 수 있다 (계정 분리까지 가능)
- prod에만 있는 리소스를 조건문 없이 그냥 추가하면 된다
- CI에서 변경된 디렉터리만 골라 plan을 돌릴 수 있다

### 단점

- 환경 수만큼 `main.tf`가 중복된다 (모듈로 빼면 호출부만 남아 대부분 해소된다)

---

## 방법 3 — backend-config 주입

디렉터리는 하나로 두고, 백엔드 설정만 파일로 갈아 끼우는 방식이다.  
백엔드 블록에는 변수를 쓸 수 없기 때문에 이 방법이 필요하다.

```hcl
# backend.tf — 값은 비워둔다 (partial configuration)
terraform {
  backend "s3" {}
}
```

```hcl
# backends/dev.hcl
bucket       = "my-tfstate-bucket"
key          = "infra-lab/dev/terraform.tfstate"
region       = "ap-northeast-2"
encrypt      = true
use_lockfile = true
```

```bash
terraform init -backend-config=backends/dev.hcl -reconfigure
terraform apply -var-file=envs/dev.tfvars

# 환경 전환 시 반드시 -reconfigure 로 다시 init
terraform init -backend-config=backends/prod.hcl -reconfigure
```

> 편리하지만 **`init`을 다시 안 하고 apply하면 이전 환경의 state를 건드린다.** 사람이 실수할 여지가 커서, 자동화 없이 손으로 쓰기에는 위험하다. CI에서 스크립트로 강제할 때 적합하다.

---

## 세 방법 비교

| | Workspace | 디렉터리 분리 | backend-config |
|---|---|---|---|
| state 분리 | ✅ (같은 백엔드 내) | ✅ (완전 분리) | ✅ |
| 계정·권한 분리 | ❌ | ✅ | △ |
| 현재 환경 가시성 | ❌ (명령 필요) | ✅ (경로) | ❌ |
| 환경별 구조 차이 | ❌ 어렵다 | ✅ 자유롭다 | ❌ 어렵다 |
| 실수 위험 | 높음 (select 누락) | 낮음 | 높음 (init 누락) |
| 코드 중복 | 없음 | 호출부만 중복 | 없음 |
| 적합한 용도 | PR 미리보기 등 단기 환경 | **dev/stg/prod 장기 환경** | CI 자동화 |

> **결론: 장기 환경은 디렉터리 분리, 계정까지 나누는 것이 안전하다.** workspace는 "같은 구성의 임시 복제본"에 쓴다.

---

## Terragrunt — 디렉터리 분리의 중복 줄이기

디렉터리 분리의 유일한 단점인 백엔드·프로바이더 설정 중복을 해결하는 래퍼 도구다.

```hcl
# terragrunt.hcl (루트) — 백엔드를 한 번만 정의
remote_state {
  backend = "s3"
  config = {
    bucket  = "my-tfstate-bucket"
    key     = "${path_relative_to_include()}/terraform.tfstate"   # 경로로 자동 분리
    region  = "ap-northeast-2"
    encrypt = true
  }
}
```

```hcl
# envs/prod/network/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/network"
}

inputs = {
  name_prefix = "tf-lab-prod"
  vpc_cidr    = "10.10.0.0/16"
}
```

```bash
terragrunt plan
terragrunt run-all apply     # 하위 모든 스택을 의존 순서대로
```

| 장점 | 단점 |
|---|---|
| 백엔드·프로바이더 설정 DRY | 도구를 하나 더 배워야 한다 |
| `run-all`로 여러 스택 일괄 실행 | 팀원이 Terraform만 알면 진입장벽 |
| 의존성 있는 스택 순서 관리 | Terraform 신기능 지원이 한 박자 늦을 수 있다 |

> 환경·스택이 10개를 넘어가기 전에는 굳이 도입하지 않아도 된다. **먼저 디렉터리 분리로 시작하고, 중복이 실제로 아플 때 검토한다.**

---

## 스택 쪼개기 (state 크기 관리)

환경 분리와 별개로, **하나의 state가 너무 커지면** plan이 느려지고 사고 반경이 커진다. 변경 주기가 다른 것끼리 나눈다.

```
envs/prod/
├── 10-network/      # 거의 안 바뀜 (VPC, 서브넷)
├── 20-platform/     # 가끔 바뀜 (EKS, RDS)
└── 30-apps/         # 자주 바뀜 (앱 리소스)
```

- 앱 배포 때마다 VPC까지 리프레시할 이유가 없다
- 네트워크 state가 깨져도 앱은 영향받지 않는다
- 스택 간 참조는 `terraform_remote_state`나 태그 기반 `data` 조회로 → `07-data-import/`

---

## 배운 점

- 환경 분리의 핵심은 **state 분리** — 한 state에 dev와 prod가 있으면 사고 반경이 전체가 된다
- **workspace는 장기 환경(dev/stg/prod) 분리용이 아니다** — 같은 백엔드·같은 권한을 공유한다
- workspace는 PR 미리보기처럼 **같은 구성의 짧은 수명 변형**에 적합하다
- `terraform.workspace`로 환경 이름을 참조할 수 있지만, 조건문이 코드 전체에 번진다
- **장기 환경은 디렉터리 분리(`envs/dev`, `envs/prod`) + 공통 로직은 모듈**이 안전하다
- 디렉터리 분리는 **경로만 봐도 어느 환경인지 알 수 있다**는 게 가장 큰 장점
- 백엔드 블록에는 변수를 못 쓴다 → `-backend-config=파일.hcl`로 주입 (전환 시 `-reconfigure` 필수)
- Terragrunt는 디렉터리 분리의 중복을 없애주지만, 아프기 전에는 도입하지 않아도 된다
- 변경 주기가 다른 리소스는 스택으로 쪼갠다 (network / platform / apps)
