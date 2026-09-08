# GitHub OIDC로 CDK 배포하기

> 메모: OIDC 기본기는 cicd-lab/03에 있다. 여기는 CDK 배포에 붙일 때 생기는 것 — 닭-달걀, provider 중복, 권한 위임 구조

---

## 개념

GitHub Actions OIDC의 기본 메커니즘(`permissions: id-token: write`, 신뢰 정책 `sub` 조건,
Environments 승인 게이트)은 → [`../cicd-lab/03-github-actions-advanced/`](../cicd-lab/03-github-actions-advanced/)

여기는 **그걸 CDK 배포에 붙일 때만 생기는 것**을 적는다.

### 닭-달걀 — 배포 역할을 어디에 두나

배포 역할을 배포 대상 스택 안에 두면 순환이 생긴다.

```
CI가 배포하려면  →  배포 역할이 필요하고
배포 역할이 생기려면  →  스택이 배포돼야 하고
스택이 배포되려면  →  CI가 배포해야 하고 …
```

**정체성(누구인가)과 배포 대상(무엇을 만드는가)을 다른 스택으로 가른다.**

```
PipelineStack   OIDC provider + 배포 역할     ← 1회 수동 배포 (로컬에서)
AppStack        애플리케이션 본체              ← 그다음부터 CI가 배포
```

정체성은 한 번 손으로 만들고, 대상은 그 뒤로 자동이다.
이건 CDK만의 문제가 아니라 **CI가 자기 인프라를 만들 때 항상 나오는 구조**다.

### 권한을 CI에 직접 주지 않는다

CDK는 부트스트랩할 때 계정에 역할 다섯 개를 만들어 둔다
(`cdk-hnb659fds-deploy-role-*`, `-file-publishing-role-*` 등).
**실제 배포 권한은 이 부트스트랩 역할이 갖는다.**

그래서 CI 역할에는 "그 역할들을 빌릴 권한"만 주면 된다.

```json
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::<account>:role/cdk-*"
}
```

관리자 정책을 붙이는 것과 결과는 같지만, **워크플로가 계정 전체 권한을 갖지 않는다.**
권한이 한 단계 위임되는 구조다.

### OIDC provider는 계정에 하나뿐이다

```
EntityAlreadyExists: ...OIDC provider
```

`token.actions.githubusercontent.com` provider는 **계정당 하나**다.
다른 프로젝트에서 이미 만들었으면 새로 만들 게 아니라 **import**해야 한다.

```ts
const provider = props.oidcProviderArn
  ? iam.OpenIdConnectProvider.fromOpenIdConnectProviderArn(this, 'Gh', props.oidcProviderArn)
  : new iam.OpenIdConnectProvider(this, 'Gh', {
      url: 'https://token.actions.githubusercontent.com',
      clientIds: ['sts.amazonaws.com'],
    });
```

```bash
npx cdk deploy -c oidcProviderArn=arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com
```

### 워크플로 파일은 저장소 루트에만

```
.github/workflows/deploy.yml          ✅ 인식된다
infra/.github/workflows/deploy.yml    ❌ 무시된다 — 에러도 경고도 없다
```

모노레포나 폴더별 프로젝트 구조를 쓰면 **"이 프로젝트 폴더 안에 넣는" 습관**이 나오는데,
CI는 저장소 단위지 폴더 단위가 아니다.
Actions 탭에 실행이 아예 안 뜨면 이걸 먼저 본다.

## 실행

```yaml
# .github/workflows/deploy.yml
name: deploy

on:
  push:            # 검증만
  pull_request:
  workflow_dispatch:   # 배포는 수동

permissions:
  id-token: write      # ← 없으면 "Credentials could not be loaded"
  contents: read

jobs:
  synth:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: npm }
      - run: npm ci
      - run: npx cdk synth          # push/PR 은 여기까지

  deploy:
    if: github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1
          # 액세스 키 입력 자체가 없다
      - run: npm ci
      - run: npx cdk deploy --require-approval never
```

신뢰 정책은 저장소로 한정한다. 안 좁히면 **아무 레포나 이 역할을 빌린다.**

```ts
new iam.Role(this, 'DeployRole', {
  assumedBy: new iam.WebIdentityPrincipal(provider.openIdConnectProviderArn, {
    StringEquals: { 'token.actions.githubusercontent.com:aud': 'sts.amazonaws.com' },
    StringLike:   { 'token.actions.githubusercontent.com:sub': 'repo:OWNER/REPO:*' },
    //                                          브랜치까지 → 'repo:OWNER/REPO:ref:refs/heads/main'
  }),
});
```

> `sub`를 안 좁히는 건 **에러가 안 나는 종류**다. 동작은 하는데 위험하다.
> 리뷰가 없으면 그대로 남는다.

### 선행 조건

```bash
npx cdk bootstrap aws://<account>/<region>     # 계정·리전마다 1회
npx cdk deploy PipelineStack                   # 로컬에서 1회 (정체성 만들기)
```

이 둘이 안 되어 있으면 deploy job이 bootstrap 에러를 낸다.

## 배운 점

**"안 돈다"와 "돌다가 실패한다"는 다른 문제다.** Actions 탭에 실행이 안 뜨면
워크플로가 인식되지 않은 것이지 코드나 권한 문제가 아니다.
**아무 로그도 없다는 게 가장 큰 단서**인데 그걸 읽기까지 시간이 걸렸다.

**에러 메시지가 실패 지점보다 뒤를 가리킬 수 있다.**
`Credentials could not be loaded`를 보면 역할 ARN이나 신뢰 정책을 의심하게 되는데,
`permissions: id-token: write`가 없어서 **토큰이 애초에 발급되지 않은** 경우가 있다.
기본값이 `none`이라 명시하지 않으면 조용히 안 된다.

**CI가 자기 인프라를 만들 때는 정체성을 먼저 분리한다.** 닭-달걀은 CDK 고유 문제가 아니라
"파이프라인이 파이프라인을 만드는" 모든 경우에 나온다. 부트스트랩 단계는 손으로 한 번 하는 게 맞다.

**권한은 주는 것보다 위임하는 게 좁다.** CI에 배포 권한을 직접 주는 대신
"부트스트랩 역할을 빌릴 권한"만 주면, 실제 권한의 정의는 CDK 쪽에 남는다.

---

- 처음 만난 곳: [aws-serverless-agent](../../projects/aws-serverless-agent/README.md) —
  [결정 13](../../projects/aws-serverless-agent/decisions/13-배포를-GitHub-OIDC로.md)
- 진단 과정: [트러블 13](../../projects/aws-serverless-agent/troubleshooting/13-키리스-배포가-자격증명을-못-읽는다.md)
- OIDC 기본 메커니즘: [`../cicd-lab/03-github-actions-advanced/`](../cicd-lab/03-github-actions-advanced/)
- Terraform 쪽 같은 패턴: [`../terraform-lab/10-cicd-policy/`](../terraform-lab/10-cicd-policy/)
