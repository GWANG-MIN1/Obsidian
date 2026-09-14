# 04. app-of-apps 루트 하나 + 멀티소스 Application

> 메모: 손으로 적용하는 Application 은 root 하나. 자식 7개는 커밋으로 추가한다.
> 업스트림 차트는 Helm repo 에서, values 만 이 저장소에서. 2026-07-12 `739d1a1` · 07-14 `7c85049` · 07-15 `d932605`

---

## 상황

클러스터에 올릴 것이 7묶음이다 — sample-app, kube-prometheus-stack, loki, promtail, kyverno, kyverno-policies, external-secrets.
각각을 ArgoCD 에 어떻게 등록하고, 수천 줄짜리 업스트림 차트를 어디에 둘지 정해야 했다.

## 결정

```
kubectl apply -k gitops/bootstrap/argocd         # ① ArgoCD v2.13.4
kubectl apply -f gitops/bootstrap/root-app.yaml  # ② root → gitops/apps/ (비재귀)
                                                 #    → 자식 Application 7개가 자동 생성
```

자식 Application 은 두 모양이다.

| 모양 | 누가 | source |
|---|---|---|
| 저장소 매니페스트 | sample-app · kyverno-policies | 이 저장소의 디렉터리 |
| **업스트림 차트 + 우리 values** (멀티소스) | kube-prometheus-stack · loki · promtail · kyverno · external-secrets | 차트는 Helm repo, values 는 `$values/...` |

순서가 필요한 곳엔 sync-wave 를 걸었다 — kyverno(0) → kyverno-policies(1), loki(0) → promtail(1).
CRD 가 큰 차트 셋엔 `ServerSideApply=true`.

## 왜

- **부트스트랩이 딱 두 번이다.** 그 뒤로 워크로드 추가는 Application 매니페스트 **커밋**이지 `kubectl` 이 아니다.
- **멀티소스로 차트를 Git 밖에 둔다.** kube-prometheus-stack Application 주석 — *"6000줄 차트는 Git 밖에, 오버라이드는 리뷰 가능하게."*
- **ServerSideApply 는 선택이 아니었다.** Prometheus operator · Kyverno · ESO 의 CRD 가 client-side apply 의
  `last-applied-configuration` 어노테이션 한도(262144 바이트)를 넘는다. → [cicd-lab/05](../../../infra-lab/cicd-lab/05-argocd-advanced/README.md)

## 왜 다른 건 안 썼나

**ApplicationSet**

제너레이터로 Application 을 찍어 내는 방식이다. 여기는 컴포넌트 7개의 설정이 전부 달라 **반복이 없다.**
[cicd-lab/06](../../../infra-lab/cicd-lab/06-app-of-apps-applicationset/README.md) 의 기준 그대로 —
*"플랫폼 컴포넌트는 App of Apps(명시적), 반복되는 앱은 ApplicationSet"*.
제너레이터가 빈 결과를 내면 `prune` 과 겹쳐 전부 사라지는 위험도 굳이 가져올 이유가 없었다.

**차트를 Git 에 vendoring (`helm pull` 후 커밋)**

리뷰할 수 없는 크기가 되고, 차트를 올릴 때마다 diff 가 수천 줄이 된다.

**umbrella 차트 하나**

모든 스택을 한 차트의 values 로 묶는다. 한 컴포넌트가 막히면 전체 sync 가 막힌다.
Application 을 나눈 덕분에 **상태가 컴포넌트별로** 보였고, [트러블 03](../troubleshooting/03-Healthy인데-영원히-OutOfSync.md) 은
*"8개 중 kyverno · external-secrets 둘만 OutOfSync"* 에서 출발할 수 있었다.

## 이 결정의 대가

### ① sync-wave 가 자식을 기다리지 않는다 *(매니페스트와 ArgoCD 문서에서 읽은 것 — 클러스터에서 확인하진 않음)*

ArgoCD 는 1.8 에서 `argoproj.io/Application` 의 health 평가를 뺐다. 문서는 *app-of-apps 에서 sync wave 로 동기화를 조율한다면*
`argocd-cm` 에 Lua health check 를 복원해야 할 수 있다고 적는다. 이 저장소의 ArgoCD 는 순정 `install.yaml` 이라 그 설정이 없다.

그래서 root 입장에서 자식 Application 은 **만들어지는 즉시 건강**하고, wave 0 → 1 이 준비를 기다리지 않고 넘어간다.
정황이 셋 있다.

- `kyverno-policies.yaml` 주석이 이미 경주를 전제한다 — *"CRD 가 등록되는 중에도 첫 sync 를 진행하고(SkipDryRunOnMissingResource) 재시도로 수렴한다."*
- [트러블 04](../troubleshooting/04-root-app-직후-아무것도-안-뜬-것처럼-보였다.md) 에서 root 적용 직후 자식 7개가 **한꺼번에** 빈칸으로 생겼다.
- `apps/README` 의 *"Kyverno (wave 0) before its policies (wave 1)"* 는 **적용 순서** 수준의 보장이다.

### ② 정책이 앱보다 늦다

`sample-app.yaml` 에 wave 가 없어서(=0) 정책(wave 1)보다 **먼저** 적용된다.
[cicd-lab/06](../../../infra-lab/cicd-lab/06-app-of-apps-applicationset/README.md) 은 정책 엔진을 -1, 앱을 1 에 두라고 권한다 —
*"아니면 정책이 적용되기 전에 앱이 배포돼서 검사를 통과해버린다."*
Audit 모드에선 background 스캔이 기존 리소스까지 리포트해서 결과(PASS 5)에 영향이 없었다. Enforce 로 넘어갈 때의 전제 조건이다.
→ [결정 07](07-Kyverno를-Audit으로-착지시켰다.md)

### ③ `repoURL` 이 8곳에 박혀 있다

root-app 주석은 *"포크하면 repoURL 을 바꾸라"* 고 하는데, 실제로는 root · sample-app · kyverno-policies 의 source 3곳과
멀티소스 5개의 `ref: values` 까지 **8곳**이다.

### ④ ArgoCD 자신은 GitOps 밖이다

ArgoCD 버전 올리기는 `kustomization.yaml` 한 줄이지만, 반영은 사람이 `kubectl apply -k` 를 다시 쳐야 한다.
`gitops/apps/` 에 ArgoCD 자신을 관리하는 Application 이 없다.

## 배운 점

**선언한 순서와 기다리는 순서는 다르다.** sync-wave 는 이름만 보면 "앞 wave 가 준비되면 다음"인데,
자식 Application 사이에선 health 가 없어서 **앞 wave 가 만들어지기만 하면** 넘어간다.
8/8 이 초록이 된 건 wave 가 순서를 지켜서라기보다, 먼저 떠도 결국 수렴하도록 붙여 둔 옵션들 덕으로 보인다(추정).

**Application 을 컴포넌트별로 쪼갠 것은 진단에서 값을 했다.** 장애의 범위가 처음부터 좁혀진 채로 보인다.
