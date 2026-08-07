# 04 Kyverno

Pod Security Admission은 표준 프로파일 3종만 강제한다. 조직 고유 규칙 — **":latest 금지", "리소스 요청 필수", "서명된 이미지만"** — 은 못 한다.  
Kyverno는 이런 규칙을 **YAML로** 쓰는 정책 엔진이다. OPA/Gatekeeper와 달리 Rego 같은 별도 언어를 배우지 않아도 된다.

---

## Admission 웹훅 구조

```
kubectl apply
     │
     ▼
API 서버 ──▶ 인증 ──▶ 인가(RBAC) ──▶ Mutating 웹훅 ──▶ 스키마 검증 ──▶ Validating 웹훅 ──▶ etcd 저장
                                          │                                  │
                                     값을 바꾼다                          허용/거부
                                     (Kyverno mutate)                  (Kyverno validate)
```

| 순서가 중요한 이유 | |
|---|---|
| **Mutate가 Validate보다 먼저** | 기본값을 채워 넣은 뒤 검사한다 |
| RBAC이 웹훅보다 먼저 | 권한 없는 요청은 정책까지 오지도 않는다 |

> **admission은 `kubectl`로 우회할 수 없다.** API 서버를 거치는 모든 요청에 적용되기 때문이다. CI 검사와의 결정적 차이다. → `01-security-basics/`  
> 단, **이미 존재하는 리소스는 검사하지 않는다.** 기존 리소스는 `background: true` 스캔이 리포트만 남긴다.

---

## 정책 종류 4가지

| 종류 | 하는 일 | 예시 |
|---|---|---|
| **validate** | 조건 검사 → 허용/거부/기록 | `:latest` 금지, 비-root 요구 |
| **mutate** | 리소스 값을 수정 | 기본 라벨·sidecar·securityContext 주입 |
| **generate** | 다른 리소스를 생성 | 네임스페이스 생성 시 NetworkPolicy 자동 생성 |
| **verifyImages** | 이미지 서명 검증 | cosign 서명 없으면 거부 → `07-image-security/` |

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy          # 또는 Policy (네임스페이스 범위)
metadata:
  name: my-policy
spec:
  background: true           # 기존 리소스도 주기적으로 검사해 리포트 생성
  rules:
    - name: my-rule
      match: ...             # 어디에 적용할지
      exclude: ...           # 무엇을 뺄지
      validate: ...          # 또는 mutate / generate / verifyImages
```

---

## validate — 가장 많이 쓴다

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations:
    policies.kyverno.io/title: Disallow Latest Tag
    policies.kyverno.io/category: Supply Chain
    policies.kyverno.io/description: >-
      Requires an explicit image tag and forbids ':latest'.
spec:
  background: true
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds: [Pod]
      exclude:
        any:
          - resources:
              namespaces: &infra-namespaces      # YAML 앵커로 중복 제거
                - kube-system
                - kube-node-lease
                - kube-public
                - kyverno
                - argocd
                - observability
                - external-secrets
                - kube-bench
      validate:
        failureAction: Audit
        message: "An explicit image tag is required (untagged images are not allowed)."
        pattern:
          spec:
            containers:
              - image: "*:*"

    - name: disallow-latest-tag
      match:
        any:
          - resources:
              kinds: [Pod]
      exclude:
        any:
          - resources:
              namespaces: *infra-namespaces      # 앵커 참조
      validate:
        failureAction: Audit
        message: "Using the ':latest' tag is not allowed — pin an explicit version."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

> **`&anchor` / `*alias`는 YAML 기능**이다. 제외 네임스페이스 목록을 규칙마다 복붙하지 않아도 된다.  
> `:latest`를 막는 이유는 보안보다 **추적 가능성**이다 — 무엇이 실제로 돌았는지 알 수 없고, 재-pull로 워크로드가 조용히 바뀔 수 있다. → `../cicd-lab/04-container-image-pipeline/`

### 패턴 앵커 문법

```yaml
pattern:
  spec:
    containers:
      - securityContext:
          allowPrivilegeEscalation: false     # 필수: 반드시 이 값
          =(privileged): false                # 조건부: 있으면 false 여야
          capabilities:
            drop: [ALL]
```

| 앵커 | 의미 |
|---|---|
| (없음) | **필수** — 이 필드가 있고 값이 일치해야 |
| `=(field)` | **조건부** — 필드가 존재하면 값이 일치해야 (없으면 통과) |
| `X(field)` | **전역 조건** — 이 조건이 참일 때만 나머지 규칙 적용 |
| `^(field)` | 배열 요소 중 **하나라도** 일치하면 통과 |
| `!value` | 이 값이 **아니어야** |
| `*` | 와일드카드 |

> **`=()`와 필수의 차이가 중요하다.** `privileged: false`를 필수로 하면 이 필드를 안 쓴 모든 파드가 실패한다 — 기본값이 false인데도. `=(privileged)`로 두면 "명시했다면 false여야 한다"가 된다.

### deny — 더 복잡한 조건

`pattern`으로 표현이 안 될 때 쓴다.

```yaml
validate:
  failureAction: Enforce
  message: "승인된 레지스트리의 이미지만 사용할 수 있습니다."
  deny:
    conditions:
      any:
        - key: "{{ request.object.spec.containers[].image }}"
          operator: AnyNotIn
          value: ["123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/*"]
