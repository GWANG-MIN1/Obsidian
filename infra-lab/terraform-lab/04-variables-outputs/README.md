# 04 변수 & 출력

같은 코드로 dev와 prod를 만들려면 **변하는 값을 코드에서 분리**해야 한다.  
`variable`은 밖에서 받는 입력, `locals`는 코드 안에서 계산한 중간값, `output`은 밖으로 내보내는 결과다.

```
             ┌──────────────────────────────┐
 variable ──▶│   resource / locals / module │──▶ output
  (입력)      └──────────────────────────────┘     (출력)
```

---

## variable — 입력 변수

```hcl
variable "region" {
  description = "리소스를 만들 AWS 리전"
  type        = string
  default     = "ap-northeast-2"
}

variable "instance_type" {
  description = "EC2 인스턴스 타입"
  type        = string
  # default 이 없으면 필수 입력 — 안 주면 apply 시 물어본다
}

variable "db_password" {
  description = "RDS 마스터 비밀번호"
  type        = string
  sensitive   = true          # CLI 출력에서 가려진다 (state에는 평문 저장!)
}
```

사용은 `var.이름`:

```hcl
provider "aws" {
  region = var.region
}

resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

---

## 타입

| 타입 | 예시 |
|---|---|
| `string` | `"dev"` |
| `number` | `3` |
| `bool` | `true` |
| `list(string)` | `["a", "b"]` — 순서 있음, 중복 허용 |
| `set(string)` | `["a", "b"]` — 순서 없음, 중복 불가 |
| `map(string)` | `{ dev = "t3.micro", prod = "t3.large" }` |
| `object({...})` | 구조가 정해진 복합 타입 |
| `any` | 검증 없음 (피할수록 좋다) |

```hcl
variable "azs" {
  type    = list(string)
  default = ["ap-northeast-2a", "ap-northeast-2c"]
}

variable "instance_by_env" {
  type = map(string)
  default = {
    dev  = "t3.micro"
    prod = "t3.large"
  }
}

variable "vpc_config" {
  type = object({
    cidr        = string
    enable_nat  = bool
    subnet_bits = number
  })
  default = {
    cidr        = "10.0.0.0/16"
    enable_nat  = true
    subnet_bits = 8
  }
}
```

```hcl
# 참조
var.azs[0]                       # "ap-northeast-2a"
var.instance_by_env["prod"]      # "t3.large"
var.vpc_config.cidr              # "10.0.0.0/16"
```

---

## validation — 잘못된 값을 apply 전에 막기

```hcl
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "stg", "prod"], var.environment)
    error_message = "environment는 dev, stg, prod 중 하나여야 합니다."
  }
}

variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr는 올바른 CIDR 표기여야 합니다. (예: 10.0.0.0/16)"
  }
}
```

> 오타 하나로 prod에 리소스가 생기는 사고를 **plan 이전 단계에서** 막을 수 있다. 환경 이름·CIDR·인스턴스 타입에는 웬만하면 validation을 붙인다.

---

## 값 주입 방법과 우선순위

```bash
terraform apply -var="environment=prod"                # 1. CLI 직접 지정
terraform apply -var-file="envs/prod.tfvars"           # 2. 지정한 파일
#                                                        3. *.auto.tfvars (파일명 사전순)
#                                                        4. terraform.tfvars
export TF_VAR_environment=prod                         # 5. 환경변수
#                                                        6. variable 블록의 default
```

**위에 있을수록 우선한다.** 같은 변수를 여러 곳에서 주면 위쪽이 이긴다.

```hcl
# envs/prod.tfvars
environment   = "prod"
instance_type = "t3.large"
azs           = ["ap-northeast-2a", "ap-northeast-2c"]
```

```bash
terraform plan  -var-file=envs/prod.tfvars -out=tfplan
terraform apply tfplan
```

> `TF_VAR_` 환경변수는 CI에서 시크릿을 넘길 때 유용하다. `TF_VAR_db_password`처럼 넘기면 코드·파일 어디에도 값이 남지 않는다.  
> `terraform.tfvars`는 **자동으로 읽히므로** 시크릿을 넣고 커밋하는 사고가 잦다. `.gitignore`에 넣고 `terraform.tfvars.example`만 커밋한다.

---

## locals — 중간 계산값

`variable`이 "밖에서 받는 값"이라면 `locals`는 "코드 안에서 만든 값"이다. 밖에서 덮어쓸 수 없다.

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  # 조건 분기도 여기서 정리한다
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

  # 계산
  public_subnet_cidrs = [
    for i, az in var.azs : cidrsubnet(var.vpc_cidr, 8, i)
  ]
}
```

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags       = merge(local.common_tags, { Name = "${local.name_prefix}-vpc" })
}
```

> 참조는 `local.이름` (블록은 `locals`, 참조는 단수 `local`).  
> 같은 표현식이 세 군데 이상 반복되면 `locals`로 뺀다 — 이름을 붙이는 것만으로 코드가 읽힌다.

### variable vs locals

| | `variable` | `locals` |
|---|---|---|
| 값의 출처 | 밖 (CLI, tfvars, 환경변수) | 코드 안에서 계산 |
| 덮어쓰기 | 가능 | 불가 |
| 용도 | 환경마다 달라지는 값 | 반복되는 표현식·조합 결과 |

---

## output — 출력값

```hcl
output "vpc_id" {
  description = "생성된 VPC ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "퍼블릭 서브넷 ID 목록"
  value       = aws_subnet.public[*].id
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true          # 출력에서 가린다
}
```

```bash
terraform output                    # 전체
terraform output vpc_id             # 특정 값 ("vpc-0abc123" — 따옴표 포함)
terraform output -raw vpc_id        # vpc-0abc123 (스크립트용)
terraform output -json              # JSON

