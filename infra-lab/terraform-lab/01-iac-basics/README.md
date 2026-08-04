# 01 IaC 기초

**Infrastructure as Code(IaC)** 는 서버·네트워크·권한 같은 인프라를 콘솔 클릭이 아니라 **코드로 정의하고 버전 관리**하는 방식이다.  
콘솔로 만든 인프라는 재현할 수 없고, 리뷰할 수 없고, 누가 왜 바꿨는지 알 수 없다. 코드로 옮기는 순간 인프라도 Git 이력·PR 리뷰·롤백의 대상이 된다.

---

## 수동 인프라의 문제

```
[ 콘솔 클릭 ]
개발자 A ──▶ 콘솔 ──▶ dev 환경 (보안그룹 3개, t3.micro)
개발자 B ──▶ 콘솔 ──▶ prod 환경 (보안그룹 5개, t3.small, 누가 열었는지 모를 0.0.0.0/0)
                        └─ "dev에서 됐는데 prod에서 안 되는데요"
```

| 문제 | 설명 |
|---|---|
| **스노우플레이크 서버** | 손으로 조금씩 고친 결과 아무도 똑같이 재현하지 못하는 유일한 서버가 된다 |
| **환경 불일치** | dev/stg/prod가 미묘하게 달라 "내 환경에선 됐는데"가 반복된다 |
| **변경 이력 부재** | CloudTrail을 뒤져야 누가 언제 뭘 바꿨는지 알 수 있다 |
| **리뷰 불가** | 보안그룹을 전체 개방해도 코드 리뷰에 걸리지 않는다 |
| **재해 복구** | 리전 장애 시 처음부터 손으로 다시 만들어야 한다 |
| **문서 노후** | 위키의 아키텍처 문서와 실제 인프라가 항상 다르다 |

> IaC의 진짜 가치는 "자동화"보다 **"인프라를 리뷰 가능한 코드로 만든다"** 는 데 있다.

---

## 선언형 vs 명령형

| 구분 | 명령형 (How) | 선언형 (What) |
|---|---|---|
| 기술 방식 | 무엇을 **어떻게** 할지 절차를 적음 | 최종 **상태**를 적음 |
| 예시 | "EC2를 만들어라 → 보안그룹을 붙여라" | "EC2 1대가 이 보안그룹과 함께 존재한다" |
| 재실행 | 두 번 실행하면 두 개가 생길 수 있음 | 이미 있으면 아무것도 안 함 (**멱등성**) |
| 도구 | 셸 스크립트, AWS CLI | **Terraform**, CloudFormation, Kubernetes |

```bash
# 명령형 — 두 번 실행하면 EC2가 두 대 생긴다
aws ec2 run-instances --image-id ami-xxx --instance-type t3.micro
```

```hcl
# 선언형 — 몇 번을 실행해도 EC2는 한 대다
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"
}
```

> Kubernetes 매니페스트를 `kubectl apply` 하는 것과 정확히 같은 사고방식이다. Terraform은 그 대상이 클러스터 리소스가 아니라 클라우드 인프라일 뿐이다. → `../k8s-manifests/`

---

## IaC 도구 비교

| 도구 | 방식 | 언어 | 범위 | 특징 |
|---|---|---|---|---|
| **Terraform** | 선언형 (프로비저닝) | HCL | 멀티 클라우드 | state로 실제 리소스를 추적, 생태계가 가장 넓다 |
| **CloudFormation** | 선언형 (프로비저닝) | YAML/JSON | AWS 전용 | AWS 네이티브, state를 AWS가 관리 |
| **Pulumi** | 선언형 (프로비저닝) | TS·Python·Go | 멀티 클라우드 | 범용 언어로 작성 (반복·조건이 자유롭다) |
| **CDK / CDKTF** | 선언형 (프로비저닝) | TS·Python 등 | AWS / 멀티 | 코드로 작성 후 CFn·Terraform 코드로 합성 |
| **Ansible** | 절차형 (구성 관리) | YAML | OS·앱 설정 | 이미 있는 서버 **안**을 설정, 에이전트리스 |

```
프로비저닝(Provisioning)          구성 관리(Configuration Management)
= 인프라를 '만든다'                = 만들어진 서버 '안'을 설정한다
Terraform, CloudFormation   →     Ansible, Chef, Puppet
```

> 둘은 경쟁 관계가 아니라 역할이 다르다. 실무에서는 Terraform으로 EC2를 띄우고 Ansible로 그 안을 설정하는 조합을 쓴다. 다만 컨테이너 시대에는 "안을 설정"하는 역할을 이미지(Dockerfile)가 대신하는 경우가 많다. → `../docker-labs/07-image/`

---

## Terraform의 핵심 개념 3가지

| 개념 | 의미 |
|---|---|
| **Provider** | AWS·GCP·Kubernetes 등 대상 플랫폼의 API를 다루는 플러그인 |
| **Resource** | 실제로 만들 대상 하나 (VPC, 서브넷, IAM 역할…) |
| **State** | "코드의 어떤 리소스가 실제 클라우드의 어떤 것인지"를 기록한 매핑 파일 |

