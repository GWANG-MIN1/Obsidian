# 03 GitHub Actions 실전

기초 문법을 알면 워크플로는 쓸 수 있다. 실전에서 갈리는 건 **중복을 어떻게 줄이고, 자격증명을 어떻게 다루고, 무엇으로 빌드를 깨뜨릴지** 세 가지다.

---

## Matrix — 조합 실행

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false          # 하나 실패해도 나머지 계속 (조합별 결과를 보려면 필수)
      max-parallel: 4
      matrix:
        os: [ubuntu-latest, macos-latest]
        node: [18, 20, 22]
        include:                # 특정 조합에 값 추가
          - os: ubuntu-latest
            node: 22
            coverage: true
        exclude:                # 조합 제외
          - os: macos-latest
            node: 18
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

```
2 OS × 3 Node = 6개 job (exclude 1개 → 5개) 병렬 실행
```

> **`fail-fast: true`(기본값)면 하나가 실패할 때 나머지가 취소된다.** "Node 18에서만 깨지는지, 전부 깨지는지"를 알고 싶으면 `false`로 둔다.  
> 매트릭스를 크게 잡으면 러너 시간이 곱셈으로 늘어난다. PR에는 대표 조합만, 전체 매트릭스는 머지 후나 야간에 돌린다.

### 동적 매트릭스

```yaml
jobs:
  discover:
    runs-on: ubuntu-latest
    outputs:
      envs: ${{ steps.set.outputs.envs }}
    steps:
      - uses: actions/checkout@v4
      - id: set
        run: echo "envs=$(ls terraform/environments | jq -R -s -c 'split(\"\n\")[:-1]')" >> "$GITHUB_OUTPUT"

  plan:
    needs: discover
    strategy:
      matrix:
        env: ${{ fromJSON(needs.discover.outputs.envs) }}
    runs-on: ubuntu-latest
    steps:
      - run: echo "plan ${{ matrix.env }}"
```

> 환경 디렉터리가 늘어도 워크플로를 고칠 필요가 없어진다. → `../terraform-lab/08-workspace-environment/`

---

## 재사용 워크플로 (workflow_call)

여러 저장소·워크플로에서 같은 파이프라인을 쓸 때.

```yaml
# .github/workflows/reusable-build.yml
on:
  workflow_call:
    inputs:
      image_name:
        type: string
        required: true
      push:
        type: boolean
        default: false
    outputs:
      image_tag:
        value: ${{ jobs.build.outputs.tag }}
    secrets:
      registry_token:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.meta.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - id: meta
        run: echo "tag=$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"
```

```yaml
# 호출하는 쪽
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml      # 같은 저장소
    # uses: org/shared-workflows/.github/workflows/build.yml@v1   # 다른 저장소
    with:
      image_name: myapp
      push: ${{ github.ref == 'refs/heads/main' }}
    secrets: inherit          # 호출자의 시크릿을 그대로 전달
```

### 재사용 워크플로 vs Composite Action

| | 재사용 워크플로 | Composite Action |
|---|---|---|
| 단위 | **job 여러 개** | **step 여러 개** |
| 러너 | 자기 러너를 지정 | 호출한 job의 러너에서 실행 |
| 시크릿 | `secrets:` 로 명시 전달 | 호출자 컨텍스트 공유 |
| 쓸 때 | 파이프라인 전체를 공유 | 반복되는 스텝 묶음 |

```yaml
# .github/actions/setup-tools/action.yml (Composite Action)
name: Setup tools
inputs:
  terraform_version:
    default: "1.6.6"
runs:
  using: composite
  steps:
    - uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ inputs.terraform_version }}
    - run: terraform version
      shell: bash          # composite 의 run 에는 shell 이 필수
```

```yaml
- uses: ./.github/actions/setup-tools
  with:
    terraform_version: "1.9.5"
```

