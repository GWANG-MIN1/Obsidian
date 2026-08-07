# 03 Pod Security Standards

`securityContext`를 잘 쓰는 것과 **모든 파드가 그렇게 쓰도록 강제하는 것**은 다르다.  
Kubernetes는 이를 위해 표준 프로파일 3단계(**Pod Security Standards**)와 내장 강제 장치(**Pod Security Admission**)를 제공한다.

---

## PSP는 죽었다

```
PodSecurityPolicy (PSP)
  1.21 deprecated → 1.25 완전 제거
  이유: RBAC 바인딩 기반이라 "어느 정책이 적용될지" 예측이 어려웠고,
        여러 PSP가 매칭되면 알파벳 순으로 하나가 선택되는 등 동작이 혼란스러웠다
        ↓
  대체: Pod Security Admission (내장, 단순) + 정책 엔진 (Kyverno·OPA, 유연)
```

> 오래된 블로그·강의에서 PSP를 보면 그냥 넘긴다. **1.25 이상에서는 존재하지 않는다.**

---

## 세 가지 프로파일

| 프로파일 | 의미 | 대상 |
|---|---|---|
| **privileged** | 제한 없음 | 시스템 컴포넌트, CNI, 노드 에이전트 |
| **baseline** | 알려진 권한 상승을 막는 최소선 | 일반 워크로드의 하한 |
| **restricted** | 강화된 모범 사례 | **애플리케이션의 목표선** |

### baseline이 막는 것

```
hostNetwork / hostPID / hostIPC
privileged: true
hostPath 볼륨
호스트 포트
위험한 capability 추가 (기본 세트 밖)
AppArmor / seccomp 를 Unconfined 로 변경
```

### restricted가 추가로 요구하는 것

```
runAsNonRoot: true
allowPrivilegeEscalation: false
capabilities.drop: ["ALL"]   (NET_BIND_SERVICE 만 예외적으로 add 허용)
seccompProfile: RuntimeDefault 또는 Localhost
볼륨 타입 제한 (configMap, secret, emptyDir, PVC, projected 등만)
```

> **baseline은 "명백히 위험한 것을 막는" 선이고, restricted는 "안전한 기본값을 강제하는" 선이다.**  
> 대부분의 애플리케이션은 restricted를 만족할 수 있다. 만족 못 하면 대개 이미지가 root로 도는 게 원인이다. → `02-pod-container-security/`

---

## Pod Security Admission (PSA)

**네임스페이스 라벨**로 프로파일을 적용한다. 별도 설치가 필요 없다(1.23+ 내장, 1.25 GA).

```bash
kubectl label ns myapp pod-security.kubernetes.io/enforce=restricted
kubectl label ns myapp pod-security.kubernetes.io/enforce-version=v1.30
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.30
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### 세 가지 모드

| 모드 | 동작 | 용도 |
|---|---|---|
| **enforce** | 위반 파드를 **거부** | 실제 차단 |
| **audit** | 감사 로그에 기록 | 추적 |
| **warn** | `kubectl` 사용자에게 **경고 출력** | **도입 전 영향 파악** |

```bash
# 안전한 도입 순서 — warn 부터
kubectl label ns myapp pod-security.kubernetes.io/warn=restricted --overwrite
kubectl -n myapp rollout restart deploy      # 경고를 실제로 확인한다

# 깨끗해지면 enforce
kubectl label ns myapp pod-security.kubernetes.io/enforce=restricted --overwrite
```

> **`warn`을 먼저 켜는 것이 PSA판 Audit-then-Enforce다.** → `01-security-basics/`  
> `enforce-version`을 고정하지 않으면 클러스터 업그레이드 시 기준이 조용히 바뀐다. **버전을 명시한다.**

### ⚠️ enforce는 기존 파드를 건드리지 않는다

```
enforce 라벨을 붙여도
  → 이미 떠 있는 파드는 그대로 돈다
  → 다음 생성(재시작·스케일·롤아웃) 때 거부된다
        ↓
새벽에 노드가 죽어 파드가 재스케줄될 때 처음 실패를 만난다  💥
```

> 그래서 라벨을 붙인 뒤 **의도적으로 `rollout restart`를 해서 확인**해야 한다. 안 그러면 시한폭탄이 된다.

---

## PSA vs Kyverno — 무엇을 쓸 것인가

| | Pod Security Admission | Kyverno / OPA |
|---|---|---|
| 설치 | **불필요** (내장) | 별도 설치 |
| 적용 단위 | **네임스페이스 전체** | 라벨·이름·조건 등 자유 |
| 정책 종류 | 표준 3종만 | **임의 규칙** |
| 예외 처리 | 네임스페이스 단위로만 | 세밀하게 가능 |
| 값 수정(mutate) | ❌ | ✅ |
| 리소스 생성(generate) | ❌ | ✅ |
| 이미지 서명 검증 | ❌ | ✅ |
| 위반 리포트 | 감사 로그 | **PolicyReport CRD** |

```
PSA      : 파드 보안의 '바닥선'을 싸게 깐다
Kyverno  : 그 위에 조직 고유 규칙을 얹는다
         → 둘은 배타적이지 않다. 함께 쓴다.
