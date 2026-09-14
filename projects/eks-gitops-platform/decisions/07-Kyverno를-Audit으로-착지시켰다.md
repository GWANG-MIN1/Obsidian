# 07. Kyverno 는 Audit 으로 착지시키고 인프라 네임스페이스는 뺀다

> 메모: 정책 4개를 전부 Audit 으로. 살아 있는 클러스터에 Enforce 를 바로 켜는 건 플랫폼 팀이 신뢰를 잃는 방식.
> 대상은 우리 워크로드만. 2026-07-16 `2197e1b`, 07-23 대조 실험

---

## 상황

admission 정책을 넣으면서 세 가지를 정해야 했다.

1. **강도** — 위반을 막을지, 기록만 할지
2. **범위** — kube-system · argocd · observability 처럼 업스트림 차트가 만든 파드까지 볼지
3. **엔진** — Kyverno 냐 OPA/Gatekeeper 냐 (architecture.md 의 열어둔 질문)

## 결정

| 정책 | 규칙 수 | 내용 | 카테고리 |
|---|---:|---|---|
| `disallow-latest-tag` | 2 | 태그 필수 · `:latest` 금지 | Supply Chain |
| `require-resources` | 1 | CPU · 메모리 requests + 메모리 limit (**CPU limit 은 일부러 안 요구**) | Reliability |
| `require-run-as-non-root` | 1 | `runAsNonRoot: true` (파드 또는 컨테이너 수준) | PSS Restricted |
| `restrict-privileges` | 1 | 권한 상승 금지 · privileged 금지 · capability ALL drop | PSS Restricted |

- 전부 `validate.failureAction: Audit` — Kyverno 1.18 의 규칙별 필드 (`spec.validationFailureAction` 은 deprecated)
- 제외 네임스페이스 8개 — kube-system · kube-node-lease · kube-public · kyverno · argocd · observability · external-secrets · kube-bench
- Kyverno 컨트롤러 4종은 레플리카 1 (values 에 *"프로덕션 HA 가 아니다"* 라고 명시)

## 왜

- **착지 먼저, 조임은 나중.** security README — Audit 으로 착지 → 리포트로 무엇이 깨질지 본다 → 리포트가 깨끗한 정책부터
  **정책별로** Enforce. → [security-lab/04](../../../infra-lab/security-lab/04-kyverno/README.md) *"Audit → Enforce"*
- **업스트림 컴포넌트는 restricted 를 다 못 맞춘다.** 정책의 대상은 *우리* 워크로드고, sample-app 은 네 개를 통과하도록 미리 만들었다(07-14).
- **검사 도구가 정책을 위반한다.** kube-bench 는 hostPID · hostPath 가 기능상 필수라 제외했다. → [결정 09](09-kube-bench를-GitOps-밖에-뒀다.md)
- **엔진을 Kyverno 로 고른 이유는 기록이 없다.** architecture.md 의 질문은 답이 안 채워진 채 남았다.

## 왜 다른 건 안 썼나

**Enforce 즉시**

살아 있는 클러스터에서 무엇이 깨질지 모르는 상태로 막는다. 한 번 막히면 팀은 정책을 끄는 법부터 배운다.

**Pod Security Admission (네임스페이스 라벨)**

내장이라 설치가 필요 없다 → [security-lab/03](../../../infra-lab/security-lab/03-pod-security-standards/README.md).
하지만 PSS 수준만 강제할 수 있어서 `:latest` 금지나 resources 필수 같은 **보안 밖의 규칙**을 못 담는다.
`require-resources` 의 카테고리가 Reliability 인 이유다.

**OPA/Gatekeeper**

비교한 기록이 없다.

## 결과 — 대조 실험이 신뢰를 만들었다

07-23, Audit 상태에서 일부러 위반하는 파드를 만들었다.

| 대상 | PASS | FAIL |
|---|---:|---:|
| sample-app — Pod ×2 · Deployment · ReplicaSet **각각** | 5 | 0 |
| `kubectl run bad-pod --image=nginx:latest` | 1 | 4 |

규칙이 5개라 sample-app 은 PASS 5 다. bad-pod 의 PASS 1 은 "태그 필수" 규칙 — `nginx:latest` 는 태그가 **있으니** 통과하고,
바로 옆 규칙 "`:latest` 금지"에서 걸린다. Audit 이라 파드 생성은 허용되고 리포트에만 FAIL 이 찍혔다.

> [security-lab/04](../../../infra-lab/security-lab/04-kyverno/README.md) 에 sample-app 이 **PASS 4** 로 적혀 있던 것을
> 이 노트를 쓰며 검증 스크린샷 기준 **PASS 5** 로 고쳤다.

## 이 결정의 대가

**① Enforce 로 한 번도 안 넘어갔다.** 후속 과제 1순위로 적고 저장소가 멈췄다. 기록만 보면 *"Audit 으로 시작했다"* 와
*"Audit 에 머물렀다"* 가 구별되지 않는다. uptime-monitor [결정 09](../../serverless-uptime-monitor/decisions/09-tfsec을-soft-fail로.md) 의 tfsec soft-fail 과 같은 모양이다.

**② Enforce 의 전제가 아직 안 맞는다.** 정책(wave 1)이 앱(wave 0)보다 늦고, 자식 Application 사이의 wave 는 기다리지도 않는다.
→ [결정 04](04-app-of-apps-루트-하나와-멀티소스.md). Audit 에선 background 스캔이 **이미 떠 있는 파드까지** 리포트해서 드러나지 않았다.
Enforce 는 **admission 시점에만** 막으니, 순서가 틀리면 정책보다 먼저 만들어진 파드는 막히지 않는다.

**③ 제외 목록이 4개 파일에 복제돼 있다.** `disallow-latest-tag` 는 YAML 앵커(`&infra-namespaces`)를 쓰지만
앵커는 파일 경계를 못 넘어서, 나머지 3개는 같은 8줄을 반복한다. 네임스페이스 하나를 더하면 4곳을 고쳐야 한다.
[security-lab/04](../../../infra-lab/security-lab/04-kyverno/README.md) 가 경고한 *"'안 되니까 뺐다'가 쌓이면 정책이 껍데기"* 의 입구다.

**④ 매니페스트가 한 약속 하나가 안 지켜졌다.** sample-app 의 `readOnlyRootFilesystem: false` 에는
*"Phase 4 에서 조인다 (readOnlyRootFilesystem + emptyDir)"* 는 주석이 달려 있다. Phase 4 의 정책 4개 어디에도 그 규칙이 없고,
값도 그대로이고, 후속 과제 목록에도 없다.

## 배운 점

**"정책을 배포했다"와 "정책이 판정한다"를 가르는 방법은 위반을 일부러 만드는 것이었다.**
통과 사례(PASS 5)만으로는 정책이 매칭조차 안 되는 상태와 구별이 안 된다. bad-pod 가 FAIL 4 를 찍은 순간 리포트가 근거가 됐다.

**다만 Audit 은 착지점이지 목적지가 아니다.** Enforce 로 넘어가는 조건 — 리포트가 깨끗한가, 정책이 앱보다 먼저 뜨는가 — 을
적어 두지 않으면 착지한 자리가 그대로 영구 상태가 된다.