```
   코드(.tf)  ─────────┐
                       ├──▶ Terraform ──▶ Provider ──▶ 클라우드 API
   state(.tfstate) ────┘        │
                                └─ 코드(원하는 상태) vs state(아는 상태) vs 실제(진짜 상태)
                                   세 개를 비교해 '차이(diff)'만 실행한다
```

> Terraform이 "무엇을 만들고 무엇을 지울지" 아는 이유는 **state가 있기 때문**이다. state가 IaC 학습에서 가장 중요한 주제다. → `03-state/`

---

## 핵심 워크플로

```
  작성        초기화        계획          적용         삭제
 write ──▶ terraform init ──▶ plan ──▶ apply ──▶ destroy
             │                 │          │
             │                 │          └─ 실제 클라우드에 반영, state 갱신
             │                 └─ 만들 것(+)·바꿀 것(~)·지울 것(-) 미리보기
             └─ 프로바이더·모듈 다운로드, 백엔드 연결
```

```bash
terraform init      # .terraform/ 에 프로바이더 설치, 백엔드 초기화
terraform fmt       # 코드 포맷 정리
terraform validate  # 문법·타입 검증 (클라우드 접근 없이)
terraform plan      # 변경 예정 사항 확인 ← 반드시 눈으로 읽는다
terraform apply     # 승인 후 적용
terraform destroy   # 관리 중인 리소스 전체 삭제
```

### plan 출력 읽는 법

```
Plan: 3 to add, 1 to change, 2 to destroy.

  + create              새로 만든다
  ~ update in-place     기존 리소스를 그대로 두고 속성만 바꾼다
-/+ destroy and then create (replace)   지웠다가 다시 만든다  ← 위험 신호
  - destroy             지운다
```

> `-/+ replace`는 리소스가 **한 번 사라졌다 생긴다**는 뜻이다. RDS나 EBS에 이게 뜨면 데이터가 날아간다. plan에서 replace가 보이면 무조건 이유를 먼저 확인한다.

---

## 설치와 첫 실행

```bash
# 설치 (Linux)
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
sudo apt update && sudo apt install terraform

terraform version
terraform -install-autocomplete    # 셸 자동완성
```

> 프로젝트마다 Terraform 버전이 다를 수 있으므로 **`tfenv`** 같은 버전 매니저를 쓰면 편하다. 팀 프로젝트라면 `required_version`으로 버전을 고정한다. → `02-provider-resource/`

### 최소 예제

```hcl
# main.tf
terraform {
  required_version = "~> 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "tf-lab-vpc"
  }
}
```

```bash
terraform init && terraform plan && terraform apply
```

---

## 디렉터리 구조 관례

Terraform은 **디렉터리 안의 모든 `.tf` 파일을 하나로 합쳐서** 읽는다. 파일 분리는 순전히 사람이 읽기 위한 것이다.

```
project/
├── main.tf           # 리소스 정의
├── variables.tf      # 입력 변수 선언
├── outputs.tf        # 출력값
├── providers.tf      # provider·terraform 블록
├── terraform.tfvars  # 변수 값 (환경별)   ← 시크릿이 들어가면 커밋 금지
└── .terraform.lock.hcl  # 프로바이더 버전 잠금 ← 반드시 커밋
```

### `.gitignore`

```gitignore
.terraform/           # 다운로드된 프로바이더·모듈 (용량이 크다)
*.tfstate             # state — 시크릿이 평문으로 들어있다
*.tfstate.backup
*.tfvars              # 값에 따라 시크릿 포함 (예제 파일만 .tfvars.example 로)
crash.log
```

> **`.terraform.lock.hcl`은 커밋한다.** `package-lock.json`과 같은 역할로, 팀원과 CI가 동일한 프로바이더 버전을 쓰게 만든다.  
> 반대로 **`*.tfstate`는 절대 커밋하지 않는다.** DB 비밀번호가 평문으로 들어있다. → `03-state/`

---

## 배운 점

- IaC의 핵심 가치는 자동화보다 **인프라를 리뷰·추적 가능한 코드로 만드는 것**
- 선언형은 "무엇을"만 적고 도달 방법은 도구가 정한다 → 여러 번 실행해도 같은 결과(**멱등성**)
- 프로비저닝(Terraform)과 구성 관리(Ansible)는 경쟁이 아니라 역할 분담
- Terraform의 3요소는 **Provider · Resource · State**
- Terraform은 **코드 / state / 실제 인프라** 세 가지를 비교해 차이만 실행한다
- 워크플로는 `init → plan → apply`, plan을 눈으로 읽는 것이 전부다
- plan에서 **`-/+ replace`** 가 보이면 리소스가 재생성된다는 뜻 — 데이터 손실 위험 신호
- 디렉터리 내 `.tf` 파일은 전부 합쳐져 읽히므로 파일 분리는 가독성 목적
- `.terraform.lock.hcl`은 커밋하고, `*.tfstate`와 시크릿이 든 `*.tfvars`는 커밋하지 않는다
