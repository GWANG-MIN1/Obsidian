# 07 이미지 보안

빌드 쪽 파이프라인(멀티스테이지·캐시·태그 전략)은 앞에서 다뤘다. → `../cicd-lab/04-container-image-pipeline/`  
이 장은 **보안 관점** — 무엇을 스캔하고, 어디까지 막고, 클러스터에서 어떻게 검증을 강제할 것인가다.

---

## 이미지 위협 모델

```
공격 지점
① 베이스 이미지의 알려진 CVE          ← 가장 흔하다
② 애플리케이션 의존성의 CVE
③ 이미지에 굽힌 시크릿
④ 위조된 이미지 (레지스트리 침해·타이포스쿼팅)
⑤ 신뢰할 수 없는 레지스트리
```

| 위협 | 대응 |
|---|---|
| ① ② CVE | Trivy 스캔 (CI) + 정기 재스캔 |
| ③ 시크릿 | `trivy fs --scanners secret`, gitleaks |
| ④ 위조 | cosign 서명 + **Kyverno verifyImages** |
| ⑤ 레지스트리 | admission으로 허용 레지스트리 제한 |

> **①이 압도적으로 많다.** 내가 쓴 코드가 아니라 베이스 이미지의 OS 패키지가 대부분의 CVE를 만든다.

---

## Trivy로 스캔하기

```bash
trivy image myapp:1.4.2                                  # 취약점
trivy image --severity CRITICAL --ignore-unfixed myapp:1.4.2
trivy image --scanners vuln,secret,misconfig myapp:1.4.2 # 시크릿·설정까지
trivy image --format cyclonedx -o sbom.json myapp:1.4.2  # SBOM
trivy fs --scanners vuln,secret .                        # 소스 코드
trivy config .                                           # IaC 미스컨피그
trivy k8s --report summary cluster                       # 실행 중인 클러스터
```

### 2단 게이팅

```yaml
# 1단계 — 보이게만 한다
- name: HIGH + CRITICAL 리포트
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    scan-type: image
    image-ref: nginxinc/nginx-unprivileged:1.30.4-alpine
    severity: HIGH,CRITICAL
    format: table
    exit-code: "0"

# 2단계 — 고칠 수 있는 CRITICAL 만 막는다
- name: 게이트
  uses: aquasecurity/trivy-action@v0.36.0
  with:
    scan-type: image
    image-ref: nginxinc/nginx-unprivileged:1.30.4-alpine
    severity: CRITICAL
    ignore-unfixed: true
    exit-code: "1"
```

> **`ignore-unfixed`가 핵심이다.** upstream에 패치가 없는 CVE로 빌드를 깨뜨리면 개발자는 스캔을 끄는 법을 배운다.  
> 게이트를 넓게 잡으면 `main`이 상시 빨개지고, 그러면 아무도 안 본다. → `01-security-basics/`

### 정기 재스캔이 필요한 이유

```yaml
on:
  schedule:
    - cron: "0 6 * * 1"      # 매주 월요일
  workflow_dispatch:
```

```
이미지 태그를 고정 → 코드가 안 바뀌면 push 트리거가 안 걸린다
                   → 하지만 그 이미지의 CVE 는 매일 새로 공개된다
                   → 스케줄 재스캔이 없으면 영원히 모른다
```

### 예외 관리

```yaml
# .trivyignore
CVE-2023-12345    # 해당 기능을 쓰지 않음 (확인: 2026-08-01, 재검토: 2026-11)
```

> **예외에는 반드시 이유와 재검토 시점을 적는다.** 이유 없는 ignore는 그냥 무시와 같다.

---

## CVE를 줄이는 실제 방법

스캔은 문제를 알려줄 뿐 줄여주지 않는다. 줄이는 건 **베이스 이미지 선택**이다.

| 베이스 | 크기 | CVE 수 | 디버깅 |
|---|---|---|---|
| `ubuntu:24.04` | ~78MB | 많다 | 쉽다 |
| `debian:bookworm-slim` | ~30MB | 보통 | 쉽다 |
| `alpine:3.20` | ~7MB | 적다 | 보통 (musl libc 주의) |
| `gcr.io/distroless/*` | ~2-20MB | **가장 적다** | **어렵다 (셸 없음)** |
| `scratch` | 0 | 없다 | 불가 (정적 바이너리만) |