```

---

## Audit → Enforce

```yaml
validate:
  failureAction: Audit      # 기록만 (Kyverno 1.13+ 필드명)
  # failureAction: Enforce  # 거부
```

> 예전 필드명은 `spec.validationFailureAction`이었다. **1.13부터 규칙별 `validate.failureAction`으로 옮겨졌다** — 오래된 예제와 다르므로 주의.

```
새 정책은 전부 Audit 으로 착지시킨다
      ↓
PolicyReport 를 읽는다 → 무엇이 깨질지 미리 본다
      ↓
워크로드를 고치거나 정책을 현실에 맞게 조정
      ↓
리포트가 깨끗해진 정책부터 하나씩 Enforce 로
```

> **살아있는 클러스터에 Enforce를 바로 켜는 것이 플랫폼 팀이 신뢰를 잃는 가장 빠른 길이다.**  
> 4개 정책을 한꺼번에 바꾸지 않는다. **정책별로** 전환한다.

---

## 실제 4개 정책

```
disallow-latest-tag       명시적 이미지 태그 필수, :latest 금지
require-resources         CPU/메모리 requests + 메모리 limit
require-run-as-non-root   runAsNonRoot=true
restrict-privileges       권한 상승 금지, privileged 금지, capability 전부 drop
```

```yaml
# require-resources — CPU limit 은 일부러 요구하지 않는다
# requests 는 스케줄러가 노드를 제대로 채우게 하고,
# 메모리 limit 은 새는 파드가 이웃을 OOM 시키는 걸 막는다.
# CPU limit 은 버스티한 워크로드를 스로틀링할 뿐 격리 이점이 없다.
```

> **"보안 정책"이라고 전부 보안만 담는 게 아니다.** `require-resources`는 카테고리가 Reliability다 — 리소스 미설정은 보안 문제가 아니라 안정성 문제지만, 같은 admission 게이트에서 막는 게 효율적이다.  
> CPU limit을 요구하지 않는 판단은 관측성 스택에서와 같다. **정책이 조직의 판단을 코드로 담는다.** → `../observability-lab/08-kube-prometheus-stack/`

---

## PolicyReport로 확인하기

```bash
kubectl get policyreport -A
kubectl get clusterpolicyreport

# 실패 항목만 추리기
kubectl -n myapp get policyreport -o json \
  | jq -r '.items[].results[] | select(.result=="fail") | "\(.policy)/\(.rule): \(.message)"'
```

### 대조 실험 — 정책이 실제로 도는지 확인하는 법

```bash
# 일부러 4개 정책을 전부 위반하는 파드
kubectl run bad-pod --image=nginx:latest
#   :latest + 리소스 미지정 + root 실행 + 권한 미제한

kubectl get policyreport -A | grep bad-pod
```

```
sample-app  : PASS 4 / FAIL 0     ← 통과하도록 만든 워크로드
bad-pod     : PASS 1 / FAIL 4     ← 일부러 위반시킨 파드
```

> **"정책을 배포했다"와 "정책이 실제로 판정한다"는 다르다.** 통과 사례만 보면 정책이 아예 매칭되지 않고 있어도 알 수 없다.  
> **위반 파드를 일부러 만들어 FAIL이 찍히는 걸 확인**해야 Audit 리포트가 신뢰할 수 있는 근거가 된다. Enforce로 켰을 때 실제로 막힐 것이라는 증거이기도 하다. 확인 후 `kubectl delete pod bad-pod`.

---

## mutate — 기본값 주입

```yaml
rules:
  - name: add-default-securitycontext
    match:
      any:
        - resources:
            kinds: [Pod]
    mutate:
      patchStrategicMerge:
        spec:
          securityContext:
            +(runAsNonRoot): true          # +() : 없을 때만 추가
          containers:
            - (name): "*"
              securityContext:
                +(allowPrivilegeEscalation): false