# 실전: 출력값을 바로 다음 명령에 넘긴다
aws eks update-kubeconfig --name "$(terraform output -raw cluster_name)"
```

### output의 세 가지 쓰임

1. **사람에게 보여주기** — apply 후 접속 주소·클러스터 이름
2. **스크립트·CI에 넘기기** — `-raw`, `-json`
3. **모듈의 반환값** — 모듈 밖에서 내부 리소스를 참조하는 유일한 통로 → `05-module/`

> 모듈 안의 리소스는 밖에서 직접 참조할 수 없다. `output`으로 내보낸 것만 `module.네트워크.vpc_id`로 쓸 수 있다.

---

## 실습 — 환경별로 갈라지는 구성

```hcl
# variables.tf
variable "project" {
  type    = string
  default = "tf-lab"
}

variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "prod"], var.environment)
    error_message = "environment는 dev 또는 prod 여야 합니다."
  }
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "azs" {
  type    = list(string)
  default = ["ap-northeast-2a", "ap-northeast-2c"]
}

# locals.tf
locals {
  name_prefix  = "${var.project}-${var.environment}"
  common_tags  = { Project = var.project, Environment = var.environment, ManagedBy = "Terraform" }
  enable_nat   = var.environment == "prod"
  subnet_cidrs = [for i, az in var.azs : cidrsubnet(var.vpc_cidr, 8, i)]
}

# outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = aws_subnet.public[*].id
}

output "name_prefix" {
  value = local.name_prefix
}
```

```bash
terraform plan -var-file=envs/dev.tfvars
terraform plan -var-file=envs/prod.tfvars    # 같은 코드, 다른 결과
```

---

## 배운 점

- 변하는 값은 `variable`로 빼고, 코드는 환경에 무관하게 유지한다
- `type`을 명시하면 잘못된 값이 들어올 때 apply 전에 걸린다 (`any`는 피한다)
- **`validation`** 으로 환경 이름·CIDR 같은 값을 사전에 막을 수 있다
- 값 주입 우선순위: `-var` > `-var-file` > `*.auto.tfvars` > `terraform.tfvars` > `TF_VAR_` > `default`
- `terraform.tfvars`는 자동으로 읽히므로 시크릿 커밋 사고에 주의 — `.example`만 커밋
- CI 시크릿은 `TF_VAR_이름` 환경변수로 넘기면 파일에 남지 않는다
- **`sensitive = true`는 CLI 출력만 가린다. state에는 평문으로 저장된다** → `03-state/`
- `locals`는 밖에서 못 바꾸는 내부 계산값 — 반복 표현식을 이름으로 정리할 때 쓴다
- 블록은 `locals`, 참조는 `local.이름` (단수)
- `output`은 사람에게 보여주기 + 스크립트 입력 + **모듈의 반환값** 세 가지 역할
- `terraform output -raw`는 따옴표 없이 출력해 스크립트에 바로 넘길 수 있다
