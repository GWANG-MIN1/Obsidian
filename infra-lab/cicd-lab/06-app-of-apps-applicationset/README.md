# 06 App of Apps & ApplicationSet

Application이 5개를 넘어가면 "Application을 어떻게 관리할 것인가"가 문제가 된다.  
**App of Apps는 Application을 Git으로 관리하는 방법**이고, **ApplicationSet은 Application을 자동 생성하는 방법**이다. 목적이 다르다.

---

## 문제 — 부트스트랩의 무한 반복

```
클러스터를 새로 만들 때마다:
  kubectl apply -f app1.yaml
  kubectl apply -f app2.yaml
  kubectl apply -f app3.yaml
  ...
  → 이게 이미 GitOps가 아니다. 손으로 kubectl 을 치고 있다.
```

---

## App of Apps

**Application 하나가 다른 Application들을 관리한다.** 손으로 적용하는 건 루트 하나뿐이다.

```
gitops/
├── bootstrap/
│   ├── argocd/           # ArgoCD 설치 (kustomize, 버전 고정) — 손으로 1회
│   └── root-app.yaml     # 손으로 적용하는 유일한 Application
└── apps/                 # 자식 Application 들 — 각각 하나의 관심사
    ├── sample-app.yaml
    ├── sample-app/       #   ← 그 앱의 실제 매니페스트
    ├── kube-prometheus-stack.yaml
    ├── loki.yaml
    ├── promtail.yaml
    ├── kyverno.yaml
    ├── kyverno-policies.yaml
    └── external-secrets.yaml
```

```yaml
# bootstrap/root-app.yaml — 이것 하나만 kubectl apply
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/GWANG-MIN1/eks-gitops-platform.git
    targetRevision: main
    path: gitops/apps                # ← 이 디렉터리의 *.yaml 이 자식 Application
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```
kubectl apply -f root-app.yaml
        ↓
root Application 이 gitops/apps/ 를 감시
        ↓
그 안의 각 *.yaml = 자식 Application → ArgoCD가 자동으로 관리 시작
        ↓
새 워크로드 추가 = Application 매니페스트를 커밋 (kubectl 아님)
```

### 부트스트랩은 딱 두 단계

```bash
# 1. ArgoCD 설치 (버전 고정된 kustomize)
kubectl apply -k gitops/bootstrap/argocd
kubectl -n argocd rollout status deploy/argocd-server

# 2. 열쇠를 넘긴다 — app-of-apps 루트
kubectl apply -f gitops/bootstrap/root-app.yaml
```

> **이 두 번이 클러스터 수명 전체에서 유일한 수동 `kubectl`이다.** 그 뒤로는 전부 `git push`.  
> `path: gitops/apps`는 비재귀(non-recursive)라 하위 디렉터리(`apps/sample-app/`)의 매니페스트는 루트가 직접 건드리지 않는다. 그건 `sample-app` Application의 몫이다.

### 두 가지 Application 형태

```yaml
# ① 우리 저장소의 매니페스트를 가리킨다
spec:
  source:
    repoURL: https://github.com/GWANG-MIN1/eks-gitops-platform.git
    path: gitops/apps/sample-app

# ② 업스트림 차트 + 우리 values (멀티소스) → 05-argocd-advanced/
spec:
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 87.16.1
      helm:
        valueFiles: ["$values/observability/kube-prometheus-stack/values.yaml"]
    - repoURL: https://github.com/GWANG-MIN1/eks-gitops-platform.git
      ref: values
