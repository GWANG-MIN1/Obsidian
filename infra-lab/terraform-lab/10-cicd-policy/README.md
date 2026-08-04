# 10 CI/CD & 정책 검사

로컬에서 `terraform apply`를 치는 동안에는 IaC의 절반만 쓰는 것이다.  
**사람의 노트북이 아니라 파이프라인이 apply할 때** 비로소 "누가 언제 무엇을 왜 바꿨는지"가 남고, 리뷰와 정책 검사가 강제된다.

```
   PR 생성 ──▶ fmt·validate·tfsec ──▶ plan ──▶ PR에 결과 코멘트 ──▶ 리뷰·승인
                                                                      │
   main 머지 ──────────────────────────────────────────────▶ apply ───┘
```

---

## 왜 로컬 apply를 그만둬야 하나

| 로컬 apply의 문제 | 파이프라인 apply |
|---|---|
| 누가 무엇을 바꿨는지 기록이 없다 | PR·커밋에 전부 남는다 |
| 리뷰 없이 prod가 바뀐다 | 승인 없이는 apply되지 않는다 |
| 각자 다른 Terraform 버전 | 버전이 고정된다 |
| 개인 노트북에 prod 자격증명 | OIDC 임시 자격증명, 장기 키 없음 |
| 보안 검사가 선택 사항 | 검사 실패 시 머지 자체가 막힌다 |

---

## 인증 — OIDC (장기 키 없이)

CI에 `AWS_ACCESS_KEY_ID`를 시크릿으로 넣는 방식은 **키가 유출되면 끝**이다. GitHub Actions는 OIDC로 임시 자격증명을 받을 수 있다.

```
GitHub Actions ──(OIDC 토큰)──▶ AWS IAM ──▶ STS AssumeRoleWithWebIdentity
                                                    │
                                                    ▼
                                            임시 자격증명 (1시간)
```

```hcl
# IAM — GitHub Actions 가 맡을 역할
data "aws_iam_policy_document" "gha_assume" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    # 특정 저장소·브랜치로 제한한다 (이 조건이 없으면 아무 저장소나 맡을 수 있다)
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:GWANG-MIN1/infra:ref:refs/heads/main"]
    }
  }
}
```

> IRSA와 정확히 같은 구조다 — **OIDC 토큰을 믿되, `sub` 조건으로 범위를 좁힌다.** → `09-aws-vpc-eks/`  
> `sub` 조건을 `repo:org/*`처럼 느슨하게 두면 다른 저장소에서도 이 역할을 맡을 수 있다.

---

## GitHub Actions — PR에서 plan, 머지에서 apply

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths: ["terraform/**"]
  push:
    branches: [main]
    paths: ["terraform/**"]

permissions:
  contents: read
  id-token: write        # OIDC 토큰 발급에 필수
  pull-requests: write   # PR 코멘트

env:
  TF_IN_AUTOMATION: "1"
  TF_INPUT: "0"

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: terraform/environments/dev

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.9.5      # 버전 고정

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
          aws-region: ap-northeast-2

      - name: Format check
        run: terraform fmt -check -recursive -diff

      - name: Init
        run: terraform init -backend-config=backend.hcl

      - name: Validate
        run: terraform validate

      - name: Plan
        id: plan
        run: terraform plan -no-color -out=tfplan

      - name: Comment plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

### 핵심 포인트

| | 이유 |
|---|---|
| `plan -out=tfplan` → `apply tfplan` | **검토한 그 계획을 그대로 적용**한다. `apply`를 새로 돌리면 그 사이 변경이 섞인다 |
| `TF_INPUT=0` | 대화형 입력을 막아 파이프라인이 멈추지 않게 |
| `id-token: write` | 없으면 OIDC 토큰 발급이 안 된다 |
| `paths:` 필터 | 무관한 변경에 파이프라인이 돌지 않게 |
| prod는 environment 승인 | GitHub Environments의 required reviewers로 게이트 |

> plan 출력을 PR에 코멘트하면 **인프라 변경이 코드 리뷰 대상**이 된다. 이게 IaC를 CI에 올리는 가장 큰 이유다.  
> plan 결과에 시크릿이 섞일 수 있다. 퍼블릭 저장소라면 코멘트 대신 아티팩트로 남기는 편이 안전하다.

---

## 정적 분석 — 잘못된 설정을 머지 전에 잡기

`terraform validate`는 문법만 본다. **"보안그룹이 0.0.0.0/0으로 열려 있다"** 같은 건 못 잡는다.

| 도구 | 특징 |
|---|---|
| **tfsec** | Terraform 전용, 빠르다. Trivy에 통합됨 |
| **Checkov** | 규칙이 가장 많다. Terraform 외에 K8s·Dockerfile도 검사 |
| **Trivy (config)** | 컨테이너 스캐너와 같은 도구로 IaC까지 (도구 통일에 유리) |
| **tflint** | 보안이 아니라 린트 — 존재하지 않는 인스턴스 타입 등 |

```bash
tfsec .                          # 취약 설정 탐지
checkov -d . --quiet             # 정책 위반
trivy config .                   # 미스컨피그
tflint --recursive               # 프로바이더별 린트
```

