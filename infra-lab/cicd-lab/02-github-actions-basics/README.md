# 02 GitHub Actions 기초

GitHub Actions는 **저장소 이벤트에 반응해 컨테이너/VM에서 스크립트를 돌리는 시스템**이다.  
별도 CI 서버를 운영하지 않아도 되고, 워크플로가 코드로 저장소 안에 있어 PR로 리뷰된다.

---

## 구조

```
Workflow (.github/workflows/ci.yml)
└── Job (병렬 실행이 기본, 각 job은 별도 러너 = 별도 머신)
    └── Step (순차 실행, 같은 러너에서)
        ├── uses: 액션 재사용
        └── run: 셸 명령
```

```yaml
name: ci                       # 워크플로 이름

on:                            # 언제 실행할지
  push:
    branches: [main]
  pull_request:

jobs:
  test:                        # job ID
    runs-on: ubuntu-latest     # 러너
    steps:
      - uses: actions/checkout@v4        # 액션 사용
      - name: Run tests                  # 셸 명령
        run: npm ci && npm test
```

> **job은 서로 다른 머신에서 돈다.** 한 job에서 만든 파일이 다른 job에 자동으로 넘어가지 않는다 — 아티팩트나 캐시를 써야 한다.  
> step은 같은 머신에서 순차 실행되므로 디렉터리·파일이 유지된다.

---

## 트리거 (on)

```yaml
on:
  push:
    branches: [main]
    tags: ["v*"]
    paths:                             # 이 경로가 바뀔 때만
      - "terraform/**"
      - ".github/workflows/terraform-ci.yml"
    paths-ignore: ["docs/**", "**.md"]

  pull_request:
    types: [opened, synchronize, reopened]   # 기본값
    branches: [main]

  schedule:
    - cron: "0 6 * * 1"                # 매주 월요일 06:00 UTC (한국 15:00)

  workflow_dispatch:                   # 수동 실행 (UI 버튼 / gh CLI)
    inputs:
      environment:
        type: choice
        options: [dev, prod]
        required: true

  workflow_call:                       # 다른 워크플로가 호출 (재사용)
  release:
    types: [published]
```

| 트리거 | 언제 쓰나 |
|---|---|
| `pull_request` | 검증 (머지 전 게이트) |
| `push` (main) | 배포·이미지 빌드 |
| `schedule` | 정기 보안 스캔, 의존성 갱신 |
| `workflow_dispatch` | 수동 배포·롤백 |
| `workflow_call` | 공통 워크플로 재사용 → `03-github-actions-advanced/` |

> **`paths` 필터를 안 걸면 문서 오타 하나에 전체 파이프라인이 돈다.** 러너 시간과 대기 시간을 모두 낭비한다.  
> 단, **워크플로 파일 자신도 `paths`에 포함**시켜야 한다. 안 그러면 워크플로를 고쳤을 때 검증이 안 돌아간다.

> ⚠️ `schedule`은 UTC 기준이고, GitHub 부하에 따라 **수 분에서 수십 분 지연**될 수 있다. 정확한 시각이 필요한 작업에는 쓰지 않는다.

---

## 러너

```yaml
runs-on: ubuntu-latest         # 가장 흔함, 가장 빠름
runs-on: ubuntu-24.04          # 버전 고정 (latest 가 바뀌어 깨지는 걸 방지)
runs-on: windows-latest
runs-on: macos-latest          # 분당 단가가 가장 비싸다
runs-on: [self-hosted, linux, x64]   # 자체 호스팅
```

| | GitHub 호스팅 | 자체 호스팅 |
|---|---|---|
| 관리 | 불필요 | 직접 운영 |
| 격리 | 매번 새 VM | **재사용됨 (오염 주의)** |
| 네트워크 | 공용 인터넷 | VPC 내부 접근 가능 |
| 비용 | 분당 과금 (퍼블릭 레포는 무료) | 인프라 비용 |

> **자체 호스팅 러너는 퍼블릭 저장소에 쓰지 않는다.** 포크에서 온 PR이 러너에서 임의 코드를 실행할 수 있다.  
> `ubuntu-latest`는 어느 날 바뀐다. 안정성이 중요하면 `ubuntu-24.04`처럼 고정한다.

---

## 액션 사용 (uses)

```yaml
- uses: actions/checkout@v4                    # 태그
- uses: actions/setup-node@v4
  with:
    node-version: "20"
    cache: npm

- uses: hashicorp/setup-terraform@v3
  with:
    terraform_version: "1.6.6"                 # 버전 고정

- uses: aquasecurity/trivy-action@v0.36.0      # 서드파티는 특히 고정
```

