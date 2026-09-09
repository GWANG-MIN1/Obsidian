# 04. 매 push 마다 배포 잡이 빨간불 — 두 번 고쳤다

- **발생**: 2026-07-11 · 커밋 `a8d831f` (23:22) → `0b869d9` (23:28)
- **증상 한 줄**: 문서 오타를 고쳐도 CI 가 빨간불. 원인은 워크플로가 아니라 AWS 키였다

---

## 현상

`main` 에 push 하면 `deploy` 잡이 항상 실패했다. 실패 지점은 항상 같다.

```
Configure AWS credentials
  Error: The security token included in the request is invalid
```

**코드와 무관하게** 빨간불이다. README 한 줄을 고쳐도 그렇다.

## 환경

- GitHub Actions, `on: push [main]` → `test` → `deploy`(`terraform init/validate/plan/apply`)
- `deploy` 는 `aws-actions/configure-aws-credentials@v4` 로 시작

## 진단

### 1) 첫 가설 — 시크릿이 없어서 → **부분적으로만 맞았다**

`configure-aws-credentials` 는 키가 **비어 있으면 즉시 실패**한다.
저장소에 시크릿을 안 넣었으면 워크플로 전체가 빨간불이 되는 게 맞다.

그래서 **가드 잡**을 넣었다 (`a8d831f`, 23:22).

```yaml
check-secrets:
  outputs:
    aws_ready: ${{ steps.check.outputs.aws_ready }}
  steps:
    - run: |
        if [ -n "$AWS_ACCESS_KEY_ID" ] && [ -n "$AWS_SECRET_ACCESS_KEY" ]; then
          echo "aws_ready=true"  >> "$GITHUB_OUTPUT"
        else
          echo "aws_ready=false" >> "$GITHUB_OUTPUT"
          echo "::notice::AWS 시크릿이 없어 배포를 건너뜁니다."
        fi

deploy:
  needs: [test, check-secrets]
  if: needs.check-secrets.outputs.aws_ready == 'true'
```

**시크릿이 없으면 실패가 아니라 skip.** 논리적으로 맞는 수정이다.

### 2) 그런데 여전히 빨간불이었다 — 6분 뒤

시크릿이 **없는 게 아니라 있는데 무효**였다. 값이 비어 있지 않으니
`check-secrets` 는 `aws_ready=true` 를 내고, `deploy` 가 돌고, AWS 가 거절한다.

```
The security token included in the request is invalid
```

이건 **"키가 없다"가 아니라 "키가 만료/무효다"** 라는 뜻이다.
빈 문자열 검사로는 절대 못 잡는다. **유효성은 AWS 만 안다.**

### 3) 그럼 유효한 키를 넣으면 되지 않나

그게 근본 해결이지만, 그때 **넣을 유효한 키가 없었다.**
그리고 키를 새로 발급해 넣어도 문제가 하나 남는다 —
**문서만 고쳐도 `terraform apply` 가 도는 구조**는 그대로다.

## 원인

두 가지가 겹쳐 있었다.

1. 저장소 Secrets 의 AWS 액세스 키가 **만료/무효**
2. **비용이 드는 배포가 `push` 라는 일상적 행위에 묶여 있었다**

첫 번째만 고치면 두 번째가 남고, 두 번째를 고치면 첫 번째는 급하지 않게 된다.

## 해결

배포를 **수동 트리거로** 옮겼다 (`0b869d9`, 23:28).

```yaml
on:
  push:         { branches: [main] }   # test 잡만
  pull_request: { branches: [main] }   # test 잡만
  workflow_dispatch:                   # ← 배포는 여기서만

check-secrets:
  if: github.event_name == 'workflow_dispatch'
```

`Terraform Apply` 의 가드도 `push` 전제에서 **`main` 브랜치 전제**로 바꿨다
(`if: github.ref == 'refs/heads/main'`) — 트리거가 바뀌었으니 조건도 같이 바뀌어야 한다.

결과적으로

- **push/PR** = 품질 게이트만 (ruff · pytest · terraform fmt · tfsec). **AWS 자격증명이 필요 없다.**
- **수동 실행** = 배포. 시크릿이 비어 있으면 실패가 아니라 skip.

그리고 저장소 README 에 **키가 만료되면 무엇을 해야 하는지**를 적었다.

> ⚠️ 수동 배포가 `The security token ... is invalid` 로 실패하면 저장된 키가
> 만료/무효한 것이니 레포 Secrets 에서 새 IAM 키로 교체하세요.

→ [결정 08](../decisions/08-배포를-자동-push에서-수동-트리거로.md)

## 재발 방지

- **자격증명이 필요한 잡과 필요 없는 잡을 나눈다.** 지금 품질 게이트는 계정 없이도 돈다.
- **skip 과 fail 을 구분한다.** `::notice::` 로 이유를 남겨서, 초록불인데 배포가 안 된 것을
  나중에 "왜 안 올라갔지"로 헷갈리지 않게 했다.
- 근본 해결은 **OIDC 키리스 배포**다. 안 쓴 이유는 [결정 08](../decisions/08-배포를-자동-push에서-수동-트리거로.md) 에 적었다 —
  기술적 판단이 아니라 **다른 저장소와 주제를 겹치지 않게 한다는 학습 계획** 때문이다.

## 배운 점

**"값이 있다"와 "값이 유효하다"는 다르다.** 빈 문자열 검사는 전자만 본다.
후자는 실제로 써 봐야 알고, 그래서 **가드로는 못 막는다.**

**첫 수정이 6분 만에 부족하다고 판명된 것이 이 기록의 값이다.**
가설(시크릿이 없다)이 그럴듯했고, 그 가설에 맞는 수정을 정확히 했는데, **상태가 달랐다.**
[gym-management-db 트러블 03](../../gym-management-db/troubleshooting/03-GET-members-p95-2초.md) 의
*"그럴듯한 원인을 검증 없이 적으면 그게 기록으로 남는다"* 와 같은 종류인데,
이번엔 **다음 커밋에서 바로 뒤집혀서** 기록에 4개월이 아니라 6분만 남았다.

**빨간불이 일상이 되면 진짜 실패를 안 보게 된다.**
[gym-management-db 트러블 08](../../gym-management-db/troubleshooting/08-린트를-우회했다가-검증이-무력화.md) 은
`|| true` 로 빨간불을 없앴고, 여기선 **잡을 나눠서** 없앴다. 겉보기 결과는 같은 초록불이지만
한쪽은 검증을 껐고 한쪽은 검증의 범위를 정한 것이다.
