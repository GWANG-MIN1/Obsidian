# 02 Provider & Resource

Terraform 코드는 **HCL(HashiCorp Configuration Language)** 로 쓴다. 문법은 단순하다 — 블록·인수·표현식 세 가지가 전부다.  
이 장에서는 프로바이더를 붙이고, 리소스를 정의하고, 리소스끼리 참조로 연결하는 기본기를 다룬다.

---

## HCL 문법 구조

```hcl
블록타입 "레이블1" "레이블2" {
  인수 = 값

  중첩블록 {
    인수 = 값
  }
}
```

```hcl
resource "aws_instance" "web" {
  #  ↑블록타입   ↑타입      ↑이름
  ami           = "ami-0abc123"
  instance_type = "t3.micro"

  root_block_device {
    volume_size = 20
  }

  tags = {
    Name = "web-server"
  }
}
```

| 블록 | 역할 |
|---|---|
| `terraform` | Terraform 자체 설정 (버전, 프로바이더 요구사항, 백엔드) |
| `provider` | 대상 플랫폼 설정 (리전, 자격증명, 기본 태그) |
| `resource` | **만들 대상** — Terraform이 생성·수정·삭제를 관리 |
| `data` | 이미 있는 것을 **조회만** (→ `07-data-import/`) |
| `variable` / `output` / `locals` | 입력·출력·중간값 (→ `04-variables-outputs/`) |
| `module` | 다른 디렉터리의 코드를 불러 쓰기 (→ `05-module/`) |

---

## terraform 블록

```hcl
terraform {
  required_version = "~> 1.9"      # Terraform 자체 버전 제약

  required_providers {
    aws = {
      source  = "hashicorp/aws"    # 레지스트리 주소
      version = "~> 5.0"           # 프로바이더 버전 제약
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.30"
    }
  }
}
```

### 버전 제약 연산자

| 표기 | 의미 |
|---|---|
| `= 5.20.0` | 정확히 이 버전만 |
| `>= 5.0` | 5.0 이상 (상한 없음 — 권장하지 않음) |
| `~> 5.0` | 5.x 허용, 6.0은 불가 (**마이너까지 허용**) |
| `~> 5.20.0` | 5.20.x 허용, 5.21은 불가 (**패치까지만 허용**) |

> 실무 기본은 `~> 5.0`처럼 **메이저를 고정**하는 것이다. 메이저 업그레이드는 파괴적 변경을 포함하므로 의도적으로 올린다.  
> 실제로 쓸 버전을 못 박는 건 `.terraform.lock.hcl`이다. `terraform init -upgrade`로 갱신한다.

---

## provider 블록

```hcl
provider "aws" {
  region = "ap-northeast-2"

  default_tags {                  # 이 프로바이더로 만드는 모든 리소스에 자동 부착
    tags = {
      ManagedBy   = "Terraform"
      Environment = "dev"
      Project     = "infra-lab"
    }
  }
}
```

### 자격증명 — 코드에 쓰지 않는다

```hcl
provider "aws" {
  region     = "ap-northeast-2"
  access_key = "AKIA..."     # ❌ 절대 금지 — Git에 올라가는 순간 유출
  secret_key = "..."         # ❌
}
```

권장 순서:

```bash
# 1. 로컬 개발 — 프로필
export AWS_PROFILE=dev
aws configure --profile dev

# 2. EC2 / EKS 등 AWS 위 — 인스턴스 역할 · IRSA (자격증명 자체가 없다)

# 3. CI/CD — OIDC 임시 자격증명 (장기 키 없음)  → 10-cicd-policy/
```

> Terraform은 AWS SDK와 동일한 순서로 자격증명을 찾는다: 환경변수 → 공유 설정 파일(`~/.aws/credentials`) → 인스턴스 메타데이터.  
> 코드에는 리전 정도만 남기고 **자격증명은 코드 밖에서 주입**하는 것이 원칙이다.

### 여러 리전·계정 쓰기 (alias)

```hcl
provider "aws" {
  region = "ap-northeast-2"       # 기본
}

provider "aws" {
  alias  = "us"                   # 별칭 지정
  region = "us-east-1"
}

resource "aws_acm_certificate" "cf" {
  provider = aws.us               # CloudFront 인증서는 us-east-1에만 만들 수 있다
  domain_name = "example.com"
  validation_method = "DNS"
}
```

---

## resource 블록

```hcl
resource "aws_vpc" "main" {
  #        ↑ 리소스 타입    ↑ 로컬 이름(코드 안에서만 쓰는 식별자)
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "tf-lab-vpc"
  }
}
```

