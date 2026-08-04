# 03 State

**State는 "코드의 리소스 ↔ 실제 클라우드 리소스"를 잇는 매핑 파일이다.**  
Terraform이 무엇을 만들고·바꾸고·지울지 아는 유일한 근거이고, 동시에 Terraform 운영에서 사고가 가장 많이 나는 지점이다.

---

## state가 없으면 생기는 일

```
코드: aws_vpc.main (cidr 10.0.0.0/16)
실제: vpc-0abc123, vpc-0def456 ... 둘 중 어느 게 내 것인지?
```

state가 있으면:

```
state: aws_vpc.main  →  vpc-0abc123
       aws_subnet.public → subnet-0xyz789
```

Terraform은 apply 때마다 세 가지를 비교한다.

```
   코드 (원하는 상태)          state (알고 있는 상태)         실제 (진짜 상태)
        │                            │                          │
        └────────────────────────────┴──────────────────────────┘
                              차이(diff)만 실행
```

| 역할 | 설명 |
|---|---|
| **매핑** | 코드 주소(`aws_vpc.main`)와 실제 ID(`vpc-0abc123`)를 연결 |
| **메타데이터** | 리소스 의존 관계를 저장해 삭제 순서를 결정 |
| **성능** | 대규모에서 매번 전체를 조회하지 않도록 캐시 역할 |
| **협업** | 원격 백엔드에 두면 팀 전체가 같은 상태를 공유 |

---

## tfstate 파일 뜯어보기

```json
{
  "version": 4,
  "terraform_version": "1.9.5",
  "serial": 12,
  "lineage": "3f8a...",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_vpc",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "vpc-0abc123",
            "cidr_block": "10.0.0.0/16"
          }
        }
      ]
    }
  ]
}
```

> **`serial`은 변경할 때마다 증가하고, `lineage`는 state의 혈통 ID다.** 서로 다른 lineage의 state를 밀어 넣으려 하면 Terraform이 막는다.  
> 이 파일은 **직접 손으로 수정하지 않는다.** 필요하면 `terraform state` 명령을 쓴다.

### ⚠️ state에는 시크릿이 평문으로 들어간다

```json
"attributes": {
  "password": "supersecret123",        // RDS 비밀번호가 그대로
  "master_password": "..."
}
```

- `sensitive = true`는 **CLI 출력만 가린다.** state 파일 안에는 평문으로 저장된다
- 그래서 `*.tfstate`는 절대 Git에 올리지 않고, 원격 백엔드에서 **암호화 + 접근 제어**를 건다

---

## 로컬 state의 한계

```
개발자 A ── terraform.tfstate (내 노트북)
개발자 B ── terraform.tfstate (네 노트북)   ← 서로 모른다 → 같은 리소스를 중복 생성
```

| 문제 | 결과 |
|---|---|
| 공유 불가 | 팀원이 각자 다른 state를 갖는다 |
| 잠금 없음 | 동시에 apply하면 state가 깨진다 |
| 유실 위험 | 노트북이 죽으면 인프라 관리 권한을 잃는다 |
| 시크릿 노출 | 평문 파일이 로컬에 굴러다닌다 |

**혼자 하는 실습이 아니면 무조건 원격 백엔드를 쓴다.**

---

## 원격 백엔드 — S3

```hcl
terraform {
  backend "s3" {
    bucket       = "my-tfstate-bucket"
    key          = "infra-lab/dev/terraform.tfstate"   # 버킷 안 경로 — 환경별로 나눈다
    region       = "ap-northeast-2"
    encrypt      = true                                # 저장 시 암호화
    use_lockfile = true                                # S3 네이티브 잠금 (Terraform 1.10+)
  }
}
```

### 잠금(locking)이 필요한 이유

```
개발자 A: terraform apply ──┐
                            ├─▶ 동시에 같은 state를 쓰면 → state 손상
개발자 B: terraform apply ──┘
```

| 방식 | 설명 |
|---|---|
| **S3 네이티브 잠금** | `use_lockfile = true` — S3에 `.tflock` 객체를 만들어 잠근다 (1.10+, **권장**) |
| **DynamoDB 잠금** | `dynamodb_table = "tf-lock"` — 오래된 표준 방식, 1.11부터 사용 중단 예정 |

```hcl
# 레거시 방식 (기존 프로젝트에서 자주 보게 된다)
terraform {
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "dev/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"     # LockID 를 파티션 키로 갖는 테이블
  }
}
```

### 백엔드용 버킷 준비