| 참조 방식 | 안전도 |
|---|---|
| `@v4` (메이저 태그) | 보통 — 태그가 이동할 수 있다 |
| `@a1b2c3d...` (커밋 SHA) | **가장 안전** — 변하지 않는다 |
| `@main` | ❌ 위험 — 언제든 바뀐다 |

> 서드파티 액션은 **저장소의 시크릿에 접근할 수 있는 코드**다. 공급망 공격의 실제 경로이므로, 민감한 워크플로에서는 **커밋 SHA로 고정**한다.  
> `actions/*`, `github/*`는 GitHub 공식이라 태그로 써도 무방하다.

---

## 컨텍스트와 표현식

```yaml
${{ github.ref }}              # refs/heads/main
${{ github.ref_name }}         # main
${{ github.sha }}              # 커밋 SHA (전체)
${{ github.event_name }}       # push / pull_request
${{ github.actor }}            # 실행한 사용자
${{ github.repository }}       # owner/repo
${{ github.run_id }}
${{ secrets.MY_SECRET }}
${{ env.MY_VAR }}
${{ vars.MY_VARIABLE }}        # 저장소 변수 (비밀 아님)
${{ steps.<id>.outputs.<key> }}
${{ needs.<job_id>.outputs.<key> }}
```

### 조건부 실행

```yaml
- name: Deploy
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  run: ./deploy.sh

- name: PR 코멘트
  if: github.event_name == 'pull_request'

- name: 실패 시에만
  if: failure()

- name: 앞 단계 결과와 무관하게 항상
  if: always()
```

| 함수 | 의미 |
|---|---|
| `success()` | 기본값 — 앞 스텝이 모두 성공 |
| `failure()` | 앞 스텝이 실패 |
| `always()` | 항상 (정리·알림용) |
| `cancelled()` | 취소됨 |
| `contains(a, b)` / `startsWith` | 문자열 검사 |

> `if:` 안에서는 `${{ }}`를 생략할 수 있다. 붙여도 동작한다.

---

## 스텝 간 값 전달

```yaml
steps:
  - name: 버전 계산
    id: version
    run: echo "tag=1.2.$(date +%s)" >> "$GITHUB_OUTPUT"

  - name: 사용
    run: echo "빌드 태그 ${{ steps.version.outputs.tag }}"
```

```yaml
# 환경변수로 넘기기
- run: echo "IMAGE_TAG=1.2.3" >> "$GITHUB_ENV"
- run: echo "$IMAGE_TAG"       # 다음 스텝부터 사용 가능

# 요약 페이지에 출력 (실행 결과 화면에 마크다운으로 표시)
- run: echo "### 배포 완료 :rocket:" >> "$GITHUB_STEP_SUMMARY"
```

> **`set-output` 명령은 폐기됐다.** `$GITHUB_OUTPUT` 파일에 쓰는 방식을 쓴다.  
> `$GITHUB_STEP_SUMMARY`는 의외로 유용하다 — plan 결과나 스캔 요약을 실행 화면에 바로 띄울 수 있다.

---

## job 간 의존과 값 전달

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tag }}
    steps:
      - id: meta
        run: echo "tag=$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build                                  # build 완료 후 실행
    runs-on: ubuntu-latest
    steps:
      - run: echo "배포 ${{ needs.build.outputs.image_tag }}"
```

```
needs 없으면 → 모든 job이 병렬
needs 있으면 → 순서가 생긴다

  lint ─┐
  test ─┼─▶ build ──▶ deploy
  scan ─┘
```

---

## 아티팩트와 캐시

**전혀 다른 용도인데 자주 혼동된다.**

| | 아티팩트 | 캐시 |
|---|---|---|
| 목적 | **결과물 보관·전달** | **속도 향상** |
| 없으면 | 워크플로가 실패한다 | 느려질 뿐 정상 동작 |
| 수명 | 기본 90일 (설정 가능) | 7일 미사용 시 삭제, 저장소당 10GB |
| 예시 | 빌드 산출물, 테스트 리포트, 스캔 결과 | node_modules, ~/.m2, 도커 레이어 |

```yaml
# 아티팩트 — job 간 전달
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
    retention-days: 7

- uses: actions/download-artifact@v4
  with:
    name: build-output