- **리소스 타입**은 프로바이더가 정한다 (`aws_vpc`, `aws_subnet`, `aws_iam_role`…)
- **로컬 이름**은 내가 정한다. 실제 AWS 이름이 아니라 코드 안의 주소일 뿐이다
- 코드 안 주소는 `aws_vpc.main` 형태 — state에도 이 주소로 기록된다

> 로컬 이름을 바꾸면 Terraform은 **다른 리소스로 인식해 기존 것을 지우고 새로 만든다.** 단순 리네임은 `moved` 블록을 쓴다. → `07-data-import/`

---

## 리소스 참조와 의존성

한 리소스의 속성을 다른 리소스에서 `타입.이름.속성`으로 참조한다.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id            = aws_vpc.main.id       # ← 참조
  cidr_block        = "10.0.1.0/24"
  availability_zone = "ap-northeast-2a"
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}
```

### 암묵적 의존성 vs 명시적 의존성

```
aws_vpc.main ──(vpc_id 참조)──▶ aws_subnet.public
             └─ Terraform이 참조를 보고 순서를 자동으로 결정한다
```

```hcl
# 참조 관계가 없지만 순서를 강제해야 할 때만
resource "aws_instance" "app" {
  # ...
  depends_on = [aws_iam_role_policy.app]     # 정책이 먼저 붙어야 부팅 스크립트가 동작
}
```

> **암묵적 의존성이 기본이고 `depends_on`은 예외다.** 참조로 표현할 수 있으면 참조를 쓴다. `depends_on`을 남발하면 그래프가 직렬화되어 apply가 느려진다.

```bash
terraform graph | dot -Tsvg > graph.svg    # 의존성 그래프 시각화
```

### 아직 모르는 값 (known after apply)

```
+ resource "aws_subnet" "public" {
    + id     = (known after apply)      ← 만들기 전에는 알 수 없는 값
    + vpc_id = "vpc-0abc123"
  }
```

ID·ARN처럼 클라우드가 생성 시점에 부여하는 값은 plan 단계에서 확정되지 않는다. 정상이다.

---

## 실습 — VPC + 퍼블릭 서브넷

```hcl
# main.tf
terraform {
  required_version = "~> 1.9"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}

provider "aws" {
  region = "ap-northeast-2"
  default_tags {
    tags = { ManagedBy = "Terraform", Project = "tf-lab" }
  }
}

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags                 = { Name = "tf-lab-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "tf-lab-igw" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-northeast-2a"
  map_public_ip_on_launch = true
  tags                    = { Name = "tf-lab-public-a" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = { Name = "tf-lab-public-rt" }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

```bash
terraform init
terraform fmt
terraform validate
terraform plan       # Plan: 5 to add, 0 to change, 0 to destroy.
terraform apply
terraform destroy    # 실습 후 반드시 정리 (NAT·EIP는 과금된다)
```

---

## 코드 품질 명령

```bash
terraform fmt -recursive        # 들여쓰기·정렬 정규화
terraform fmt -check -diff      # 고치지 않고 검사만 (CI에서 사용)
terraform validate              # 문법·타입·필수 인수 검증 (클라우드 접근 없음)
terraform console               # 표현식을 대화형으로 평가
```

```bash
$ terraform console
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
```

> `validate`는 **클라우드에 접근하지 않는다.** 자격증명 없이도 CI에서 돌릴 수 있다(`terraform init -backend=false` 후).  
> 반대로 `plan`은 실제 API를 호출해 현재 상태를 읽으므로 자격증명이 필요하다.

---

## 배운 점

- HCL은 **블록 · 인수 · 표현식** 세 가지가 전부다
- `required_version`·`required_providers`로 버전을 고정하고, 실제 버전은 `.terraform.lock.hcl`이 못 박는다
- 버전 제약은 `~> 5.0`(메이저 고정)이 실무 기본
- **자격증명은 코드에 쓰지 않는다** — 프로필·인스턴스 역할·OIDC로 밖에서 주입
- `default_tags`로 모든 리소스에 공통 태그를 자동 부착할 수 있다
- 여러 리전·계정은 `provider ... alias`로 나누고 리소스에서 `provider = aws.us`로 선택
- 리소스의 **로컬 이름은 코드 안의 주소**일 뿐 — 바꾸면 재생성되므로 `moved` 블록을 쓴다
- 의존성은 **참조로 자동 결정(암묵적)** 되며 `depends_on`은 예외적으로만
- `(known after apply)`는 생성 후에야 값이 정해진다는 정상 표시
- `validate`는 클라우드 접근 없이, `plan`은 실제 API를 호출해 동작한다