```yaml
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false        # 발견되면 파이프라인 실패

      - name: Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform/
          framework: terraform
          soft_fail: false
```

### 잡히는 것들

```
CRITICAL  보안그룹 인그레스가 0.0.0.0/0 에 열림
HIGH      S3 버킷 암호화 미설정
HIGH      RDS 퍼블릭 액세스 활성화
MEDIUM    EBS 볼륨 미암호화
MEDIUM    CloudTrail 로그 검증 비활성화
LOW       태그 누락
```

> 처음 돌리면 수십 개가 쏟아진다. **전부 고치려 하지 말고, 예외를 명시적으로 남기며 CRITICAL·HIGH부터 잡는다.**

```hcl
#tfsec:ignore:aws-ec2-no-public-ingress-sgr  # ALB는 공개가 의도된 구성
resource "aws_security_group_rule" "alb_http" {
  cidr_blocks = ["0.0.0.0/0"]
}
```

> 예외에는 **반드시 이유를 적는다.** 이유 없는 ignore는 그냥 무시와 같다.

---

## Policy as Code — 조직 규칙을 코드로

정적 분석이 일반적인 보안 규칙이라면, Policy as Code는 **우리 조직만의 규칙**을 강제한다.

```rego
# OPA/Conftest — "모든 리소스에 Owner 태그가 있어야 한다"
package terraform.tags

deny[msg] {
  resource := input.resource_changes[_]
  resource.change.actions[_] == "create"
  not resource.change.after.tags.Owner
  msg := sprintf("%s 에 Owner 태그가 없습니다", [resource.address])
}
```

```bash
terraform show -json tfplan > plan.json
conftest test plan.json --policy policy/
```

| 도구 | 비고 |
|---|---|
| **OPA / Conftest** | 오픈소스, Rego 언어. plan JSON을 검사 |
| **Sentinel** | Terraform Cloud/Enterprise 전용 |
| **Kyverno** | Kubernetes 리소스용 — 클러스터 쪽 대응물 |

> Kyverno가 클러스터에서 하는 일(잘못된 파드 차단)을 **Conftest가 인프라에서** 한다. `plan`을 JSON으로 뽑아 검사하므로 **apply 전에** 막을 수 있다.

---

## Atlantis — PR에서 apply까지

GitHub Actions로 직접 짜는 대신, Terraform PR 워크플로에 특화된 도구를 쓸 수도 있다.

```
PR 생성 ──▶ Atlantis가 자동으로 plan 후 코멘트
개발자: atlantis apply    ← PR 코멘트로 명령
Atlantis ──▶ apply 후 결과 코멘트 ──▶ 머지
```

| 장점 | 단점 |
|---|---|
| PR 코멘트로 plan/apply 제어 | 서버를 직접 운영해야 한다 |
| 디렉터리별 자동 감지·잠금 | Actions로도 대부분 흉내낼 수 있다 |
| 승인 없이는 apply 차단 | |

> 소규모라면 GitHub Actions로 충분하다. **스택과 팀이 늘어 PR마다 어느 디렉터리를 apply할지 관리가 어려워질 때** 검토한다.

---

## 파이프라인 체크리스트

```
□ terraform fmt -check      포맷 강제
□ terraform validate        문법 검증
□ tflint                    잘못된 값 검출
□ tfsec / checkov           보안 정책
□ conftest                  조직 규칙
□ terraform plan -out       계획 생성 + PR 코멘트
□ (승인 게이트)              prod는 required reviewers
□ terraform apply tfplan    검토한 그 계획을 적용
□ 상태 알림                  실패 시 Slack 통보
```

> 앞의 5개는 **자격증명 없이도 돌릴 수 있다** (`terraform init -backend=false`). 포크 PR에서도 안전하게 검사할 수 있다.

---

## 배운 점

- 로컬 apply를 그만두는 이유는 자동화가 아니라 **리뷰·이력·권한 통제**를 얻기 위해서다
- CI 인증은 장기 액세스 키가 아니라 **OIDC 임시 자격증명** — IRSA와 같은 구조
- OIDC 신뢰 정책의 **`sub` 조건으로 저장소·브랜치를 좁혀야** 한다
- `permissions: id-token: write`가 없으면 OIDC 토큰이 발급되지 않는다
- **`plan -out=tfplan` → `apply tfplan`** — 검토한 그 계획을 그대로 적용한다
- `TF_INPUT=0`, `TF_IN_AUTOMATION=1`로 파이프라인이 입력 대기에 멈추지 않게 한다
- plan 결과를 PR에 코멘트하면 **인프라 변경이 코드 리뷰 대상**이 된다 (시크릿 노출은 주의)
- `terraform validate`는 문법만 본다 — 보안 설정은 **tfsec·checkov**가 잡는다
- 처음엔 경고가 쏟아지므로 CRITICAL·HIGH부터, 예외는 **이유와 함께** 명시
- 조직 고유 규칙은 **OPA/Conftest로 plan JSON을 검사** — Kyverno의 인프라 버전
- fmt·validate·lint·보안검사는 **자격증명 없이** 돌릴 수 있다 (`-backend=false`)
