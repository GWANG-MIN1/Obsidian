# 04. root-app 직후 아무것도 안 뜬 것처럼 보였다

- **발생**: 2026-07-22 · Phase 3 검증, 클러스터 재생성 직후 (기록 커밋 20:34 `e88980f`)
- **증상 한 줄**: root-app 을 적용했는데 `observability` 에 파드가 없고, 자식 앱의 SYNC/HEALTH 칸이 비어 있다가 OutOfSync · Missing · Progressing 으로 바뀜
- **원문**: 저장소 [`docs/troubleshooting/04-app-of-apps-convergence-timing.md`](https://github.com/GWANG-MIN1/eks-gitops-platform/blob/main/docs/troubleshooting/04-app-of-apps-convergence-timing.md)

---

## 현상

```
kubectl -n observability get pods    → No resources found
kubectl -n argocd get applications   → root 만 값이 있고, 자식 앱은 SYNC/HEALTH 가 빈칸
(잠시 뒤)                             → OutOfSync / Missing / Progressing
```

배포가 안 된 것처럼 보였다. **실제로는 장애가 아니었다.**

## 환경

- 07-22 재생성 클러스터 — 처음부터 노드 3대 ([트러블 01](01-노드-두-대가-파드-34개로-만석.md) 회피)
- 07-20 에 고친 Loki · CRD 수정이 이미 Git 에 있음
- ArgoCD v2.13.4 순정 설치 · 자식 Application 7개 · sync-wave 0 / 1

## 진단

"정말 멈춘 것"과 "아직 수렴 중"을 가르는 기준을 세웠다.

| 보이는 것 | 판단 |
|---|---|
| 상태가 시간이 지나며 **변한다** (빈칸 → Progressing → Synced) | 기다린다 |
| 같은 상태로 **10분 이상 정지** + `Pending` · `CrashLoop` 파드가 있다 | 그때부터 실제 진단 — 트러블 01 ~ 03 |

기다렸고, 8/8 Synced/Healthy 로 끝났다.

## 원인 — 선언형은 수렴형이다

```
root-app apply
  → (1) root 가 자식 Application 7개를 생성          SYNC/HEALTH 빈칸
  → (2) 자식마다 차트 · 매니페스트를 받아 배포 시작    Progressing · OutOfSync · Missing
  → (3) 네임스페이스와 파드 생성                      이때부터 pods 가 보인다
  → (4) 8/8 Synced/Healthy
```

kube-prometheus-stack 은 CRD 가 많아 (2) ~ (4) 에 수 분이 걸린다.

**저장소 문서가 안 적은 것 하나** — (1) 에서 자식 7개가 **한꺼번에** 생긴 이유다. sync-wave 를 걸어 뒀는데도 차례로가 아니라 동시에 나타났다.
ArgoCD 는 1.8 부터 자식 `Application` 의 health 를 부모가 평가하지 않아서, wave 는 **적용 순서**만 정하고 자식이 준비될 때까지 기다리지 않는다.
이 착시의 일부는 그 구조에서 온다. *(ArgoCD 문서와 매니페스트에서 읽은 것 — 클러스터에서 따로 확인하진 않았다)*
→ [결정 04](../decisions/04-app-of-apps-루트-하나와-멀티소스.md)

## 해결

기다렸다. 조급하면 `argocd.argoproj.io/refresh=normal` 어노테이션으로 기본 3분 폴링을 앞당길 수 있다 — 관찰을 빠르게 할 뿐 해결은 아니다.

## 재발 방지

코드로 막을 문제는 아니다. 대신 **판단 기준**을 문서로 남겼다.

코드로 할 수 있었던 것은 **기다리는 일의 자동화**다. Makefile 의 `argocd-root` 는 root 를 apply 하고 바로 끝나서,
지금의 대기 방식은 사람이 `get applications` 를 반복해서 치는 것이다.
`kubectl wait --for=jsonpath=...` 로 8개가 Synced/Healthy 가 될 때까지 기다리는 타깃이 있었다면 착시를 볼 일 자체가 없었다.

## 배운 점

- **GitOps 에서 "즉시 반영"을 기대하지 않는다.** 중간 상태는 알림처럼 보일 뿐, 대부분 더 기다리면 되는 상황이다.
- **매일 재생성하는 워크플로우라 이 착시는 한 번이 아니라 세션마다 온다.** 그래서 판단 기준을 문서로 남긴 게 맞았다. → [결정 02](../decisions/02-매일-destroy를-전제로-설계했다.md)
- **알림 쪽은 이미 대비돼 있었다.** `ArgoCDAppNotSynced` · `ArgoCDAppUnhealthy` 의 `for: 15m` 은 수렴에 걸리는 수 분보다 길다.
  이 착시가 알림으로 번지지 않게 하는 장치다 — 알림이 도착할 곳이 있었다면. → [결정 06](../decisions/06-컨트롤플레인-수집을-끄고-알림은-3개만.md)