> **"setup 스텝 3~4개가 모든 워크플로에 반복된다"면 Composite Action**, **"빌드→테스트→스캔 전체 흐름이 반복된다"면 재사용 워크플로.**  
> composite action의 `run:`에는 `shell:`을 반드시 써야 한다 — 자주 빠뜨리는 부분이다.

---

## Concurrency — 중복 실행 제어

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true        # 같은 그룹의 이전 실행을 취소
```

```
PR에 커밋 3번 연속 푸시
  concurrency 없이 → 3개가 동시에 돌고 앞의 2개는 무의미
  cancel-in-progress → 최신 것만 남는다 (러너 시간 절약)
```

```yaml
# 배포는 절대 취소하면 안 된다 — 큐잉만 한다
concurrency:
  group: deploy-prod
  cancel-in-progress: false       # 순서대로 하나씩
```

> **PR 검증은 `cancel-in-progress: true`, 배포는 `false`.** 배포 도중 취소하면 클러스터가 중간 상태로 남는다.  
> 배포 그룹에 `github.ref`를 넣지 않는 게 포인트다 — 브랜치가 달라도 같은 환경에 배포한다면 하나씩 나가야 한다.

---

## OIDC — 장기 키 없이 클라우드 인증

```yaml
permissions:
  id-token: write        # 없으면 OIDC 토큰이 발급되지 않는다
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-deploy
          aws-region: ap-northeast-2
```

```
GitHub Actions ──OIDC 토큰──▶ AWS STS ──▶ 임시 자격증명 (1시간)
                                 │
                       신뢰 정책의 sub 조건으로 검증
                       "repo:GWANG-MIN1/infra:ref:refs/heads/main"