```

> **PSA로 못 하는 것들이 실제로 많이 필요하다** — ":latest 금지", "리소스 요청 필수", "특정 레지스트리만 허용", "서명된 이미지만". 전부 표준 프로파일 밖이다.  
> 그래서 실무 구성은 대개 **PSA(baseline/restricted) + Kyverno(조직 규칙)** 조합이다. → `04-kyverno/`

---

## 도입 전략

```
1. 전수 조사     현재 클러스터가 어느 수준인지 본다
2. warn 적용     경고를 보고 무엇이 깨질지 파악
3. 워크로드 수정  securityContext 를 채운다
4. enforce 전환  네임스페이스별로 하나씩
5. 예외 명시     정당한 예외는 privileged 로 두되 이유를 기록
```

```bash
# 1) 현재 상태 전수 조사
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.securityContext.runAsNonRoot != true)
  | "\(.metadata.namespace)/\(.metadata.name)"'

kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.hostNetwork==true or .spec.hostPID==true or .spec.hostIPC==true)
  | "\(.metadata.namespace)/\(.metadata.name)"'

# 2) 라벨 현황
kubectl get ns --show-labels | grep pod-security
```

### 네임스페이스별 목표

| 네임스페이스 | 프로파일 | 이유 |
|---|---|---|
| 애플리케이션 | **restricted** | 목표선 |
| `argocd`, `observability` | baseline 또는 privileged | 업스트림 차트가 restricted를 만족 못 하는 경우가 많다 |
| `kube-system` | privileged | 시스템 컴포넌트 |
| `kube-bench` | **privileged** | `hostPID`·hostPath가 기능상 필수 |

> **보안 검사 도구가 보안 정책을 위반한다**는 역설이 실제로 나온다. kube-bench는 노드를 들여다봐야 하므로 `hostPID`와 hostPath 마운트가 필요하다.  
> 이런 예외는 **네임스페이스를 따로 파고, 왜 예외인지 매니페스트 주석에 남긴다.** 그래야 나중에 "이거 왜 열려 있지"가 안 생긴다.

---

## 자주 겪는 실패

| 에러 | 원인 | 해결 |
|---|---|---|
| `violates PodSecurity "restricted"` | securityContext 미설정 | 필수 필드 채우기 |
| `container has runAsNonRoot and image will run as root` | 이미지가 root, 또는 `USER`가 이름 지정 | `runAsUser` 숫자 지정 |
| `unrestricted capabilities` | `drop: ["ALL"]` 없음 | 추가 |
| `seccompProfile type unconfined` | seccomp 미설정 | `RuntimeDefault` |
| `restricted volume type` | hostPath 사용 | emptyDir·PVC로 교체 |
| 라벨 붙였는데 아무 일도 없음 | 기존 파드는 재생성 때 검사 | `rollout restart`로 확인 |

```bash
# 어떤 프로파일을 만족하는지 dry-run 으로 확인
kubectl label --dry-run=server ns myapp pod-security.kubernetes.io/enforce=restricted
# → 위반 중인 기존 파드 목록이 경고로 출력된다
```

> **`--dry-run=server`로 라벨을 붙여보면 현재 위반 중인 파드를 미리 알려준다.** 도입 전에 반드시 돌려본다.

---

## 배운 점

- **PSP는 1.25에서 완전히 제거됐다** — 오래된 자료의 PSP는 무시한다
- 프로파일 3종: **privileged(제한없음) / baseline(위험 차단) / restricted(모범 사례 강제)**
- baseline은 hostNetwork·privileged·hostPath 등을 막는다
- restricted는 **비-root·capability drop·seccomp**까지 요구한다
- restricted를 못 맞추는 원인은 대개 **이미지가 root로 도는 것**
- PSA는 **네임스페이스 라벨**로 적용하며 별도 설치가 필요 없다
- 모드는 **enforce(차단) / audit(기록) / warn(경고)** — **warn부터 켠다**
- **`enforce-version`을 고정**하지 않으면 클러스터 업그레이드 때 기준이 바뀐다
- ⚠️ **enforce는 기존 파드를 건드리지 않는다** — 재생성 시점에 처음 실패한다
- 라벨 적용 후 **의도적으로 `rollout restart`** 해서 확인해야 시한폭탄이 안 된다
- **`--dry-run=server`로 라벨을 붙여보면 위반 파드를 미리 알려준다**
- PSA는 표준 3종만, 조직 고유 규칙(`:latest` 금지 등)은 **Kyverno**가 필요하다
- 실무 구성은 **PSA(바닥선) + Kyverno(조직 규칙)** 조합
- 인프라·검사 도구 네임스페이스는 예외로 두되 **이유를 매니페스트에 남긴다**