```

### 계층에 sync-wave 걸기

```yaml
# 정책 엔진이 먼저, 앱이 나중
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"    # kyverno.yaml
```

```
wave -2: cert-manager, external-secrets   (다른 것들이 의존)
wave -1: kyverno (정책 엔진)
wave  0: 관측성 스택
wave  1: 애플리케이션
```

> **정책 엔진이 앱보다 먼저 떠야 한다.** 아니면 정책이 적용되기 전에 앱이 배포돼서 검사를 통과해버린다.

---

## ApplicationSet

App of Apps는 **Application을 손으로 쓴다**(파일마다 하나). 환경 3개 × 앱 10개 = 30개 파일이 되면 그것도 복붙이다.  
ApplicationSet은 **템플릿 + 제너레이터**로 Application을 자동 생성한다.

```
App of Apps    :  Application 을 Git 으로 관리 (파일은 사람이 쓴다)
ApplicationSet :  Application 을 자동 생성    (파일이 안 늘어난다)
```

### List 제너레이터 — 가장 단순

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-envs
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            revision: main
            replicas: "1"
          - env: prod
            revision: v1.4.2
            replicas: "3"
  template:
    metadata:
      name: "myapp-{{env}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/org/manifests.git
        targetRevision: "{{revision}}"
        path: "apps/myapp/overlays/{{env}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "myapp-{{env}}"
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

```
→ myapp-dev (main 브랜치 추적), myapp-prod (v1.4.2 태그 고정) 두 개가 자동 생성
```

### Git 제너레이터 — 디렉터리가 곧 Application

```yaml
spec:
  generators:
    - git:
        repoURL: https://github.com/org/manifests.git
        revision: main
        directories:
          - path: apps/*                 # apps/ 아래 각 디렉터리마다 하나씩
          - path: apps/experimental      # 제외
            exclude: true
  template:
    metadata:
      name: "{{path.basename}}"
    spec:
      source:
        repoURL: https://github.com/org/manifests.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
```

> **디렉터리를 추가하면 Application이 생기고, 지우면 사라진다.** 개발자가 매니페스트만 넣으면 배포되는 셀프서비스 구조가 된다.

### Cluster 제너레이터 — 멀티 클러스터

```yaml
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production      # 이 라벨이 붙은 클러스터 전부
  template:
    metadata:
      name: "monitoring-{{name}}"
    spec:
      destination:
        server: "{{server}}"
        namespace: observability
```

> 클러스터를 ArgoCD에 등록하기만 하면 **관측성 스택이 자동으로 배포된다.** 클러스터가 10개든 100개든 같다.

### Matrix 제너레이터 — 조합

```yaml
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/org/manifests.git
              directories: [{ path: "apps/*" }]
          - clusters:
              selector:
                matchLabels: { environment: production }
```

```
앱 5개 × 클러스터 3개 = Application 15개 자동 생성
```

| 제너레이터 | 용도 |
|---|---|
| **List** | 소수의 고정 환경 |
| **Git (directories)** | 디렉터리 = 앱 (셀프서비스) |
| **Git (files)** | 설정 파일(JSON/YAML)에서 값을 읽어 생성 |
| **Cluster** | 등록된 클러스터마다 |
| **Matrix** | 두 제너레이터의 조합 |
| **Merge** | 여러 제너레이터 결과를 키로 병합 |
| **Pull Request** | PR마다 미리보기 환경 |

### PR 미리보기 환경

```yaml
spec:
  generators:
    - pullRequest:
        github:
          owner: GWANG-MIN1
          repo: myapp
          labels: ["preview"]            # 이 라벨이 붙은 PR 만
        requeueAfterSeconds: 300
  template:
    metadata:
      name: "myapp-pr-{{number}}"
    spec:
      source:
        targetRevision: "{{head_sha}}"
      destination:
        namespace: "preview-{{number}}"
```

> PR을 열면 미리보기 환경이 생기고 **닫으면 자동으로 사라진다.** Terraform workspace가 원래 노리던 용도와 같은 발상이다. → `../terraform-lab/08-workspace-environment/`

---

## ⚠️ ApplicationSet의 위험

```yaml
spec:
  syncPolicy:
    applicationsSync: create-update       # 삭제는 하지 않는다
    preserveResourcesOnDeletion: true     # ApplicationSet 삭제 시 리소스 보존
```

```
제너레이터가 빈 결과를 반환하면
      ↓
생성했던 Application 이 전부 사라진다
      ↓
prune: true 면 → 클러스터 리소스도 전부 삭제  💥
```

> **Git 제너레이터의 경로 오타 하나로 운영 앱 전체가 지워질 수 있다.** 프로덕션 ApplicationSet에는 `preserveResourcesOnDeletion`을 켜거나 `applicationsSync: create-update`로 삭제를 막는다.  
> ApplicationSet 변경은 **dry-run으로 몇 개가 생성/삭제되는지 먼저 확인**한다.

```bash
kubectl -n argocd get applicationset
kubectl -n argocd describe applicationset myapp-envs      # 생성된 Application 목록
kubectl -n argocd get applications -l argocd.argoproj.io/application-set-name=myapp-envs
```

---

## 어느 것을 쓸 것인가

```
Application 5개 이하, 각각 다른 성격     →  App of Apps (파일로 명시)
같은 앱을 여러 환경/클러스터에            →  ApplicationSet
둘 다                                    →  App of Apps 안에 ApplicationSet 을 둔다
```

| | App of Apps | ApplicationSet |
|---|---|---|
| Application 정의 | 파일 하나씩 (명시적) | 템플릿 + 제너레이터 |
| 읽기 쉬움 | ✅ 뭐가 배포되는지 보인다 | △ 렌더링해봐야 안다 |
| 확장성 | △ 파일이 늘어난다 | ✅ 자동 생성 |
| 실수의 폭발 반경 | 작다 | **크다** (전부 사라질 수 있다) |
| 적합 | 플랫폼 컴포넌트 | 다수 환경·클러스터·팀 앱 |

> **플랫폼 컴포넌트(관측성·정책·시크릿)는 App of Apps가 낫다.** 개수가 적고 각각 설정이 달라서, 파일로 보이는 편이 리뷰와 디버깅에 유리하다.  
> ApplicationSet은 **같은 모양이 반복될 때** 진가가 나온다.

---

## 배운 점

- **App of Apps = Application을 Git으로 관리**, **ApplicationSet = Application을 자동 생성** — 목적이 다르다
- 손으로 적용하는 건 **루트 Application 하나뿐**, 나머지는 커밋으로 추가
- 부트스트랩은 딱 두 단계(ArgoCD 설치 + 루트 적용) — 이후는 전부 `git push`
- 새 워크로드 추가 = **Application 매니페스트 커밋** (`kubectl`이 아니라)
- 루트의 `path`는 비재귀 — 하위 디렉터리는 자식 Application이 담당
- 계층에 `sync-wave`를 걸어 **정책 엔진이 앱보다 먼저** 뜨게 한다
- ApplicationSet 제너레이터: List / Git / Cluster / Matrix / Merge / PullRequest
- **Git 제너레이터**로 "디렉터리를 넣으면 배포되는" 셀프서비스를 만들 수 있다
- **Cluster 제너레이터**로 클러스터 등록만 하면 표준 스택이 깔린다
- **PullRequest 제너레이터**로 PR 미리보기 환경을 자동 생성·삭제
- ⚠️ **제너레이터가 빈 결과를 내면 Application이 전부 사라진다** — `prune`과 겹치면 참사
- 운영에는 `preserveResourcesOnDeletion` 또는 `applicationsSync: create-update`
- ApplicationSet 변경은 **생성/삭제 개수를 먼저 확인**하고 적용한다
- 플랫폼 컴포넌트는 App of Apps(명시적), 반복되는 앱은 ApplicationSet
