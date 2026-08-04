# 05 모듈

VPC 한 세트를 만드는 데 필요한 리소스는 서브넷·라우트테이블·NAT·IGW까지 10개가 넘는다. 이걸 dev/stg/prod마다 복붙하면 관리가 불가능해진다.  
**모듈은 관련 리소스를 하나의 재사용 단위로 묶는 장치다.** 함수에 인수를 넘기고 반환값을 받는 것과 같다.

```
   variable          ┌─────────────┐         output
  (입력 인수)   ────▶ │   module    │ ────▶  (반환값)
                     │  리소스 N개  │
                     └─────────────┘
```

---

## 모든 Terraform 코드는 이미 모듈이다

`terraform apply`를 실행하는 디렉터리를 **루트 모듈(root module)** 이라 부른다.  
다른 디렉터리를 불러 쓰면 그것이 **자식 모듈(child module)** 이다.

```
envs/dev/            ← 루트 모듈 (여기서 apply)
└── module "network" → modules/network/    ← 자식 모듈
    module "eks"     → modules/eks/
```

---

## 모듈 만들기

관례상 파일 3개로 나눈다.

```
modules/network/
├── main.tf        # 리소스 정의
├── variables.tf   # 입력 (모듈의 인터페이스)
└── outputs.tf     # 출력 (모듈의 반환값)
```

```hcl
# modules/network/variables.tf
variable "name_prefix" {
  description = "리소스 이름 접두사"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR"
  type        = string
}

variable "azs" {
  description = "사용할 가용영역 목록"
  type        = list(string)
}

variable "tags" {
  description = "공통 태그"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/network/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  tags                 = merge(var.tags, { Name = "${var.name_prefix}-vpc" })
}

resource "aws_subnet" "public" {
  for_each = { for i, az in var.azs : az => i }

  vpc_id            = aws_vpc.this.id
  availability_zone = each.key
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, each.value)
  tags              = merge(var.tags, { Name = "${var.name_prefix}-public-${each.key}" })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  tags   = merge(var.tags, { Name = "${var.name_prefix}-igw" })
}
```

```hcl
# modules/network/outputs.tf
output "vpc_id" {
  description = "생성된 VPC ID"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "퍼블릭 서브넷 ID 목록"
  value       = [for s in aws_subnet.public : s.id]
}
```

> 모듈 안 리소스의 로컬 이름은 관례상 **`this`** 를 쓴다. `aws_vpc.network_vpc`처럼 쓰면 `module.network.aws_vpc.network_vpc`가 되어 이름이 중복된다.

---

## 모듈 호출하기

```hcl
# envs/dev/main.tf
module "network" {
  source = "../../modules/network"     # 상대 경로

  name_prefix = "tf-lab-dev"
  vpc_cidr    = "10.0.0.0/16"
  azs         = ["ap-northeast-2a", "ap-northeast-2c"]
  tags        = local.common_tags
}

module "eks" {
  source = "../../modules/eks"

  cluster_name = "tf-lab-dev"
  vpc_id       = module.network.vpc_id                # ← 모듈 출력을 다음 모듈 입력으로
  subnet_ids   = module.network.public_subnet_ids
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}
```

```bash
terraform init      # 모듈을 불러오려면 반드시 init 을 다시 돌린다
terraform get -update
```

> **모듈을 추가하거나 source를 바꿨으면 `terraform init`을 다시 실행해야 한다.** "Module not installed" 에러의 대부분이 이것이다.  
> 모듈 밖에서는 `output`으로 내보낸 값만 참조할 수 있다. 내부 리소스를 직접 건드릴 수 없는 것이 모듈의 캡슐화다.

---

## source 종류

| source | 예시 | 비고 |
|---|---|---|
| **로컬 경로** | `"../../modules/network"` | 반드시 `./` 또는 `../`로 시작 |
| **Terraform Registry** | `"terraform-aws-modules/vpc/aws"` | `version` 지정 가능 |
| **Git (HTTPS)** | `"git::https://github.com/org/repo.git//modules/vpc?ref=v1.2.0"` | `ref`로 태그·브랜치 고정 |
| **Git (SSH)** | `"git::ssh://git@github.com/org/repo.git//modules/vpc?ref=v1.2.0"` | 프라이빗 저장소 |
| **S3 / GCS** | `"s3::https://s3-....amazonaws.com/bucket/vpc.zip"` | 사내 배포용 |

