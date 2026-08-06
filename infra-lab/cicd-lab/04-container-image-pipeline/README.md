# 04 컨테이너 이미지 파이프라인

CI의 최종 산출물은 테스트 통과가 아니라 **배포 가능한 이미지 하나**다.  
그 이미지가 어떻게 만들어졌고, 무엇이 들어있고, 위조되지 않았는지를 보장하는 것이 이 장의 주제다.

---

## 전체 흐름

```
소스 ──▶ 멀티스테이지 빌드 ──▶ 스캔 ──▶ 태그 ──▶ 레지스트리 푸시 ──▶ SBOM·서명
          (buildx + 캐시)      (Trivy)   (불변)     (ECR)          (cosign)
                                                      │
                                          매니페스트 태그 갱신 → GitOps
```

---

## 멀티스테이지 빌드

빌드 도구와 런타임을 분리한다. **최종 이미지에 컴파일러·빌드 캐시가 남지 않게** 하는 것이 핵심이다.

```dockerfile
# ---- 빌드 스테이지 ----
FROM node:20-alpine AS builder
WORKDIR /app

# 의존성 파일만 먼저 복사 → 소스가 바뀌어도 npm ci 레이어는 캐시 적중
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# ---- 런타임 스테이지 ----
FROM node:20-alpine
WORKDIR /app

# 비-root 사용자
RUN addgroup -S app && adduser -S app -G app

COPY --from=builder --chown=app:app /app/dist ./dist
COPY --from=builder --chown=app:app /app/node_modules ./node_modules

USER app
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

| 원칙 | 이유 |
|---|---|
| **의존성 파일을 먼저 COPY** | 소스 변경 시 의존성 설치 레이어가 캐시된다 |
| **런타임 스테이지는 최소 베이스** | alpine·distroless — 공격 표면과 CVE 수가 줄어든다 |
| **비-root 실행 (`USER`)** | 컨테이너 탈출 시 피해 축소 → `../docker-labs/09-security/` |
| **`.dockerignore`** | `.git`, `node_modules`가 빌드 컨텍스트에 들어가지 않게 |

```gitignore
# .dockerignore
.git
node_modules
.env
*.md
.github
```

> **레이어 순서가 캐시 효율을 결정한다.** `COPY . .`를 맨 앞에 두면 파일 하나만 바뀌어도 그 뒤 모든 레이어가 무효화된다.  
> distroless(`gcr.io/distroless/nodejs20`)는 셸조차 없어 더 안전하지만, 디버깅이 어렵다는 트레이드오프가 있다.

---

## buildx와 캐시

CI 러너는 매번 새 머신이라 로컬 도커 캐시가 없다. **캐시를 외부에 저장**해야 빌드가 빨라진다.

```yaml
- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    cache-from: type=gha              # GitHub Actions 캐시에서 읽기
    cache-to: type=gha,mode=max       # 중간 레이어까지 저장
    platforms: linux/amd64,linux/arm64
    provenance: true                  # 빌드 출처 정보 첨부
```

| 캐시 백엔드 | 특징 |
|---|---|
| `type=gha` | GitHub Actions 캐시 (저장소당 10GB 공유) |
| `type=registry` | 레지스트리에 캐시 이미지로 저장 (크기 제한 없음) |
| `type=local` | 자체 호스팅 러너용 |

```yaml
# 레지스트리 캐시 — 대형 프로젝트
cache-from: type=registry,ref=<ACCT>.dkr.ecr.../myapp:buildcache
cache-to: type=registry,ref=<ACCT>.dkr.ecr.../myapp:buildcache,mode=max
```

> `mode=max`는 중간 스테이지 레이어까지 캐시한다. 멀티스테이지에서 효과가 크다.  
> **멀티 아키텍처 빌드(`linux/arm64`)는 QEMU 에뮬레이션으로 느리다.** Graviton 노드를 쓸 게 아니면 amd64만 빌드한다.

---

## 태그 전략

```
❌ myapp:latest        ← 무엇이 배포됐는지 알 수 없다. 롤백 불가.
✅ myapp:1.4.2         ← 시맨틱 버전
✅ myapp:a1b2c3d       ← 커밋 SHA (추적 가능)
✅ myapp@sha256:...    ← 다이제스트 (가장 강력, 불변 보장)
```

| 태그 | 용도 |
|---|---|
| **커밋 SHA** | 모든 빌드에 부여. "이 이미지 = 이 커밋" |
| **시맨틱 버전** | 릴리스 태그에서 |
| **다이제스트** | 운영 배포 — 태그는 옮길 수 있지만 다이제스트는 못 옮긴다 |
| `latest` | 로컬 실험용. **운영 금지** |

```yaml
- uses: docker/metadata-action@v5
  id: meta
  with:
    images: <ACCT>.dkr.ecr.ap-northeast-2.amazonaws.com/myapp
    tags: |
      type=sha,format=long              # sha-a1b2c3d...
      type=ref,event=branch             # main
      type=semver,pattern={{version}}   # v1.4.2 태그 → 1.4.2
      type=raw,value=latest,enable={{is_default_branch}}
