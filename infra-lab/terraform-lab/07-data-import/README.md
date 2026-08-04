# 07 Data Source & 임포트

실무에서 처음부터 Terraform으로 만든 인프라는 드물다. 이미 콘솔로 만들어 둔 것이 있고, 다른 팀이 관리하는 것도 있다.  
`data`는 **내가 관리하지 않는 것을 조회**하고, `import`는 **이미 있는 것을 Terraform 관리로 편입**한다. `moved`·`removed`는 그 이후의 리팩터링을 다룬다.

```
   조회만 한다        →  data 블록          (Terraform이 만들지도 지우지도 않음)
   관리로 가져온다     →  import 블록        (실제 리소스 → state 편입)
   코드 구조만 바꾼다  →  moved 블록         (state 주소 이동, 재생성 없음)
   관리에서 뗀다      →  removed 블록       (state에서만 제거, 실제는 유지)
```

---

## data — 기존 리소스 조회

```hcl
data "aws_vpc" "existing" {
  id = "vpc-0abc123"
}

resource "aws_subnet" "new" {
  vpc_id     = data.aws_vpc.existing.id      # data.타입.이름.속성
  cidr_block = "10.0.50.0/24"
}
```

`resource`와 `data`의 차이:

| | `resource` | `data` |
|---|---|---|
| 역할 | 생성·수정·삭제를 **관리** | **조회만** |
| state | 관리 대상으로 기록 | 조회 결과 캐시 |
| 코드에서 지우면 | 실제 리소스가 삭제됨 | 아무 일도 안 일어남 |
| 참조 | `aws_vpc.main.id` | `data.aws_vpc.existing.id` |

### 자주 쓰는 data source

```hcl
# 현재 리전·계정 정보 (하드코딩을 없앤다)
data "aws_region" "current" {}
data "aws_caller_identity" "current" {}
# → data.aws_caller_identity.current.account_id

# 사용 가능한 가용영역 (리전마다 다르므로 하드코딩 금지)
data "aws_availability_zones" "available" {
  state = "available"
}
# → data.aws_availability_zones.available.names

# 최신 AMI 조회 (AMI ID 하드코딩을 없앤다)
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
# → data.aws_ami.amazon_linux.id

# 태그로 기존 리소스 찾기
data "aws_vpc" "shared" {
  filter {
    name   = "tag:Name"
    values = ["shared-vpc"]
  }
}

# IAM 정책 문서 (JSON 문자열을 직접 쓰지 않는다)
data "aws_iam_policy_document" "s3_read" {
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.data.arn}/*"]
  }
}
```

> `data "aws_ami"`처럼 `most_recent = true`를 쓰면 **AMI가 갱신될 때마다 인스턴스가 재생성**될 수 있다. 운영에서는 조회한 ID를 변수로 고정하거나 `ignore_changes = [ami]`를 건다. → `06-meta-arguments/`

### 다른 state 참조 (remote state)

팀이 나뉘어 네트워크와 앱을 따로 관리할 때 쓴다.

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = "my-tfstate-bucket"
    key    = "infra-lab/network/terraform.tfstate"
    region = "ap-northeast-2"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.public_subnet_ids[0]
}
```

> 상대 state의 **`output`으로 내보낸 값만** 읽을 수 있다.  
> 다만 이 방식은 두 state를 강하게 묶는다. 태그 기반 `data` 조회나 SSM Parameter Store 경유가 결합도를 낮춘다.

---

## import — 기존 리소스를 관리로 편입

콘솔에서 만든 VPC를 Terraform이 관리하게 만드는 작업이다. **리소스를 새로 만들지 않고 state에만 연결한다.**

### 방법 1 — import 블록 (Terraform 1.5+, 권장)

```hcl
# 1) 임포트할 대상을 코드로 선언
import {
  to = aws_vpc.main
  id = "vpc-0abc123"        # 실제 AWS 리소스 ID
}

# 2) 대응하는 resource 블록 (비어 있어도 됨 — 설정 생성 기능 사용 시)
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

```bash
# 설정을 자동 생성해준다
terraform plan -generate-config-out=generated.tf

# 확인 후 적용
terraform plan       # Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
terraform apply
```

> import 블록의 장점은 **plan으로 미리 검증할 수 있고, 코드로 남아 리뷰된다**는 점이다.  
> 임포트가 끝나면 `import` 블록은 지워도 된다 (멱등하므로 남겨도 무해하다).

### 방법 2 — CLI (레거시)

```bash
terraform import aws_vpc.main vpc-0abc123
terraform import 'aws_instance.web[0]' i-0abc123
terraform import 'module.network.aws_vpc.this' vpc-0abc123
```

### 임포트 실무 절차

