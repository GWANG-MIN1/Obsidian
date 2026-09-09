# 08. 배포를 push 자동 실행에서 수동 트리거로

> 메모: `main` 에 push 할 때마다 `terraform apply` 가 돌게 해 뒀는데,
> AWS 키가 만료된 뒤로 **모든 push 가 빨간불**이었다. 2026-07-11 커밋 `a8d831f` · `0b869d9`

---

## 상황

1단계에서 만든 워크플로우는 이랬다.

```yaml
on:
  push:
    branches: [main]
jobs:
  test:    ...
  deploy:  needs: test   # terraform init → validate → plan → apply
```

문제가 둘.

1. **AWS 자격증명이 유효하지 않으면 매 push 가 실패한다.** 문서 오타를 고쳐도 빨간불이다.
   `The security token included in the request is invalid`.
2. **문서만 고쳐도 `terraform apply` 가 돈다.** 실행 자체가 비용이고, 의도치 않은
   인프라 변경이 섞여 들어갈 자리다.

빨간불이 일상이 되면 **진짜 실패를 안 보게 된다.** [gym-management-db 트러블 08](../../gym-management-db/troubleshooting/08-린트를-우회했다가-검증이-무력화.md)
에서 `|| true` 로 CI 를 무력화한 채 4개월을 간 것과 **같은 함정의 다른 입구**다.

## 결정

**두 단계로** 나눠서 처리했다. 커밋 순서가 그대로 사고 과정이다.

**1) 시크릿이 없으면 실패가 아니라 skip** (`a8d831f`)

```yaml
check-secrets:
  outputs:
    aws_ready: ${{ steps.check.outputs.aws_ready }}
  steps:
    - run: |
        if [ -n "$AWS_ACCESS_KEY_ID" ] && [ -n "$AWS_SECRET_ACCESS_KEY" ]; then
          echo "aws_ready=true" >> "$GITHUB_OUTPUT"
        else
          echo "aws_ready=false" >> "$GITHUB_OUTPUT"
          echo "::notice::AWS 시크릿이 없어 배포를 건너뜁니다."
        fi

deploy:
  needs: [test, check-secrets]
  if: needs.check-secrets.outputs.aws_ready == 'true'
```

**2) 배포를 아예 수동 트리거로** (`0b869d9`)

```yaml
on:
  push:         [main]      # test 잡만 돈다
  pull_request: [main]      # test 잡만 돈다
  workflow_dispatch:        # ← 배포는 여기서만
```

```yaml
check-secrets:
  if: github.event_name == 'workflow_dispatch'
```

결과적으로 **push/PR = 품질 게이트, 수동 실행 = 배포**로 갈렸다.
품질 게이트(ruff · pytest · terraform fmt · tfsec)는 **AWS 자격증명이 필요 없다.**

## 왜

- **자격증명 없이도 CI 가 의미 있게 돌아야 한다.** 이 저장소를 클론한 사람은 AWS 계정이 없다.
  테스트와 린트는 그래도 돌아야 하고, 그게 초록불이면 "이 코드는 통과했다"가 성립한다.
- **비용이 드는 동작은 명시적 의도로만.** `terraform apply` 는 되돌리기 어렵고 과금된다.
  push 라는 **일상적 행위**에 묶어 두면 안 된다.
- **skip 과 fail 을 구분한다.** "할 수 없었다"와 "하려다 실패했다"는 다른 사건이다.
  `::notice::` 로 이유를 로그에 남겨서, 초록불인데 배포가 안 된 것을 나중에 헷갈리지 않게 했다.
- 저장소 README 에 **왜 수동인지와, 토큰 만료 시 무엇을 해야 하는지**를 적었다.
  이 판단은 코드만 봐선 "CD 를 안 만들었네"로 읽히기 때문이다.

## 왜 다른 건 안 썼나

**GitHub OIDC 로 키리스 배포**

**정답에 가깝다.** 액세스 키 자체가 없어지므로 만료 문제도 없다.
[aws-serverless-agent 결정 13](../../aws-serverless-agent/decisions/13-배포를-GitHub-OIDC로.md) 에서
실제로 이걸 했고, [트러블 13](../../aws-serverless-agent/troubleshooting/13-키리스-배포가-자격증명을-못-읽는다.md) 에서
그 대가도 치렀다.

여기서 안 쓴 이유는 **`docs/upgrades/README.md` 에 적어 둔 로드맵 원칙** 때문이다.

> 다른 레포(`aws-serverless-agent`)와 겹치지 않는 주제로만 선정했습니다.
> (OIDC·X-Ray·CloudWatch 대시보드·CloudFront 등은 그쪽에서 이미 다뤘으므로 제외)

**학습 가치가 없어서 뺀 것이지, 이게 더 나은 방식이라서가 아니다.**
운영이라면 OIDC 를 써야 한다. 이 저장소는 그 자리를 비워 뒀다고 적어 두는 게 정직하다.

**키를 갱신하고 자동 배포를 유지한다**

증상만 없앤다. 키는 또 만료되고, 문서 수정에 `apply` 가 도는 문제는 그대로다.

**deploy 잡에 `continue-on-error: true`**

빨간불은 없어지고 **배포 실패도 안 보이게** 된다. `|| true` 와 같은 종류의 해법이라
의식적으로 피했다.

## 결과

- push/PR 은 자격증명 없이 초록불이 뜬다. **이제 빨간불은 진짜 실패다.**
- 배포는 Actions → Deploy → Run workflow.
- 저장소 README 는 **평소 배포는 로컬 `terraform apply` 를 권장**한다고 적었다 —
  실제 운영 방식과 문서가 어긋나지 않게.

## 배운 점

**CI 의 빨간불에는 하나의 뜻만 있어야 한다.** "코드가 틀렸다"와 "환경이 준비 안 됐다"가
같은 색으로 나오면, 사람은 곧 그 색을 무시한다.