```
공격 표면 = 이미지에 들어있는 패키지 수
  → 셸이 없으면 셸을 통한 공격도 없다
  → distroless 는 CVE 와 탈출 경로를 동시에 줄인다
```

> **디버깅이 어렵다는 단점은 `kubectl debug` 임시 컨테이너로 상당 부분 해소된다.**

```bash
kubectl debug -it <POD> --image=busybox --target=app -n myapp
```

### 자동 갱신

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: docker      # Dockerfile 의 FROM
    directory: /
    schedule: { interval: weekly }
  - package-ecosystem: github-actions
    directory: /
    schedule: { interval: weekly }
```

> 베이스 이미지를 최신으로 유지하는 것만으로 CVE의 상당수가 사라진다. **PR로 올라오게 자동화**해두면 잊지 않는다.

---

## SBOM

```
SBOM (Software Bill of Materials) = 이 이미지에 무엇이 들어있는지의 목록
```

```bash
trivy image --format cyclonedx -o sbom.json myapp:1.4.2
syft myapp:1.4.2 -o spdx-json > sbom.json
cosign attach sbom --sbom sbom.json myapp:1.4.2      # 이미지에 첨부
```

> **SBOM의 가치는 사고가 났을 때 드러난다.** 새 CVE가 공개됐을 때 "우리가 영향받는가"를 몇 분 안에 답할 수 있다. 없으면 이미지를 하나씩 다시 스캔해야 한다.  
> 형식은 **CycloneDX**와 **SPDX** 두 표준이 있다. 도구 지원이 넓은 쪽을 고른다.

---

## 서명과 검증

```
서명(cosign)      "이 이미지를 우리 파이프라인이 만들었다"는 증명
검증(Kyverno)     서명이 없거나 신원이 다르면 클러스터가 거부
```

### keyless 서명 — 키 관리 없이

```yaml
- name: 서명
  run: cosign sign --yes "$IMAGE@${{ steps.build.outputs.digest }}"
  # OIDC 신원(워크플로 정체성)으로 서명 → 개인키 파일이 없다
```

```bash
cosign verify myapp@sha256:... \
  --certificate-identity-regexp "https://github.com/GWANG-MIN1/.*" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

### ⭐ 클러스터에서 검증 강제하기

**서명만 하고 검증을 강제하지 않으면 아무 의미가 없다.** 공격자는 서명 없는 이미지를 배포하면 그만이다.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      exclude:
        any:
          - resources:
              namespaces: [kube-system, argocd, observability, kyverno]
      verifyImages:
        - imageReferences:
            - "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/*"
          failureAction: Audit          # 익숙해지면 Enforce
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/GWANG-MIN1/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

> **이게 공급망 보안의 완결 지점이다.** CI가 서명하고 클러스터가 검증하면, 파이프라인을 거치지 않은 이미지는 실행될 수 없다.  
> 다른 정책과 마찬가지로 **Audit으로 시작**한다. Enforce로 바로 켜면 서명 안 된 기존 워크로드가 전부 막힌다. → `04-kyverno/`

---

## 레지스트리 제한

```yaml
validate:
  failureAction: Audit
  message: "승인된 레지스트리의 이미지만 사용할 수 있습니다."
  pattern:
    spec:
      containers:
        - image: "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/*"
```

> **타이포스쿼팅**(`nginx` 대신 `ngnix`)과 침해된 퍼블릭 이미지를 막는다.  
> 퍼블릭 이미지를 쓸 수밖에 없다면 **ECR pull-through 캐시**로 한 번 거쳐 들어오게 한다 — 스캔과 감사가 가능해진다.

---

## 태그 불변성

```bash
aws ecr put-image-tag-mutability --repository-name myapp --image-tag-mutability IMMUTABLE
```