```

> 액세스 키를 시크릿에 넣지 않아도 된다. **키 유출·로테이션 문제가 사라진다.**  
> IAM 신뢰 정책의 `sub` 조건을 `repo:org/*`처럼 느슨하게 두면 다른 저장소에서도 그 역할을 맡을 수 있다. 브랜치·환경까지 좁힌다. → `../terraform-lab/10-cicd-policy/`

### Environment로 승인 게이트 걸기

```yaml
jobs:
  deploy-prod:
    environment:
      name: production          # GitHub Environments (required reviewers 설정)
      url: https://app.example.com
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

> Environment에 **required reviewers**를 걸면 job이 승인 대기 상태로 멈춘다. Environment 단위로 시크릿을 분리할 수도 있다 — prod 자격증명이 dev 워크플로에 노출되지 않는다.

---

## 게이팅 전략 — 2단 구성

전부 실패로 만들면 `main`이 상시 빨개지고, 그러면 아무도 안 본다.

```yaml
jobs:
  image-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 1단계: 보이게만 한다 (실패시키지 않음)
      - name: HIGH + CRITICAL 리포트
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          scan-type: image
          image-ref: nginxinc/nginx-unprivileged:1.30.4-alpine
          severity: HIGH,CRITICAL
          format: table
          exit-code: "0"                 # ← 리포트

      # 2단계: 고칠 수 있는 CRITICAL 만 막는다
      - name: 게이트
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          scan-type: image
          image-ref: nginxinc/nginx-unprivileged:1.30.4-alpine
          severity: CRITICAL
          ignore-unfixed: true           # ← 패치 없는 건 제외
          format: table
          exit-code: "1"                 # ← 게이트
```

| | 리포트 | 게이트 |
|---|---|---|
| exit-code | 0 | 1 |
| 범위 | HIGH + CRITICAL 전부 | **고칠 수 있는** CRITICAL |
| 목적 | 가시성 | 차단 |

> **`ignore-unfixed`가 핵심이다.** upstream에 패치가 없는 CVE로 빌드를 깨뜨리면 개발자는 스캔을 끄는 법을 배운다.  
> 처음엔 전부 리포트로 시작하고, 팀이 대응 가능해지면 게이트로 승격한다. **IaC 스캔도 마찬가지** — 런타임에서 Kyverno가 막고 있다면 CI는 리포트로 두고 나중에 게이트로 바꾼다.

```yaml
on:
  schedule:
    - cron: "0 6 * * 1"        # 월요일 06:00 UTC — 코드가 안 바뀌어도 재스캔
  workflow_dispatch:
```

> **주기적 재스캔이 필요한 이유:** 이미지 태그를 고정해두면 코드가 안 바뀌므로 push 트리거가 안 걸린다. 하지만 그 이미지의 CVE는 매일 새로 공개된다.

---

## 실패를 다루는 패턴

```yaml
- name: 테스트
  id: test
  continue-on-error: true              # 실패해도 다음 스텝 진행
  run: npm test

- name: 리포트 업로드
  if: always()                         # 성공·실패 모두
  uses: actions/upload-artifact@v4
  with:
    name: test-report
    path: reports/

- name: 실패 알림
  if: failure()
  run: ./notify-slack.sh

- name: 결과 반영
  if: steps.test.outcome == 'failure'
  run: exit 1                          # 마지막에 실제로 실패시킨다
```

| | `outcome` | `conclusion` |
|---|---|---|
| 의미 | `continue-on-error` **적용 전** 결과 | 적용 **후** 결과 |
| 용도 | 진짜 실패했는지 판단 | 워크플로 상태 |

> 테스트 리포트는 **실패했을 때 더 필요하다.** 업로드 스텝에는 `if: always()`를 붙인다.  
> `timeout-minutes`를 job에 걸어두면 무한 대기로 러너 시간이 새는 걸 막는다.

```yaml
jobs:
  build:
    timeout-minutes: 15        # 기본 360분(6시간)은 너무 길다
```

---

## 워크플로 검증

```bash
actionlint                                  # 문법·표현식 린트
act pull_request                            # 로컬 실행 (nektos/act)
gh workflow run deploy.yml -f env=dev       # 수동 트리거
gh run watch                                # 실시간 추적
gh run view <ID> --log-failed               # 실패 로그만
```

> **워크플로도 코드다.** `actionlint`를 pre-commit이나 CI에 넣으면 "`${{ }}` 오타로 조용히 조건이 항상 false가 되는" 사고를 막는다.

---

## 배운 점

- 매트릭스에서 **`fail-fast: false`** 여야 어느 조합이 깨지는지 알 수 있다
- 매트릭스는 러너 시간이 곱셈으로 는다 — PR엔 대표 조합만
- `fromJSON`으로 **동적 매트릭스**를 만들면 환경이 늘어도 워크플로를 안 고쳐도 된다
- **재사용 워크플로 = job 단위**, **Composite Action = step 단위**
- composite action의 `run:`에는 **`shell:`이 필수**
- `secrets: inherit`으로 호출자의 시크릿을 통째로 넘길 수 있다
- **PR 검증은 `cancel-in-progress: true`, 배포는 `false`** — 배포 취소는 중간 상태를 남긴다
- 배포 concurrency 그룹에는 `github.ref`를 넣지 않는다 (같은 환경은 하나씩)
- OIDC는 `permissions: id-token: write`가 없으면 동작하지 않는다
- IAM 신뢰 정책의 **`sub` 조건으로 저장소·브랜치까지 좁힌다**
- GitHub **Environments**로 승인 게이트와 환경별 시크릿 분리를 건다
- 게이팅은 **2단** — 전부 리포트 + 고칠 수 있는 것만 게이트
- **`ignore-unfixed`가 없으면 개발자는 스캔을 끄는 법을 배운다**
- 이미지 태그를 고정하면 push 트리거가 안 걸린다 → **주기적 재스캔(cron)** 필요
- 테스트 리포트 업로드에는 **`if: always()`**
- `outcome`(continue-on-error 적용 전)과 `conclusion`(적용 후)은 다르다
- `timeout-minutes`를 걸어 무한 대기를 막는다 (기본 6시간)
- `actionlint`로 워크플로 자체를 린트한다