```
1. terraform state pull > backup.json      ← 먼저 백업
2. 대상 리소스의 실제 설정을 콘솔/CLI 로 확인
3. import 블록 + resource 블록 작성
4. terraform plan
     └─ "1 to import, 0 to change"  → 코드가 실제와 일치. OK
     └─ "1 to import, 1 to change"  → 코드가 실제와 다름. 코드를 고친다
     └─ "1 to destroy"              → 🚨 중단. 주소가 잘못됐다
5. terraform apply
```

> **`plan`에서 `destroy`나 예상 못 한 `change`가 보이면 절대 apply하지 않는다.** 운영 리소스가 날아간다.  
> ID 형식은 리소스마다 다르다. 서브넷은 `subnet-xxx`, IAM 역할은 역할 이름, 라우트는 `rtb-xxx_10.0.0.0/16` 같은 복합 키다. 프로바이더 문서의 **Import** 절을 확인한다.

---

## moved — 재생성 없이 주소 바꾸기

코드에서 리소스 **이름만** 바꾸면 Terraform은 다른 리소스로 인식해 **기존 것을 지우고 새로 만든다.** `moved` 블록이 이걸 막는다.

```hcl
# 이름 변경: aws_vpc.old → aws_vpc.main
moved {
  from = aws_vpc.old
  to   = aws_vpc.main
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

```
moved 없이:  Plan: 1 to add, 1 to destroy.      💥 재생성
moved 있이:  Plan: 0 to add, 0 to change, 0 to destroy.   ✅ 주소만 이동
```

### 모듈로 리팩터링할 때

```hcl
moved {
  from = aws_vpc.main
  to   = module.network.aws_vpc.this
}

moved {
  from = aws_subnet.public[0]
  to   = aws_subnet.public["ap-northeast-2a"]     # count → for_each 전환
}
```

> **`count`를 `for_each`로 바꿀 때 `moved` 블록이 필수다.** 안 쓰면 인덱스 기반 주소가 전부 사라지고 리소스가 통째로 재생성된다. → `06-meta-arguments/`  
> `moved`는 `terraform state mv` 명령과 같은 일을 하지만, **코드로 남아 리뷰되고 팀원이 자동으로 적용받는다**는 점이 낫다.

---

## removed — 관리에서만 떼어내기

리소스를 코드에서 지우고 apply하면 **실제 리소스도 삭제된다.** 실제는 살려두고 관리만 그만두려면 `removed` 블록을 쓴다 (Terraform 1.7+).

```hcl
removed {
  from = aws_instance.legacy

  lifecycle {
    destroy = false      # 실제 리소스는 지우지 않는다
  }
}
```

```
코드에서 삭제 + apply       →  실제 리소스도 삭제  💥
removed { destroy = false } →  state 에서만 제거, 실제는 유지  ✅
terraform state rm ...      →  같은 효과지만 코드에 안 남는다
```

---

## 정리 — 언제 무엇을 쓰나

| 하고 싶은 것 | 방법 |
|---|---|
| 남이 만든 걸 참조만 | `data` 블록 |
| 다른 state의 output 참조 | `data "terraform_remote_state"` |
| 콘솔에서 만든 걸 관리로 편입 | `import` 블록 → plan → apply |
| 리소스 이름만 변경 | `moved` 블록 |
| count → for_each 전환 | `moved` 블록 (필수) |
| 코드를 모듈로 리팩터링 | `moved` 블록 |
| 관리에서만 떼기 (실제는 유지) | `removed { lifecycle { destroy = false } }` |
| 진짜 삭제 | 코드에서 제거 후 apply |

---

## 배운 점

- `data`는 **조회만** 한다 — 코드에서 지워도 실제 리소스에 아무 일도 없다
- 리전·계정 ID·AZ·AMI는 하드코딩하지 말고 `data`로 조회한다
- `data "aws_ami"`에 `most_recent = true`는 편하지만 **인스턴스 재생성을 유발**할 수 있다
- `terraform_remote_state`로 다른 state의 **output만** 읽을 수 있다 (결합도가 높아지는 건 감안)
- **`import` 블록(1.5+)** 은 plan으로 검증되고 코드로 리뷰된다 — CLI `terraform import`보다 낫다
- `-generate-config-out`으로 임포트 대상의 설정을 자동 생성할 수 있다
- 임포트 plan에서 **`destroy`나 예상 못 한 `change`가 보이면 중단**한다
- 리소스 **이름만 바꿔도 재생성**된다 — `moved` 블록으로 막는다
- **`count` → `for_each` 전환에는 `moved`가 필수** (주소 체계가 통째로 바뀌므로)
- `moved`는 `state mv`와 같은 일을 하지만 코드로 남아 팀에 자동 적용된다
- 실제는 살리고 관리만 뗄 때는 `removed { lifecycle { destroy = false } }`