```bash
aws s3api create-bucket --bucket my-tfstate-bucket \
  --region ap-northeast-2 \
  --create-bucket-configuration LocationConstraint=ap-northeast-2

aws s3api put-bucket-versioning --bucket my-tfstate-bucket \
  --versioning-configuration Status=Enabled        # 실수로 깨뜨렸을 때 되돌릴 수 있다

aws s3api put-public-access-block --bucket my-tfstate-bucket \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

> **버전 관리는 반드시 켠다.** state가 깨졌을 때 이전 버전으로 되돌리는 것이 유일한 복구 수단이다.  
> 백엔드 블록에는 변수를 쓸 수 없다. 환경별로 다르게 하려면 `terraform init -backend-config=dev.hcl`로 주입한다. → `08-workspace-environment/`

### 백엔드 전환

```bash
terraform init -migrate-state     # 로컬 → 원격으로 state 이전
terraform init -reconfigure       # 이전 없이 백엔드 설정만 교체
```

---

## state 조회 명령

```bash
terraform state list                    # 관리 중인 리소스 주소 목록
terraform state show aws_vpc.main       # 특정 리소스의 모든 속성
terraform show                          # 전체 state를 읽기 좋게
terraform show -json | jq '.values'     # JSON 으로 (스크립트 처리용)
terraform state pull > backup.json      # 원격 state 내려받기 (백업)
```

---

## state 조작 명령 (위험, 반드시 백업 후)

```bash
# 1) 리네임 — 코드에서 이름만 바꿨을 때 destroy/create 를 피한다
terraform state mv aws_vpc.old aws_vpc.new
terraform state mv aws_vpc.main module.network.aws_vpc.main   # 모듈로 옮길 때

# 2) 관리 대상에서 제외 — 실제 리소스는 그대로 두고 state에서만 뺀다
terraform state rm aws_instance.legacy

# 3) 잠금이 걸린 채 프로세스가 죽었을 때
terraform force-unlock <LOCK_ID>
```

> `state rm`은 **리소스를 삭제하지 않는다.** Terraform이 그 리소스를 "모르는 상태"로 만들 뿐이다. 그래서 관리에서 떼어낼 때 쓴다.  
> 반대로 코드에서만 지우고 `apply`하면 **실제 리소스가 삭제된다.** 이 둘을 헷갈리면 사고가 난다. (의도적으로 떼어내려면 `removed` 블록 → `07-data-import/`)

```
코드에서 삭제 + apply   →  실제 리소스도 삭제됨   💥
terraform state rm      →  실제 리소스는 살아있음, 관리만 중단
```

---

## 드리프트 (Drift)

**드리프트 = 코드와 실제 인프라가 어긋난 상태.** 누군가 콘솔에서 손으로 바꾸면 발생한다.

```bash
terraform plan            # 리프레시 후 차이를 보여준다 → 드리프트 감지
```

```
Note: Objects have changed outside of Terraform
  ~ aws_security_group.web
      ~ ingress = [ + { cidr_blocks = ["0.0.0.0/0"] ... } ]
```

대응은 두 갈래다.

| 상황 | 조치 | 결과 |
|---|---|---|
| 수동 변경이 **잘못된 것** | `terraform apply` | 코드 기준으로 되돌린다 |
| 수동 변경이 **맞는 것** | 코드를 고치고 `apply`, 또는 `terraform apply -refresh-only` | 실제 상태를 인정한다 |

```bash
terraform plan -refresh-only     # 실제 변경만 보여준다 (코드 변경은 제외)
terraform apply -refresh-only    # 실제 상태를 state에 반영
terraform plan -refresh=false    # 리프레시 생략 (리소스가 많아 느릴 때)
```

> 드리프트를 없애는 근본 해법은 **콘솔 쓰기 권한을 회수하는 것**이다. ArgoCD의 `selfHeal`이 K8s에서 하는 일과 같은 문제의식이다. → `../k8s-manifests/10-helm-gitops/`

---

## 운영 원칙

| 원칙 | 이유 |
|---|---|
| state는 원격 + 암호화 + 버전관리 | 유실·유출 방지, 복구 가능 |
| state 파일을 직접 편집하지 않는다 | `serial`·`lineage`가 깨지면 복구가 어렵다 |
| 환경별로 state를 분리한다 | dev 실수가 prod에 닿지 않게 |
| state 접근 권한을 최소화한다 | state = 인프라 전체에 대한 지도 + 시크릿 |
| 조작 전에 `terraform state pull`로 백업 | 되돌릴 수단을 먼저 확보 |

---

## 배운 점

- state는 **코드 주소 ↔ 실제 리소스 ID 매핑** — Terraform이 diff를 계산하는 유일한 근거
- state에는 **DB 비밀번호 같은 시크릿이 평문**으로 저장된다. `sensitive = true`는 출력만 가린다
- 로컬 state는 공유·잠금·복구가 안 되므로 실습 외에는 **원격 백엔드(S3) 필수**
- 잠금은 Terraform 1.10+의 `use_lockfile = true`(S3 네이티브)가 권장, DynamoDB 방식은 레거시
- 백엔드 버킷은 **버전 관리를 반드시 켠다** — state 손상 시 유일한 복구 수단
- 백엔드 블록에는 변수를 못 쓴다 → `-backend-config`로 주입
- **`state rm`은 리소스를 지우지 않는다**(관리만 중단), 코드에서 지우고 apply하면 진짜 지워진다
- 이름만 바꿀 때 `state mv`(또는 `moved` 블록)를 쓰면 destroy/create를 피할 수 있다
- 드리프트는 콘솔 수동 변경에서 온다 — `apply`로 되돌리거나 `-refresh-only`로 인정하거나
- 근본 해법은 콘솔 쓰기 권한 회수. GitOps의 셀프힐과 같은 발상
