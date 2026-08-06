# 05 ArgoCD 심화

GitOps의 기본 개념(Git이 진실의 원천, pull 방식 동기화)은 앞에서 다뤘다. → `../k8s-manifests/10-helm-gitops/`  
이 장은 실제로 운영할 때 필요한 것들 — **동기화 옵션, 순서 제어, 커다란 CRD, 영원히 안 없어지는 OutOfSync** — 를 다룬다.

---

## Application 구조

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app
  namespace: argocd                 # Application 은 argocd 네임스페이스에 산다
  finalizers:
    - resources-finalizer.argocd.argoproj.io    # 삭제 시 하위 리소스도 정리
spec:
  project: default

  source:                           # 무엇을
    repoURL: https://github.com/GWANG-MIN1/eks-gitops-platform.git
    targetRevision: main
    path: gitops/apps/sample-app

  destination:                      # 어디에
    server: https://kubernetes.default.svc
    namespace: sample-app

  syncPolicy:                       # 어떻게
    automated:
      prune: true                   # Git 에서 지운 리소스를 클러스터에서도 삭제
      selfHeal: true                # 수동 kubectl 변경을 Git 기준으로 되돌림
    syncOptions:
      - CreateNamespace=true        # 대상 네임스페이스가 없으면 만든다
```

| 필드 | 의미 |
|---|---|
| `finalizers` | 이게 없으면 Application을 지워도 **배포된 리소스가 클러스터에 남는다** |
| `targetRevision` | 브랜치·태그·커밋 SHA. 운영은 태그 고정을 권장 |
| `prune` | Git에서 삭제 → 클러스터에서도 삭제 |
| `selfHeal` | 드리프트 자동 복구 |

> **`finalizers`를 빠뜨리면 Application 삭제 후 고아 리소스가 남는다.** 반대로 finalizer 때문에 삭제가 안 걸릴 때는 강제로 제거해야 한다.

```bash
kubectl -n argocd patch app sample-app -p '{"metadata":{"finalizers":null}}' --type merge
```

---

## targetRevision 선택

| 값 | 동작 | 적합한 곳 |
|---|---|---|
| `main` | 브랜치 최신을 따라간다 | dev |
| `v1.4.2` | 태그 고정 | **prod** |
| `a1b2c3d...` | 커밋 고정 | 가장 엄격 |
| `HEAD` | 기본 브랜치 | 비권장 (모호하다) |

> **dev는 브랜치를 따라가고, prod는 태그를 올려서 승격한다.** 이게 GitOps에서 환경 승격을 표현하는 가장 단순한 방법이다. → `07-gitops-repo-strategy/`

---

## 멀티소스 Application

**차트는 업스트림에서, values는 우리 저장소에서** 가져오는 패턴이다. 6000줄짜리 차트를 Git에 복사하지 않아도 된다.

```yaml
spec:
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: 87.16.1              # 차트 버전 고정
      helm:
        releaseName: kube-prometheus-stack
        valueFiles:
          - $values/observability/kube-prometheus-stack/values.yaml
          - $values/observability/kube-prometheus-stack/alerts.yaml

    - repoURL: https://github.com/GWANG-MIN1/eks-gitops-platform.git
      targetRevision: main
      ref: values                          # ← 위에서 $values 로 참조되는 이름
```

```
소스 1: 업스트림 Helm 차트 (Git 에 없다)
소스 2: 우리 저장소 (ref: values 로 이름표를 단다)
        └─ 다른 소스에서 $values/경로 로 참조
```

| 이점 | 설명 |
|---|---|
| 차트가 Git에 안 들어온다 | 저장소가 가벼워지고 diff가 읽힌다 |
| **오버라이드만 리뷰된다** | PR에서 우리가 바꾼 것만 보인다 |
| 차트 버전을 명시적으로 올린다 | `targetRevision` 한 줄 = 업그레이드 PR |

> values 파일을 여러 개 나열하면 **순서대로 병합**된다. "스택 튜닝(values.yaml)"과 "무엇을 문제로 볼 것인가(alerts.yaml)"를 분리해두면 리뷰 대상이 명확해진다. → `../observability-lab/08-kube-prometheus-stack/`

---

## Sync Options

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true          # 네임스페이스 자동 생성
    - ServerSideApply=true          # 큰 CRD 대응 (아래 참조)
    - ApplyOutOfSyncOnly=true       # 변경된 리소스만 apply (대규모에서 빠르다)
    - PruneLast=true                # 삭제를 맨 마지막에 (의존 리소스 보호)
    - RespectIgnoreDifferences=true
    - Validate=false                # kubectl 검증 생략 (CRD 순환 문제 회피)
```

### ⚠️ ServerSideApply — 큰 CRD를 만나면 반드시