```

> **태그는 이동 가능하다.** 누군가 `myapp:1.4.2`를 다시 푸시하면 같은 태그가 다른 이미지를 가리킨다.  
> 그래서 운영 매니페스트에는 **다이제스트를 박는 것**이 가장 안전하다. ECR의 **태그 불변성(immutable tags)** 을 켜두면 덮어쓰기 자체가 막힌다.

```bash
aws ecr put-image-tag-mutability --repository-name myapp --image-tag-mutability IMMUTABLE
```

---

## ECR에 푸시

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-actions-ecr
      aws-region: ap-northeast-2

  - uses: aws-actions/amazon-ecr-login@v2
    id: ecr

  - uses: docker/build-push-action@v6
    with:
      push: true
      tags: ${{ steps.meta.outputs.tags }}
```

```bash
# 리포지터리 준비 — 푸시 시 자동 스캔
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability IMMUTABLE
```

### 수명주기 정책 — 안 지우면 요금이 쌓인다

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "태그 없는 이미지 1일 후 삭제",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "최근 30개만 보관",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["sha-"],
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": { "type": "expire" }
    }
  ]
}
```

> 커밋마다 이미지를 푸시하면 금방 수백 개가 쌓인다. **수명주기 정책은 나중이 아니라 리포지터리를 만들 때 같이 건다.**  
> 단, 롤백 대상이 지워지지 않도록 보관 개수를 넉넉히 잡는다.

---

## 취약점 스캔

```yaml
- name: HIGH + CRITICAL 리포트 (게이트 아님)
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    scan-type: image
    image-ref: myapp:${{ github.sha }}
    severity: HIGH,CRITICAL
    format: table
    exit-code: "0"

- name: 고칠 수 있는 CRITICAL 만 게이트
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    scan-type: image
    image-ref: myapp:${{ github.sha }}
    severity: CRITICAL
    ignore-unfixed: true
    exit-code: "1"

- name: GitHub Security 탭에 업로드
  if: always()
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: results.sarif
```

> 2단 게이팅의 근거는 `03-github-actions-advanced/`와 같다. **패치 없는 CVE로 막으면 스캔 자체가 꺼진다.**  
> 스캔은 **빌드 직후·푸시 전**이 이상적이다. 취약한 이미지를 레지스트리에 올리지 않는다.  
> 런타임 쪽에서는 Kyverno가 같은 일을 한다 — CI는 shift-left, Kyverno는 admission. 두 겹이다.

### 베이스 이미지가 진짜 원인이다

```
myapp 이미지의 CVE 대부분 = 내 코드가 아니라 베이스 이미지의 OS 패키지
  → 해결책: 베이스를 최신으로 갱신, 더 작은 베이스(alpine·distroless)로 교체
  → 자동화: Dependabot / Renovate 가 Dockerfile 의 FROM 을 PR로 올려준다
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: docker
    directory: /
    schedule: { interval: weekly }
  - package-ecosystem: github-actions        # 액션 버전도 갱신
    directory: /
    schedule: { interval: weekly }
```

---

## SBOM과 서명 (공급망 보안)

```
SBOM (Software Bill of Materials) = 이 이미지에 무엇이 들어있는지의 목록
서명 (Signing)                    = 이 이미지를 내가 만든 게 맞다는 증명
```

```yaml
- name: SBOM 생성
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json