# 캐시 — 의존성 재사용
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-npm-
```

> **캐시 키에 lock 파일 해시를 넣는다.** 의존성이 바뀌면 키가 바뀌어 새로 받고, 안 바뀌면 그대로 쓴다.  
> `restore-keys`는 정확히 일치하는 캐시가 없을 때 접두사로 가장 최근 것을 가져온다 — 부분 캐시라도 없는 것보다 낫다.  
> `setup-node`·`setup-python` 등은 `cache:` 옵션으로 이걸 내장하고 있다. 직접 쓰기 전에 확인한다.

---

## 시크릿과 권한

```yaml
permissions:              # 최소 권한 원칙 — 워크플로 최상단에 명시
  contents: read

jobs:
  deploy:
    permissions:          # job 단위로도 지정 가능
      contents: read
      id-token: write     # OIDC 토큰 발급 (클라우드 인증)
      pull-requests: write
```

```yaml
- run: ./deploy.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}        # 로그에 자동으로 마스킹된다
```

| | 용도 |
|---|---|
| `secrets` | 비밀 값 — 로그에서 `***`로 마스킹 |
| `vars` | 비밀 아닌 설정값 (리전, 환경 이름) |
| `GITHUB_TOKEN` | 자동 제공 토큰 — `permissions`로 범위 제어 |

> **`permissions`를 명시하지 않으면 저장소 기본값**(대개 넓다)이 적용된다. 최상단에 `contents: read`를 두고 필요한 job에서만 확장하는 게 안전하다.  
> 시크릿은 마스킹되지만 **base64로 인코딩해 출력하면 그대로 노출된다.** 신뢰할 수 없는 액션에 시크릿을 넘기지 않는다.

> ⚠️ **`pull_request_target`은 포크 PR에서도 시크릿에 접근할 수 있다.** 편해 보이지만 대표적인 권한 상승 경로다. 반드시 필요한 경우가 아니면 쓰지 않는다.

---

## 실전 — 자격증명 없는 검증 파이프라인

```yaml
name: terraform-ci

# 포맷 + 검증만. apply 없음, 자격증명 없음 —
# 배포 파이프라인이 아니라 main 을 초록으로 유지하는 안전망이다.

on:
  push:
    paths:
      - "terraform/**"
      - ".github/workflows/terraform-ci.yml"    # 워크플로 자신도 포함
  pull_request:
    paths:
      - "terraform/**"

permissions:
  contents: read                                # 최소 권한

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.6.6"            # 버전 고정

      - run: terraform fmt -check -recursive

      - name: Init (백엔드 없이)
        working-directory: terraform/environments/dev
        run: terraform init -backend=false      # 자격증명 불필요

      - name: Validate
        working-directory: terraform/environments/dev
        run: terraform validate
```

> **자격증명이 필요 없는 검증을 먼저 갖추는 게 순서다.** `-backend=false`로 init하면 클라우드 접근 없이 문법·타입 검증이 된다.  
> 포크 PR에서도 안전하게 돌릴 수 있고, 시크릿 설정 전에 바로 시작할 수 있다. → `../terraform-lab/10-cicd-policy/`

---

## 배운 점

- 구조는 **Workflow → Job(병렬, 별도 머신) → Step(순차, 같은 머신)**
- **job은 머신이 다르다** — 파일 전달은 아티팩트/캐시로
- `paths` 필터로 무관한 변경에 파이프라인이 돌지 않게 한다
- **워크플로 파일 자신도 `paths`에 포함**시켜야 워크플로 수정이 검증된다
- `schedule`은 UTC 기준이고 수십 분 지연될 수 있다
- `ubuntu-latest`는 바뀐다 — 안정성이 필요하면 버전을 고정
- **자체 호스팅 러너를 퍼블릭 저장소에 쓰지 않는다** (포크 PR이 코드를 실행)
- 서드파티 액션은 저장소 시크릿에 닿는 코드 — **커밋 SHA로 고정**
- 스텝 간 값 전달은 `$GITHUB_OUTPUT` (`set-output`은 폐기)
- `$GITHUB_STEP_SUMMARY`로 실행 화면에 마크다운 요약을 남길 수 있다
- job 순서는 `needs`, 값 전달은 `outputs` + `needs.<job>.outputs`
- **아티팩트=결과물 전달, 캐시=속도** — 캐시가 없어도 동작해야 정상
- 캐시 키에 **lock 파일 해시**를 넣고 `restore-keys`로 부분 적중을 노린다
- **`permissions`를 최상단에 `contents: read`로** 두고 필요한 곳만 확장
- **`pull_request_target`은 포크 PR에 시크릿을 노출**할 수 있다 — 피한다
- 시작은 **자격증명 없는 검증 파이프라인**이 안전하고 빠르다