```
client-side apply 는 전체 매니페스트를
  kubectl.kubernetes.io/last-applied-configuration 어노테이션에 넣는다
      ↓
어노테이션 크기 제한 262144 바이트
      ↓
Prometheus Operator·Kyverno 같은 거대한 CRD 는 이 한도를 넘는다  💥
```

```yaml
syncOptions:
  - ServerSideApply=true
```

> **`metadata.annotations: Too long` 에러를 보면 이것부터 의심한다.** 서버 사이드 어플라이는 API 서버가 필드 소유권을 관리하므로 어노테이션에 전체 스펙을 넣지 않는다.  
> kube-prometheus-stack, Kyverno, cert-manager처럼 CRD가 큰 차트에는 사실상 필수다.

---

## Sync Wave와 Hook — 순서 제어

ArgoCD는 기본적으로 리소스를 병렬 적용한다. 순서가 필요할 때 쓴다.

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"     # 낮은 숫자가 먼저 (기본 0, 음수 가능)
```

```
wave -1: Namespace, CRD, ConfigMap, Secret
wave  0: Deployment, Service            ← 기본값
wave  1: Ingress, HPA
wave  2: 스모크 테스트 Job
```

### Hook — 배포 전후 작업

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync                    # 동기화 '전'에 실행
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # 성공하면 Job 삭제
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp:1.4.2
          command: ["./migrate.sh"]
```

| Hook | 시점 |
|---|---|
| `PreSync` | 동기화 전 — **DB 마이그레이션** |
| `Sync` | 동기화 중 |
| `PostSync` | 동기화 후 — 스모크 테스트, 캐시 워밍 |
| `SyncFail` | 동기화 실패 시 — 정리·알림 |

| delete-policy | 동작 |
|---|---|
| `HookSucceeded` | 성공하면 삭제 |
| `HookFailed` | 실패하면 삭제 |
| `BeforeHookCreation` | 다음 실행 직전에 이전 것 삭제 (**로그를 남기고 싶을 때**) |

> **PreSync 훅이 실패하면 동기화 자체가 진행되지 않는다.** DB 마이그레이션이 깨진 채로 새 버전이 뜨는 걸 막아준다.  
> 훅 Job은 **멱등해야 한다.** 재동기화 때마다 다시 실행될 수 있다.

---

## 드리프트와 selfHeal

```
누군가 kubectl edit deploy/myapp 으로 replicas 를 5로 변경
        ↓
ArgoCD 가 Git(3) 과 클러스터(5) 의 차이를 감지 → OutOfSync
        ↓
selfHeal: true  →  자동으로 3으로 되돌린다
selfHeal: false →  OutOfSync 로 표시만, 사람이 판단
```

> **selfHeal은 "긴급 수동 조치"도 되돌린다.** 장애 대응 중 `kubectl scale`로 늘린 레플리카가 몇 초 뒤 원복되면 당황한다.  
> 긴급 상황에는 Application의 자동 동기화를 잠시 끄거나(`argocd app set --sync-policy none`), Git에 바로 커밋하는 쪽이 GitOps에 맞다.

### 의도적으로 무시할 차이

```yaml
spec:
  ignoreDifferences:
    # HPA 가 관리하는 필드는 드리프트로 보지 않는다
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

    # 웹훅이 주입하는 caBundle
    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jqPathExpressions:
        - .webhooks[].clientConfig.caBundle
```

> HPA를 쓰면서 `spec.replicas`를 Git에 두면 **HPA와 ArgoCD가 서로 값을 되돌리는 싸움**이 난다. `ignoreDifferences`로 빼거나, 매니페스트에서 `replicas`를 아예 제거한다. → `../k8s-manifests/09-scheduling/`

---

## 🔧 실제로 겪은 것 — CRD가 영원히 OutOfSync

```
증상: kyverno(CRD 11개), external-secrets(CRD 1개) Application 이
      Healthy 인데 OutOfSync 에서 안 빠짐. hard refresh 로도 해소 안 됨.
```

**진단 순서**

```bash
# 1) 어떤 리소스가 걸렸는지 좁힌다
kubectl -n argocd get application kyverno -o jsonpath=\
"{range .status.resources[?(@.status=='OutOfSync')]}{.kind}/{.name}{'\n'}{end}"
# → 전부 CustomResourceDefinition

# 2) 추측하지 말고 실제 diff 를 본다 (ArgoCD UI 의 DIFF 탭 또는)
argocd app diff kyverno

# 3) 원본 대조 — 차트(desired) vs 라이브(cluster)
kubectl get crd <name> -o yaml | grep -c selectableFields   # → 0
kubectl version                                              # Server: v1.30.x-eks
```

