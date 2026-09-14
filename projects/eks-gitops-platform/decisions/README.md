# Decisions

eks-gitops-platform 에서 **무엇을 왜 그렇게 정했는가**의 기록.
형식과 원칙은 [`../../gym-management-db/decisions/README.md`](../../gym-management-db/decisions/README.md) 와 같다.

```
## 상황              무엇을 정해야 했는가
## 결정              무엇을 택했는가
## 왜                근거 — 확인한 사실 기준으로
## 왜 다른 건 안 썼나   버린 선택지와 그 이유
```

## 이 프로젝트에서 특히 남길 만한 것

- **01 · 10** 은 한 경계의 양쪽이다. Terraform 이 IAM 역할을 만들고(01) GitOps 가 ServiceAccount 를 만들면
  **역할 ARN 이 둘 사이를 건너는 인계점**이 된다. 그 인계가 계정 종속이라 Git 에 못 들어가고
  사람 손(`kubectl annotate`)에 남았다(10).
- **03** 이 이 묶음의 중심이다. 버전 고정은 재현성을 위해서였는데, **고정한 쿠버네티스 버전이 늙어서**
  [트러블 03](../troubleshooting/03-Healthy인데-영원히-OutOfSync.md) 의 원인이 됐고, 두 달 뒤엔 기본값으로 클러스터를 만들 수 없게 됐다.
- **06 · 07 · 08** 은 전부 **"있다"와 "작동한다" 사이에서 의도적으로 약한 쪽을 고른** 것들이다 —
  알림은 있는데 받는 곳이 없고, 정책은 있는데 Audit 이고, 스캔은 있는데 CRITICAL 만 막는다.
  uptime-monitor [결정 09](../../serverless-uptime-monitor/decisions/09-tfsec을-soft-fail로.md) (tfsec soft-fail)와 같은 계열인데,
  08 은 **첫 push 에서 실제로 막았다**는 점이 다르다.
- **02 · 05** 는 "매일 destroy" 가 낳은 결정들이다. 비용이 전제였기 때문에 영속성을 통째로 포기했다.

## 저장소 문서와의 관계

`docs/architecture.md` 에 "왜 이렇게 나눴나"가, values · 매니페스트 주석마다 선택 이유가 이미 길게 있다.
여기는 그걸 옮기지 않고 **버린 선택지**와 **그 결정이 나중에 무엇을 치르게 했는지**만 적는다.
저장소가 이유를 안 적은 선택(왜 1.30 인가, 왜 Pod Identity 대신 IRSA 인가)은 **이유가 안 적혀 있다**고 적었다.
