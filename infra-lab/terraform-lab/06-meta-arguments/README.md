# 06 메타 인수 & 표현식

**메타 인수(meta-argument)** 는 리소스 타입과 무관하게 모든 `resource`·`module` 블록에서 쓸 수 있는 공통 인수다.  
반복(`count`, `for_each`), 순서(`depends_on`), 수명주기(`lifecycle`), 프로바이더 선택(`provider`) 네 가지가 핵심이다.

---

## count — 개수로 반복

```hcl
resource "aws_instance" "web" {
  count = 3

  ami           = var.ami_id
  instance_type = "t3.micro"
  tags          = { Name = "web-${count.index}" }   # 0, 1, 2
}
```

```hcl
# 참조
aws_instance.web[0].id        # 개별
aws_instance.web[*].id        # 전체 (splat 표현식) → 리스트
```

### 조건부 생성 (count의 흔한 용법)

```hcl
resource "aws_nat_gateway" "this" {
  count = var.enable_nat ? 1 : 0        # false면 아예 만들지 않는다

  allocation_id = aws_eip.nat[0].id
  subnet_id     = var.public_subnet_id
}
```

> `count = 0`은 "리소스를 만들지 않는다"는 관용구다. 다만 참조할 때 항상 `[0]`을 붙여야 해서 코드가 지저분해진다.

### ⚠️ count의 함정 — 인덱스가 밀린다

```hcl
variable "names" {
  default = ["a", "b", "c"]
}

resource "aws_instance" "web" {
  count = length(var.names)
  tags  = { Name = var.names[count.index] }
}
```

여기서 `"a"`를 지우고 `["b", "c"]`로 바꾸면:

```
변경 전: [0]=a  [1]=b  [2]=c
변경 후: [0]=b  [1]=c
         └─ [0]은 a→b 로 '변경', [1]은 b→c 로 '변경', [2]는 '삭제'
            결과: 멀쩡한 b, c 인스턴스가 재생성된다 💥
```

**count는 위치(인덱스)로 리소스를 식별하므로, 중간 요소가 빠지면 뒤가 전부 밀린다.**

---

## for_each — 키로 반복

```hcl
resource "aws_instance" "web" {
  for_each = toset(["api", "worker", "batch"])

  ami           = var.ami_id
  instance_type = "t3.micro"
  tags          = { Name = each.key }
}

# 참조: aws_instance.web["api"].id
```

같은 상황에서 `"api"`를 지우면 **`api`만 삭제**되고 나머지는 그대로다. 키로 식별하기 때문이다.

### map으로 반복 — 값도 함께

```hcl
variable "instances" {
  type = map(object({
    instance_type = string
    az            = string
  }))

  default = {
    api    = { instance_type = "t3.small", az = "ap-northeast-2a" }
    worker = { instance_type = "t3.micro", az = "ap-northeast-2c" }
  }
}

resource "aws_instance" "app" {
  for_each = var.instances

  ami               = var.ami_id
  instance_type     = each.value.instance_type
  availability_zone = each.value.az
  tags              = { Name = each.key }
}
```

| | `each.key` | `each.value` |
|---|---|---|
| `for_each = toset([...])` | 요소 값 | 요소 값 (동일) |
| `for_each = { k = v }` | 키 | 값 |

### count vs for_each

| | `count` | `for_each` |
|---|---|---|
| 식별 방식 | 인덱스 (위치) | 키 (이름) |
| 중간 삭제 시 | **뒤가 전부 밀려 재생성** | 해당 것만 삭제 |
| 참조 | `res[0]` | `res["api"]` |
| 입력 타입 | number | set 또는 map |
| 쓸 때 | 조건부 생성(`0 or 1`), 진짜 동일한 N개 | **그 외 대부분** |

> **기본은 `for_each`, `count`는 조건부 생성에만.** 이것만 지켜도 "왜 멀쩡한 리소스가 재생성되지" 사고를 대부분 막는다.

```hcl
# 리스트를 for_each 에 쓰려면 set 또는 map 으로 변환
for_each = toset(var.names)                            # list → set
for_each = { for i, n in var.names : n => i }           # list → map
```

---

## depends_on — 명시적 순서

```hcl
resource "aws_instance" "app" {
  # ...
  depends_on = [
    aws_iam_role_policy.app,      # 정책이 먼저 붙어야 부팅 스크립트가 S3를 읽는다
    aws_nat_gateway.this,         # NAT 가 있어야 패키지를 받는다
  ]
}
```

> Terraform은 **참조 관계로 의존성을 자동 판단한다.** `depends_on`은 참조로 표현할 수 없는 숨은 의존이 있을 때만 쓴다.  
> 남발하면 병렬 실행이 막혀 apply가 느려지고, 그래프가 사람이 읽기 어려워진다.

---

## lifecycle — 수명주기 제어

```hcl
resource "aws_instance" "web" {
  # ...

  lifecycle {
    create_before_destroy = true       # 새로 만든 뒤 기존 것을 지운다 (무중단)
    prevent_destroy       = true       # destroy 시도를 에러로 막는다
    ignore_changes        = [tags["LastScanned"], ami]   # 이 속성의 변화는 무시
    replace_triggered_by  = [aws_launch_template.web.id] # 이게 바뀌면 강제 재생성
  }
}
```