**원인:** `spec.versions[].selectableFields`는 **Kubernetes 1.31+ 기능**이다. 차트는 이 필드를 포함해 배포하는데, 1.30 API 서버는 **모르는 필드를 조용히 버린다.** 그래서 desired에는 있고 live에는 없는 diff가 영구적으로 남는다.

| 교훈 | |
|---|---|
| **추측을 버리고 실제 diff를 본다** | 1차 가설(caBundle 주입)은 반증됐다 |
| **차트가 요구하는 k8s 버전을 확인한다** | 차트 버전 ≠ 클러스터 버전 호환 |
| 해결책 | 클러스터 업그레이드 / 차트 다운그레이드 / `ignoreDifferences` |

> **영구 OutOfSync는 알림 피로를 만든다.** `ArgoCDAppNotSynced` 알림이 계속 발화하면 진짜 드리프트를 놓친다. → `../observability-lab/05-alerting/`

---

## AppProject — 경계 긋기

`default` 프로젝트는 무엇이든 어디에나 배포할 수 있다. 팀이 늘면 제한이 필요하다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/org/team-a-manifests.git      # 이 저장소만

  destinations:
    - server: https://kubernetes.default.svc
      namespace: team-a-*                              # 이 네임스페이스만

  clusterResourceWhitelist: []                         # 클러스터 범위 리소스 금지

  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota                              # 쿼터는 못 건드린다

  roles:
    - name: developer
      policies:
        - p, proj:team-a:developer, applications, sync, team-a/*, allow
```

> **`clusterResourceWhitelist: []`가 중요하다.** 비워두면 ClusterRole·CRD 같은 클러스터 범위 리소스를 만들 수 없다 — 앱 팀이 권한을 스스로 올리는 걸 막는다.

---

## 자주 겪는 문제

| 증상 | 원인·해결 |
|---|---|
| Application 삭제해도 리소스가 남음 | `finalizers` 누락 |
| `metadata.annotations: Too long` | `ServerSideApply=true` |
| 영구 OutOfSync (CRD) | 클러스터 버전이 필드를 미지원 → 실제 diff 확인 |
| replicas가 계속 되돌아감 | HPA와 충돌 → `ignoreDifferences` |
| Git을 바꿨는데 반영이 늦음 | 기본 폴링 3분 → **웹훅** 설정 |
| 네임스페이스가 없다고 실패 | `CreateNamespace=true` |
| Helm 차트가 렌더링 실패 | `argocd app manifests <app>` 로 렌더 결과 확인 |
| 삭제 순서 때문에 실패 | `PruneLast=true` |

```bash
# 캐시된 매니페스트를 무시하고 강제 새로고침
kubectl -n argocd annotate app sample-app argocd.argoproj.io/refresh=hard --overwrite

# 컨트롤러 로그
kubectl -n argocd logs deploy/argocd-application-controller -f
kubectl -n argocd logs deploy/argocd-repo-server -f     # 렌더링 문제는 여기
```

> **폴링 간격이 기본 3분이다.** "푸시했는데 안 바뀐다"의 대부분은 그냥 아직 안 온 것이다. 즉시 반영이 필요하면 GitHub 웹훅을 ArgoCD에 연결한다.

---

## 배운 점

- **`finalizers`가 없으면 Application 삭제 후 리소스가 고아로 남는다**
- `targetRevision`은 **dev는 브랜치, prod는 태그** — 태그를 올리는 게 승격
- **멀티소스 Application**으로 차트는 업스트림에서, values는 우리 저장소에서
- values 파일은 나열 순서대로 병합 — 튜닝과 알림 정의를 파일로 분리하면 리뷰가 쉽다
- **거대한 CRD에는 `ServerSideApply=true`** — client-side는 어노테이션 262144B 한도에 걸린다
- `sync-wave`로 적용 순서를 제어 (CRD·네임스페이스는 음수 wave)
- **PreSync 훅으로 DB 마이그레이션** — 실패하면 동기화 자체가 멈춘다
- 훅 Job은 **멱등해야** 한다 (재동기화 때 다시 실행된다)
- **selfHeal은 긴급 수동 조치도 되돌린다** — 급하면 자동 동기화를 잠시 끈다
- HPA와 `spec.replicas`가 충돌하면 `ignoreDifferences`로 뺀다
- **영구 OutOfSync는 추측하지 말고 실제 diff를 본다** — 차트가 요구하는 k8s 버전 확인
- 클러스터가 모르는 필드는 **조용히 버려져** 영구 diff가 된다 (`selectableFields`는 1.31+)
- 영구 OutOfSync는 알림 피로를 만들어 진짜 드리프트를 가린다
- **AppProject의 `clusterResourceWhitelist: []`** 로 권한 상승을 막는다
- 폴링 기본 3분 — "안 바뀐다"의 대부분은 아직 안 온 것. 급하면 웹훅