- name: 이미지 서명 (keyless — OIDC 사용, 키 관리 불필요)
  run: |
    cosign sign --yes \
      "$IMAGE@${{ steps.build.outputs.digest }}"
  env:
    COSIGN_EXPERIMENTAL: "1"

- name: 검증
  run: |
    cosign verify "$IMAGE@$DIGEST" \
      --certificate-identity-regexp "https://github.com/GWANG-MIN1/.*" \
      --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

| 항목 | 답하는 질문 |
|---|---|
| **SBOM** | "log4j 취약점이 터졌는데 우리 이미지에 들어있나?" |
| **서명** | "이 이미지가 우리 파이프라인에서 나온 게 맞나?" |
| **Provenance** | "어떤 소스·어떤 빌더로 만들어졌나?" (SLSA) |

> **SBOM의 가치는 사고가 났을 때 드러난다.** 새 CVE가 공개됐을 때 "우리가 영향받는가"를 몇 분 안에 답할 수 있다.  
> cosign의 **keyless 서명**은 OIDC 신원으로 서명해 개인키를 관리할 필요가 없다. 서명 자체보다 **클러스터에서 검증을 강제**하는 게 중요하다 — Kyverno의 `verifyImages` 정책으로 서명 없는 이미지의 배포를 막는다.

---

## 전체 워크플로 예시

```yaml
name: build-image

on:
  push:
    branches: [main]
    paths: ["src/**", "Dockerfile", ".github/workflows/build-image.yml"]

permissions:
  contents: read
  id-token: write

concurrency:
  group: build-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.ECR_ROLE_ARN }}
          aws-region: ap-northeast-2
      - uses: aws-actions/amazon-ecr-login@v2

      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ vars.ECR_REGISTRY }}/myapp
          tags: |
            type=sha,format=long
            type=ref,event=branch

      - uses: docker/build-push-action@v6
        id: build
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true

      - name: 스캔 (게이트)
        uses: aquasecurity/trivy-action@v0.36.0
        with:
          scan-type: image
          image-ref: ${{ vars.ECR_REGISTRY }}/myapp@${{ steps.build.outputs.digest }}
          severity: CRITICAL
          ignore-unfixed: true
          exit-code: "1"

      - name: 요약
        run: |
          echo "### 이미지 빌드 완료" >> "$GITHUB_STEP_SUMMARY"
          echo "digest: \`${{ steps.build.outputs.digest }}\`" >> "$GITHUB_STEP_SUMMARY"
```

> 여기서 CI는 끝난다. **다음은 이 digest를 매니페스트에 반영하는 일** — GitOps의 영역이다. → `07-gitops-repo-strategy/`

---

## 배운 점

- CI의 산출물은 테스트 통과가 아니라 **배포 가능한 이미지 하나**
- 멀티스테이지로 **빌드 도구를 최종 이미지에서 제거**한다
- **의존성 파일을 먼저 COPY** 해야 소스 변경 시 설치 레이어가 캐시된다
- `.dockerignore`가 없으면 `.git`·`node_modules`가 빌드 컨텍스트에 들어간다
- CI 러너는 매번 새 머신 — **`cache-from/to type=gha`** 로 외부 캐시를 쓴다
- `mode=max`는 중간 스테이지까지 캐시 (멀티스테이지에서 효과가 크다)
- **멀티 아키텍처 빌드는 QEMU라 느리다** — 필요할 때만
- **`latest`는 운영 금지** — 무엇이 배포됐는지 알 수 없고 롤백이 불가능
- 태그는 이동 가능하다 → 운영에는 **다이제스트**, ECR은 **태그 불변성**을 켠다
- ECR **수명주기 정책은 리포지터리를 만들 때 함께** 건다 (롤백 대상은 남기고)
- 스캔은 **푸시 전**에 — 취약한 이미지를 레지스트리에 올리지 않는다
- CVE 대부분은 내 코드가 아니라 **베이스 이미지의 OS 패키지**
- Dependabot/Renovate로 **베이스 이미지와 액션 버전을 자동 갱신**
- **SBOM의 가치는 사고 때 드러난다** — "우리가 영향받는가"를 즉답
- cosign **keyless 서명**은 OIDC 신원 기반이라 키 관리가 필요 없다
- 서명보다 중요한 건 **클러스터에서 검증을 강제**하는 것 (Kyverno `verifyImages`)