| 인수 | 용도 |
|---|---|
| `create_before_destroy` | 교체 시 다운타임을 없앤다 (이름 충돌에 주의) |
| `prevent_destroy` | RDS·S3 등 지우면 안 되는 것에 안전장치 |
| `ignore_changes` | 외부(오토스케일러·콘솔)가 바꾸는 속성을 드리프트로 보지 않는다 |
| `replace_triggered_by` | 다른 리소스가 바뀌면 이것도 다시 만든다 |

```hcl
# 실전 예: ASG의 desired_capacity 는 오토스케일러가 바꾸므로 무시
resource "aws_autoscaling_group" "this" {
  desired_capacity = 2

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

> `prevent_destroy = true`가 걸린 리소스는 **`terraform destroy` 전체가 실패한다.** 진짜 지워야 할 땐 코드에서 이 줄을 먼저 지우고 apply해야 한다.  
> `ignore_changes`를 남용하면 코드와 실제가 조용히 벌어진다. "왜 무시하는지"를 주석으로 남긴다.

---

## provider — 프로바이더 선택

```hcl
provider "aws" {
  region = "ap-northeast-2"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}

resource "aws_acm_certificate" "cdn" {
  provider = aws.us            # CloudFront 인증서는 us-east-1 전용

  domain_name       = "example.com"
  validation_method = "DNS"
}

module "dr" {
  source    = "./modules/network"
  providers = { aws = aws.us }   # 모듈에 넘길 때는 providers (복수)
}
```

---

## dynamic 블록 — 중첩 블록 반복

`ingress`처럼 **블록**이 반복되는 경우, 인수와 달리 `for_each`를 직접 못 쓴다. 이때 `dynamic`을 쓴다.

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    cidr_blocks = list(string)
  }))

  default = [
    { port = 80,  cidr_blocks = ["0.0.0.0/0"] },
    { port = 443, cidr_blocks = ["0.0.0.0/0"] },
    { port = 22,  cidr_blocks = ["10.0.0.0/16"] },
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules

    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

> `dynamic "이름"` 안에서는 `이름.value`, `이름.key`로 접근한다 (`each`가 아니다).  
> 남용하면 읽기 어려워진다. 규칙이 3~4개로 고정이면 그냥 블록을 나열하는 편이 낫다.

---

## 자주 쓰는 표현식

### 조건 (삼항 연산자)

```hcl
instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
count         = var.enable_nat ? 1 : 0
```

### for 표현식

```hcl
# 리스트 → 리스트
[for s in var.subnets : upper(s)]

# 리스트 → 맵
{ for i, az in var.azs : az => cidrsubnet(var.vpc_cidr, 8, i) }

# 필터링
[for s in var.subnets : s.id if s.public]
```

### splat

```hcl
aws_instance.web[*].id                       # count 로 만든 것
[for k, v in aws_instance.app : v.id]        # for_each 로 만든 것은 splat 대신 for
values(aws_instance.app)[*].id               # 또는 values() 로 리스트화
```

### 자주 쓰는 내장 함수

```hcl
merge(local.common_tags, { Name = "web" })      # 맵 병합
lookup(var.map, "key", "default")               # 키 조회 + 기본값
coalesce(var.a, var.b, "fallback")              # 첫 번째 non-null
cidrsubnet("10.0.0.0/16", 8, 1)                 # → "10.0.1.0/24"
try(var.config.optional, "default")             # 에러 시 대체값
format("%s-%s", var.project, var.env)           # 문자열 포매팅
join(",", var.list)  /  split(",", var.str)
toset(var.list)  /  tolist(var.set)  /  tomap(...)
file("${path.module}/user_data.sh")             # 파일 읽기
templatefile("${path.module}/init.tpl", { name = var.name })
```

```bash
terraform console      # 표현식을 직접 실행해보며 익히는 게 가장 빠르다
```

---

## 배운 점

- 메타 인수는 리소스 타입과 무관하게 쓰는 공통 인수 — `count`·`for_each`·`depends_on`·`lifecycle`·`provider`
- **`count`는 인덱스(위치)로, `for_each`는 키(이름)로 리소스를 식별한다**
- count에서 중간 요소를 지우면 **뒤 리소스가 전부 밀려 재생성된다** — 가장 흔한 사고
- **기본은 `for_each`, `count`는 조건부 생성(`? 1 : 0`)에만 쓴다**
- 리스트는 `toset()`이나 map 변환을 거쳐야 `for_each`에 넣을 수 있다
- `depends_on`은 참조로 표현 못 하는 숨은 의존에만 — 남발하면 apply가 직렬화된다
- `lifecycle`: `create_before_destroy`(무중단), `prevent_destroy`(안전장치), `ignore_changes`(외부 변경 무시)
- `prevent_destroy`가 걸리면 `terraform destroy` 전체가 실패한다
- 여러 리전은 `provider ... alias` + 리소스의 `provider =` (모듈은 `providers =` 복수)
- 중첩 **블록**의 반복은 `dynamic` — 안에서는 `each`가 아니라 `블록이름.value`
- 표현식은 `terraform console`로 직접 실행해보며 익히는 게 가장 빠르다