```
태그는 이동할 수 있다
  누군가 myapp:1.4.2 를 다시 푸시 → 같은 태그가 다른 이미지를 가리킨다
  → 스캔을 통과한 그 이미지가 실제로 도는 것이 아닐 수 있다
```

| 대응 | |
|---|---|
| **다이제스트로 배포** | `myapp@sha256:...` — 이동 불가 |
| **ECR 태그 불변성** | 덮어쓰기 자체를 막는다 |
| **`:latest` 금지** | Kyverno `disallow-latest-tag` |

> `:latest`를 막는 이유는 보안보다 **추적 가능성**이다 — 무엇이 실제로 돌았는지 알 수 없고, 재-pull로 워크로드가 조용히 바뀔 수 있다.

---

## 런타임 스캔

빌드 시점에 깨끗했어도 **시간이 지나면 CVE가 생긴다.** 지금 클러스터에서 도는 것을 스캔한다.

```bash
trivy k8s --report summary cluster
trivy k8s --namespace myapp --report all
```

```yaml
# Trivy Operator — 클러스터 안에서 상시 스캔, CRD 로 결과 저장
kubectl get vulnerabilityreports -A
kubectl get configauditreports -A
```

> **CI 스캔은 "그때 깨끗했다", 런타임 스캔은 "지금 깨끗한가"** 를 답한다. 둘 다 필요하다.  
> Trivy Operator를 GitOps로 배포하면 결과가 CRD로 남아 Prometheus로 지표화할 수도 있다. → `../observability-lab/`

---

## 전체 방어선

```
개발      Dockerfile: 최소 베이스, 비-root, .dockerignore
   ↓
CI        Trivy 스캔 (2단 게이팅) → 서명 → SBOM 첨부
   ↓
레지스트리  ECR 태그 불변성, scanOnPush, 수명주기 정책
   ↓
Admission  Kyverno: 레지스트리 제한, :latest 금지, 서명 검증
   ↓
런타임      Trivy Operator 상시 스캔, 정기 재스캔
```

> **어느 한 겹도 완벽하지 않다는 전제로 겹쳐 쌓는다.** CI 스캔은 `kubectl apply`로 우회되고, admission은 이미 도는 파드를 못 막고, 런타임 스캔은 이미 배포된 뒤다.

---

## 배운 점

- CVE의 대부분은 내 코드가 아니라 **베이스 이미지의 OS 패키지**
- **`ignore-unfixed` 없이 게이트를 걸면 개발자는 스캔을 끄는 법을 배운다**
- 2단 게이팅: **전부 리포트 + 고칠 수 있는 CRITICAL만 차단**
- 태그를 고정하면 push 트리거가 안 걸린다 → **정기 재스캔(cron) 필수**
- 예외(`.trivyignore`)에는 **이유와 재검토 시점**을 적는다
- 스캔은 문제를 알려줄 뿐 — **줄이는 건 베이스 이미지 선택**
- distroless·alpine은 CVE와 **탈출 경로를 동시에** 줄인다 (셸이 없으면 셸 공격도 없다)
- 디버깅은 **`kubectl debug` 임시 컨테이너**로 해소된다
- Dependabot으로 **베이스 이미지·액션 버전을 주간 자동 갱신**
- **SBOM의 가치는 사고 때 드러난다** — "우리가 영향받는가"를 즉답
- cosign **keyless 서명**은 OIDC 신원 기반이라 개인키가 없다
- ⭐ **서명만 하고 검증을 강제하지 않으면 무의미하다** — Kyverno `verifyImages`
- 서명 검증도 **Audit으로 시작** — Enforce로 바로 켜면 기존 워크로드가 전부 막힌다
- 레지스트리 제한으로 **타이포스쿼팅**과 침해된 퍼블릭 이미지를 막는다
- **태그는 이동 가능하다** → 다이제스트 배포 + ECR 태그 불변성
- `:latest` 금지의 이유는 보안보다 **추적 가능성**
- **CI 스캔은 "그때", 런타임 스캔은 "지금"** — 둘 다 필요하다
- 어느 한 겹도 완벽하지 않다는 전제로 **겹쳐 쌓는다**