```hcl
# 레지스트리 모듈 — version 인수를 쓸 수 있다
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  # ...
}

# Git 모듈 — 경로 구분은 // 두 개, 버전은 ?ref=
module "network" {
  source = "git::https://github.com/GWANG-MIN1/tf-modules.git//network?ref=v1.0.0"
}
```

> **버전을 반드시 고정한다.** `?ref` 없이 Git 모듈을 쓰면 기본 브랜치를 따라가서, 어제 되던 apply가 오늘 깨진다.  
> 로컬 경로 모듈에는 `version` 인수를 쓸 수 없다 (파일 시스템에는 버전 개념이 없다).

---

## 레지스트리 모듈 활용

바퀴를 다시 만들 필요는 없다. VPC·EKS처럼 정형화된 것은 검증된 공개 모듈을 쓴다.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "tf-lab-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-2a", "ap-northeast-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true      # dev에서는 NAT 1개로 비용 절감

  tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}
```

> 공개 모듈을 쓸 때는 **버전을 고정하고, 소스 코드를 한 번은 읽어본다.** 인수 이름이 버전마다 바뀌고, 내가 모르는 리소스를 만들 수도 있다.  
> registry.terraform.io에서 `Inputs`/`Outputs` 탭이 곧 모듈의 API 문서다.

---

## 모듈 설계 원칙

| 원칙 | 이유 |
|---|---|
| **하나의 책임** | "네트워크", "EKS 클러스터"처럼 한 덩어리만. 모든 걸 담는 거대 모듈은 재사용되지 않는다 |
| **입력은 최소로** | 인수가 30개면 그건 모듈이 아니라 복잡한 복붙이다 |
| **provider를 모듈 안에 두지 않는다** | 모듈은 provider 설정을 상속받는다. 안에 정의하면 재사용성이 깨진다 |
| **하드코딩 금지** | 리전·계정 ID·이름을 박아두면 그 모듈은 한 곳에서만 쓸 수 있다 |
| **출력을 충분히 노출** | 밖에서 필요한 ID·ARN을 output으로 열어둔다 |
| **README와 예제** | `terraform-docs`로 변수·출력 표를 자동 생성한다 |

```bash
terraform-docs markdown table ./modules/network > ./modules/network/README.md
```

### 모듈로 뺄 타이밍

```
같은 코드를 두 번째 복붙하려는 순간
       ↓
그때가 모듈로 뺄 때다 (한 번만 쓸 거면 굳이 모듈로 만들지 않는다)
```

> 초보자가 가장 많이 하는 실수는 **처음부터 과하게 모듈화**하는 것이다. 리소스 3개짜리를 모듈로 감싸면 간접 계층만 늘고 읽기 어려워진다.

---

## 모듈 반복 호출

모듈에도 `count`와 `for_each`를 쓸 수 있다. → `06-meta-arguments/`

```hcl
module "app" {
  source   = "../../modules/service"
  for_each = toset(["api", "worker", "batch"])

  name       = each.key
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.public_subnet_ids
}

# 참조: module.app["api"].service_url
```

---

## 배운 점

- **apply를 실행하는 디렉터리가 루트 모듈**이고, 불러 쓰는 것이 자식 모듈이다
- 모듈은 `main.tf` / `variables.tf`(입력) / `outputs.tf`(출력) 세 파일 구조가 관례
- 모듈 내부 리소스의 로컬 이름은 **`this`** 를 쓴다 (`module.network.aws_vpc.this`)
- 모듈 밖에서는 **`output`으로 내보낸 값만** 참조 가능 — 이게 캡슐화다
- 한 모듈의 output을 다른 모듈의 input으로 넘겨 조립한다
- **모듈을 추가·변경하면 `terraform init`을 다시 돌려야 한다**
- source는 로컬 경로 / 레지스트리 / Git 등 — **Git은 `?ref=태그`로 버전을 반드시 고정**
- 로컬 경로 모듈에는 `version` 인수를 쓸 수 없다
- VPC·EKS처럼 정형화된 것은 검증된 공개 모듈을 쓰되 버전 고정 + 코드 확인
- 모듈은 **하나의 책임**, provider는 모듈 안에 두지 않는다
- 두 번째 복붙하려는 순간이 모듈로 뺄 타이밍 — 처음부터 과하게 모듈화하지 않는다