```

| 앵커 | 의미 |
|---|---|
| `+(field)` | 필드가 **없을 때만** 추가 (있으면 그대로 둔다) |
| `(field)` | 매칭 조건 (이 값과 일치하는 요소에 적용) |

> **mutate는 강력하지만 조심스럽게 쓴다.** 개발자가 쓴 매니페스트와 실제 배포된 것이 달라지므로, 디버깅할 때 혼란이 생긴다.  
> "왜 내 파드에 이 라벨이 붙어 있지?"의 답을 찾기 어렵다. **validate로 거부하고 이유를 알려주는 편이 교육적**인 경우가 많다.

---

## generate — 리소스 자동 생성

```yaml
rules:
  - name: default-deny-networkpolicy
    match:
      any:
        - resources:
            kinds: [Namespace]
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: default-deny
      namespace: "{{request.object.metadata.name}}"
      synchronize: true             # 원본이 바뀌면 생성된 것도 갱신
      data:
        spec:
          podSelector: {}
          policyTypes: [Ingress, Egress]
```

> 네임스페이스를 만들면 **default-deny NetworkPolicy가 자동으로 붙는다.** 안전한 기본값을 조직 차원에서 보장하는 방법이다. → `08-network-security/`

---

## 정책 테스트

```bash
# 클러스터 없이 매니페스트 검사 (CI 에서 쓴다)
kyverno apply security/kyverno/policies/ --resource bad-pod.yaml
kyverno apply security/kyverno/policies/ --resource manifests/ --policy-report

# 정책 단위 테스트
kyverno test ./tests
```

```yaml
# tests/kyverno-test.yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
policies:
  - ../policies/disallow-latest-tag.yaml
resources:
  - resources.yaml
results:
  - policy: disallow-latest-tag
    rule: disallow-latest-tag
    resource: bad-pod
    kind: Pod
    result: fail
  - policy: disallow-latest-tag
    rule: disallow-latest-tag
    resource: good-pod
    kind: Pod
    result: pass
```

> **정책도 코드다.** CI에서 `kyverno test`를 돌리면 "정책을 고쳤는데 아무것도 안 걸리게 됐다"를 잡을 수 있다. → `../cicd-lab/03-github-actions-advanced/`

---

## 운영 주의사항

| 항목 | 주의 |
|---|---|
| **웹훅 장애** | Kyverno가 죽으면 admission이 실패할 수 있다 (`failurePolicy`) |
| CRD가 크다 | ArgoCD 배포 시 `ServerSideApply=true` 필요 → `../cicd-lab/05-argocd-advanced/` |
| 순서 | 정책 엔진이 앱보다 먼저 떠야 한다 (sync-wave) |
| 리소스 | admission 컨트롤러는 모든 요청 경로에 있다 — 여유 있게 |
| 제외 목록 | "안 되니까 뺐다"가 쌓이면 정책이 껍데기 |

```yaml
# failurePolicy: Kyverno 가 응답 못 할 때
#   Ignore : 통과시킨다 (가용성 우선, 보안 구멍)
#   Fail   : 거부한다 (보안 우선, 클러스터가 멈출 수 있다)
```

> **Kyverno가 죽으면 클러스터 전체 배포가 막힐 수 있다.** 컨트롤러를 여러 레플리카로 띄우고, 처음에는 `Ignore`로 시작해 안정화 후 `Fail`로 옮기는 게 안전하다.

---

## 배운 점

- admission 웹훅은 **Mutate → 스키마 검증 → Validate** 순서로 동작한다
- **admission은 `kubectl`로 우회할 수 없다** — CI 검사와의 결정적 차이
- 단, **이미 존재하는 리소스는 검사하지 않는다** (`background: true`가 리포트만 생성)
- 정책 4종: **validate / mutate / generate / verifyImages**
- YAML 앵커(`&`/`*`)로 제외 네임스페이스 목록 중복을 없앤다
- **`=()`는 조건부 앵커** — "있으면 이 값이어야" (필드를 필수로 만들지 않는다)
- `+()`는 mutate에서 "없을 때만 추가"
- **필드명이 `validate.failureAction`으로 바뀌었다** (1.13+, 이전은 `validationFailureAction`)
- **새 정책은 전부 Audit으로 착지**시키고 정책별로 Enforce 전환
- 보안 정책에 안정성 규칙(`require-resources`)이 섞여도 된다 — **같은 게이트가 효율적**
- 정책은 **조직의 판단을 코드로** 담는다 (CPU limit 미요구 같은)
- ⭐ **"배포했다"와 "실제로 판정한다"는 다르다** — 위반 파드를 일부러 만들어 FAIL을 확인
- 대조 실험(PASS 4 vs FAIL 4)이 Audit 리포트를 신뢰할 근거가 된다
- **mutate는 매니페스트와 실물이 달라져** 디버깅을 어렵게 한다 — validate 쪽이 교육적
- `generate`로 네임스페이스 생성 시 default-deny NetworkPolicy를 자동 부착할 수 있다
- `kyverno apply`·`kyverno test`로 **클러스터 없이 CI에서 정책을 검증**한다
- **Kyverno가 죽으면 배포가 막힐 수 있다** — `failurePolicy`는 `Ignore`에서 시작
