# 03. Healthy 인데 영원히 OutOfSync — 클러스터가 모르는 필드

- **발생**: 2026-07-20 · Phase 2 검증 중 (수정 커밋 21:31 `c639992` · 21:35 `3bbdfb6`)
- **증상 한 줄**: kyverno(CRD 11개) · external-secrets(CRD 1개) Application 이 `Healthy` 인데 `OutOfSync` 에서 안 빠짐. hard refresh 로도 안 됨
- **원문**: 저장소 [`docs/troubleshooting/03-crd-permanent-outofsync.md`](https://github.com/GWANG-MIN1/eks-gitops-platform/blob/main/docs/troubleshooting/03-crd-permanent-outofsync.md)

---

## 현상

Application 8개 중 두 개만 노란 상태로 굳었다. 워크로드는 멀쩡히 돌았고 에러도 없었다.
**아무것도 안 깨졌는데 Git 과 클러스터가 영원히 다르다고 나오는** 상태였다.

## 환경

- EKS 1.30 (`Server: v1.30.14-eks`)
- kyverno 차트 3.8.2 (Kyverno v1.18) · external-secrets 차트 2.7.0 — 둘 다 고정한 날(07-16) 기준 최신 릴리스
- 두 Application 모두 `ServerSideApply=true` (CRD 가 커서) → [결정 04](../decisions/04-app-of-apps-루트-하나와-멀티소스.md)

## 진단 — 가설 → 반증 → 대조

| 단계 | 한 것 | 결과 |
|---|---|---|
| 1 | OutOfSync 리소스만 jsonpath 로 추림 | 전부 CRD, kyverno 는 `policies.kyverno.io` 그룹만 |
| 2 | **가설: 웹훅이 CRD 에 caBundle 을 주입해서 생긴 차이** | ❌ 라이브 CRD 의 `conversion.webhook.clientConfig.caBundle` 이 비어 있음 → **반증** |
| 3 | ArgoCD UI 의 DIFF 탭 | `selectableFields` 가 한쪽에만 있음 · `labels: {}` / `annotations: {}` 도 차이로 표시 |
| 4 | 차트 tgz 속 CRD 와 라이브 CRD 를 `grep -c selectableFields` 로 대조 | 차트 **3** · 라이브 **0** |

2단계가 중요했다. caBundle 주입은 흔히 알려진 드리프트 원인이라 먼저 의심했는데, **확인 명령 한 줄로 기각하고 실제 diff 로 갔다.**
붙들고 있었다면 무시 규칙을 엉뚱한 경로에 걸었을 것이다.

## 원인 — 두 개가 겹쳐 있었다

### ① 1.30 API 서버가 `spec.versions[].selectableFields` 를 버린다

저장소 문서는 이 필드를 *"1.31+ 기능"* 이라고 적었다. 틀린 말은 아닌데, Kubernetes 문서의 feature gate
`CustomResourceFieldSelectors` 를 보면 정확히는 이렇다.

| 버전 | 단계 | 기본값 |
|---|---|---|
| 1.30 | 알파 | **꺼짐** |
| 1.31 | 베타 | 켜짐 |
| 1.32 ~ | GA | 켜짐 |

1.30 에서는 기능 게이트가 꺼져 있고, **EKS 는 알파 기능을 지원하지 않는다**(AWS 문서). 그래서 API 서버가 저장하기 전에 필드를 떨군다.
차트에는 있고 클러스터에는 영원히 없으니 diff 가 사라질 수 없다. 그리고 **에러는 한 줄도 없다.**

### ② kyverno-api 서브차트가 CRD metadata 에 빈 맵을 그대로 렌더링한다

`labels: {}` · `annotations: {}` — API 서버는 빈 맵을 "필드 없음"으로 저장하고, server-side apply 비교는 그걸 영구 차이로 본다.
external-secrets 는 ① 만 무시해도 Synced 가 됐고, kyverno 는 ② 까지 무시해야 됐다. 커밋이 21:31 과 21:35 두 개인 이유다.

## 해결

```yaml
ignoreDifferences:
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    jqPathExpressions:
      - .spec.versions[].selectableFields
    jsonPointers:                 # kyverno 만 — 차트가 실제 값을 넣지 않는 필드라 잃는 것 없음
      - /metadata/labels
      - /metadata/annotations
syncPolicy:
  syncOptions:
    - RespectIgnoreDifferences=true
```

`RespectIgnoreDifferences` 가 짝으로 필요하다. diff 에서만 빼면 sync 할 때마다 그 필드를 다시 적용하고
API 서버가 다시 버리는 헛바퀴가 돈다. 두 커밋 뒤 21:41 에 **8/8 Synced/Healthy**.

## 재발 방지

✅ **반영됐고, 방식이 좋았다.** 매니페스트 주석에 *왜 무시하는지*(검증된 원인 두 개)와 *언제 지울지*(클러스터가 1.31+ 가 되면)를 같이 적었다.
`ignoreDifferences` 는 쌓이기 쉬운 설정이라 **제거 조건을 적어 두는 것**이 핵심이다.

⚠️ **그런데 그 "언제"가 강제로 왔다.** 1.30 은 2026-07-23 에 연장 지원이 끝나 이제 새 클러스터를 못 만든다.
다음 클러스터는 1.31 이상일 수밖에 없고, 그 순간 `selectableFields` 항목은 **아무것도 안 가리는 잔재**가 된다
(빈 맵 항목은 차트가 고치기 전까지 유효하다). → [결정 03](../decisions/03-전부-버전을-고정했다.md)

그리고 **근본 원인 — 최신 차트 × 오래된 쿠버네티스 — 은 안 고쳤다.** 차트를 올릴 때마다 같은 종류의 필드가 또 나올 수 있다.

## 배운 점

- **API 서버는 모르는(꺼진) 필드를 에러 없이 버린다.** 그래서 버전 skew 는 실패가 아니라 *영구 OutOfSync* 라는 조용한 형태로 나타난다.
  → [cicd-lab/05](../../../infra-lab/cicd-lab/05-argocd-advanced/README.md) *"🔧 실제로 겪은 것 — CRD 가 영원히 OutOfSync"*
- **흔한 답을 먼저 반증하는 게 가장 빨랐다.** 가설을 확인하는 명령이 한 줄이면, 그 한 줄을 먼저 친다.
- **"1.31+ 기능"은 조건이 빠진 문장이다.** 필드는 1.30 에도 있고 기본으로 켜진 게 1.31 이며, 관리형 K8s 에서는 *"알파는 없다"* 가 조건으로 하나 더 붙는다.
  [결정 06](../decisions/06-컨트롤플레인-수집을-끄고-알림은-3개만.md) 의 *"스크레이프 불가"* 와 같은 종류의 문장이다.
- **방치했다면 알림 피로의 입구였다.** 영구 OutOfSync 가 남으면 `ArgoCDAppNotSynced` 가 늘 참이 된다. 그날 안에 고친 게 맞았다.
