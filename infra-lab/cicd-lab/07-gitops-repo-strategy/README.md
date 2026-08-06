# 07 GitOps 저장소 전략

CI는 이미지를 만들고, CD는 Git을 본다. 그 사이를 잇는 건 **"새 이미지 태그를 어떻게 매니페스트에 반영하는가"** 다.  
여기서 저장소를 어떻게 나눌지, 태그를 누가 갱신할지, 환경 간 승격을 어떻게 표현할지가 결정된다.

---

## 저장소 분리 — 앱 코드 vs 매니페스트

```
[ 단일 저장소 (mono) ]
myapp/
├── src/
├── Dockerfile
└── k8s/            # 매니페스트도 같이

[ 분리 (별도 저장소) ]
myapp/              # 앱 코드
myapp-manifests/    # 배포 매니페스트
```

| | 단일 저장소 | 분리 |
|---|---|---|
| 코드-매니페스트 원자적 변경 | ✅ 한 PR로 | ❌ 두 PR |
| CI 무한 루프 위험 | **있다** (아래 참조) | 없다 |
| 배포 권한 분리 | 어렵다 | ✅ 매니페스트 저장소만 제한 |
| 배포 이력 | 코드 커밋에 섞인다 | ✅ 깔끔하다 |
| 여러 앱을 한 클러스터에 | 저장소가 흩어진다 | ✅ 한 곳에 모인다 |

### ⚠️ 단일 저장소의 무한 루프

```
코드 커밋 → CI 실행 → 이미지 빌드 → 매니페스트 태그 갱신 커밋
                                              ↓
                                     또 CI 트리거 → 또 빌드 → ...  💥
```

```yaml
# 방지 1: paths 필터
on:
  push:
    paths: ["src/**", "Dockerfile"]      # k8s/ 변경은 무시

# 방지 2: 커밋 메시지에 스킵 지시자
# git commit -m "chore: bump image tag [skip ci]"
```

> **분리 저장소가 이 문제를 구조적으로 없앤다.** 다만 "코드와 매니페스트를 한 PR로 리뷰할 수 없다"는 대가가 있다.  
> 현실적인 절충: **플랫폼 매니페스트는 한 저장소에 모으고, 앱 매니페스트는 앱 저장소에 둔다.**

### 이 프로젝트의 구조

```
eks-gitops-platform/           # 단일 저장소지만 관심사로 디렉터리를 나눔
├── terraform/                 # 인프라 (AWS API)
├── gitops/                    # ArgoCD 가 감시하는 desired state
│   ├── bootstrap/
│   └── apps/
├── observability/             # Helm values (gitops/apps 가 $values 로 참조)
└── security/                  # Kyverno 정책, ESO 설정
```

> **values를 `gitops/` 밖에 두고 멀티소스 Application으로 참조**하는 게 포인트다. ArgoCD가 감시하는 경로와 사람이 편집하는 경로를 나누면 구조가 읽힌다. → `05-argocd-advanced/`

---

## 이미지 태그 갱신 — 세 가지 방법

CI가 이미지를 만든 뒤, 매니페스트의 태그를 누가 어떻게 바꾸는가.

### ① CI가 커밋한다 (가장 흔함)

```yaml
- name: 매니페스트 태그 갱신
  run: |
    git clone https://x-access-token:${{ secrets.MANIFEST_TOKEN }}@github.com/org/manifests.git
    cd manifests
    yq -i '.images[0].newTag = "'"${{ github.sha }}"'"' apps/myapp/overlays/dev/kustomization.yaml
    git config user.name "ci-bot"
    git config user.email "ci@example.com"
    git commit -am "chore(myapp): bump image to ${{ github.sha }}"
    git push
```

| 장점 | 단점 |
|---|---|
| 단순하고 명시적 | CI가 매니페스트 저장소 쓰기 권한 필요 |
| 커밋 이력이 남는다 | 동시 배포 시 push 충돌 |

> 충돌 대비로 `git pull --rebase` 후 재시도 루프를 넣는다. 여러 앱이 같은 저장소를 갱신하면 실제로 자주 부딪힌다.

### ② Argo CD Image Updater

```yaml
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=<ACCT>.dkr.ecr.../myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: digest
    argocd-image-updater.argoproj.io/write-back-method: git
```

레지스트리를 감시하다가 새 이미지가 올라오면 자동으로 갱신한다.

| 장점 | 단점 |
|---|---|
| CI가 매니페스트 저장소를 몰라도 된다 | 도구가 하나 늘어난다 |
| 레지스트리가 진실의 원천 | **어떤 커밋의 이미지인지 추적이 흐려질 수 있다** |

### ③ Kustomize / Helm 파라미터

```bash
# Kustomize
kustomize edit set image myapp=<ACCT>.dkr.ecr.../myapp:sha-a1b2c3d

# Helm values
yq -i '.image.tag = "sha-a1b2c3d"' values-dev.yaml
```

> **어느 방법이든 `latest`를 쓰지 않는 게 전제다.** 태그가 고정돼 있지 않으면 "지금 뭐가 돌고 있는지"를 Git으로 알 수 없고, 그 순간 GitOps가 아니다. → `04-container-image-pipeline/`

---

## 환경 승격 (Promotion)

같은 이미지를 dev → stg → prod로 올리는 과정이다. **재빌드하지 않는다** — 검증한 그 아티팩트가 그대로 올라가야 한다.

```
빌드 1회 ──▶ 이미지 sha-a1b2c3d
              │
              ├─▶ dev   (자동)      매니페스트 태그 갱신 커밋
              ├─▶ stg   (자동/승인)  같은 태그를 stg 오버레이에
              └─▶ prod  (승인)       같은 태그를 prod 오버레이에
```

> **환경마다 다시 빌드하면 "테스트한 것과 배포한 것이 다르다".** 빌드는 한 번, 승격은 태그 이동이다.

### 방법 1 — 디렉터리 오버레이 (Kustomize)

```
manifests/
├── base/
│   ├── deployment.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml     # images: newTag: sha-a1b2c3d
    ├── stg/
    │   └── kustomization.yaml     # images: newTag: sha-a1b2c3d
    └── prod/
        └── kustomization.yaml     # images: newTag: sha-9f8e7d6  ← 아직 이전 버전
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
images:
  - name: myapp
    newTag: sha-9f8e7d6
replicas:
  - name: myapp
    count: 3
patches:
  - path: resources-patch.yaml
```

> **승격 = prod 오버레이의 태그를 stg의 것으로 바꾸는 PR.** 리뷰·승인이 자연스럽게 붙고, 롤백은 `git revert`다.  
> 환경 차이(레플리카 수, 리소스, 도메인)는 오버레이 패치로 표현한다.

### 방법 2 — 브랜치/태그 분리

```yaml
# dev Application
targetRevision: main

# prod Application
targetRevision: v1.4.2      # 태그를 올리는 것이 승격
```

| | 오버레이 방식 | 브랜치/태그 방식 |
|---|---|---|
| 승격 | 태그 값을 바꾸는 PR | Application의 `targetRevision` 변경 |
| 환경 차이 | 오버레이 패치 | 브랜치 간 diff (drift 위험) |
| 가시성 | ✅ 한 브랜치에서 전부 보인다 | 브랜치를 오가야 한다 |
| 권장 | ✅ | 단순한 경우만 |

> **브랜치로 환경을 나누면 cherry-pick 지옥이 온다.** dev에만 있는 변경, prod에만 있는 핫픽스가 쌓이면 브랜치가 영영 안 맞는다.  
> **환경은 디렉터리로, 버전은 태그로** 표현하는 게 실무 기본이다.

---

## 시크릿은 Git에 넣지 않는다

GitOps의 유일한 진짜 난제다. Git이 진실의 원천인데 비밀번호는 Git에 못 넣는다.

| 방법 | 동작 |
|---|---|
| **External Secrets Operator** | 클러스터가 AWS SSM·Secrets Manager에서 당겨온다 |
| **Sealed Secrets** | 공개키로 암호화해 Git에 커밋, 클러스터가 복호화 |
| **SOPS + age/KMS** | 파일 단위 암호화, ArgoCD 플러그인으로 복호화 |
| ❌ 평문 Secret | 절대 안 됨 (base64는 암호화가 아니다) |

```yaml
# External Secrets — Git에는 '어디서 가져올지'만 남는다
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: grafana-admin
  namespace: observability
spec:
  secretStoreRef:
    name: aws-ssm
    kind: SecretStore
  target:
    name: grafana-admin           # 생성될 K8s Secret
  data:
    - secretKey: admin-password
      remoteRef:
        key: /eks-gitops/dev/grafana/admin-password
```

```
Git:        "SSM 의 /eks-gitops/dev/grafana/admin-password 를 가져와라"  ← 값이 아니라 참조
클러스터:    ESO 가 IRSA 로 SSM 을 읽어 Secret 을 만든다
```

> **IAM 역할은 Terraform, ServiceAccount 어노테이션은 GitOps** — 역할 ARN이 두 세계의 인계점이다. → `../terraform-lab/09-aws-vpc-eks/`  
> Sealed Secrets는 클러스터를 재생성하면 복호화 키가 사라져 시크릿을 못 푼다. 키 백업이 필수다. 그 점에서 **ESO 쪽이 클러스터를 매일 지우는 환경에 더 맞는다.**

---

## 무엇을 Git에 둘 것인가

| Git에 둔다 | Git에 두지 않는다 |
|---|---|
| 매니페스트·Helm values | 시크릿 값 |
| 이미지 태그·차트 버전 | 6000줄짜리 업스트림 차트 (멀티소스로 참조) |
| 정책·RBAC | state 파일 (`*.tfstate`) |
| ArgoCD Application | 개인 kubeconfig |

> 판단 기준: **"클러스터를 통째로 잃었을 때 이 저장소만으로 복구되는가?"** 그렇다면 충분하고, 그 이상은 노이즈다.

---

## 배운 점

- CI와 CD의 인계점은 **"새 이미지 태그를 매니페스트에 반영하는 커밋"**
- 단일 저장소는 **CI 무한 루프** 위험 — `paths` 필터나 `[skip ci]`로 막는다
- 분리 저장소는 루프가 구조적으로 없지만 원자적 리뷰를 잃는다
- 절충: **플랫폼 매니페스트는 모으고, 앱 매니페스트는 앱 저장소에**
- values를 `gitops/` 밖에 두고 **멀티소스로 참조**하면 감시 경로와 편집 경로가 분리된다
- 태그 갱신은 **CI 커밋 / Image Updater / kustomize edit** 세 가지
- CI 커밋 방식은 **동시 배포 시 push 충돌** — rebase 재시도가 필요하다
- 어느 방법이든 **`latest`를 쓰지 않는 게 전제**
- **승격은 재빌드가 아니라 태그 이동** — 테스트한 아티팩트가 그대로 올라가야 한다
- **환경은 디렉터리(오버레이), 버전은 태그** — 브랜치로 환경을 나누면 cherry-pick 지옥
- 승격 = prod 오버레이의 태그를 바꾸는 PR, 롤백 = `git revert`
- 시크릿은 Git에 두지 않는다 — **ESO / Sealed Secrets / SOPS**
- Git에는 **값이 아니라 참조**만 남긴다
- Sealed Secrets는 클러스터 재생성 시 키 분실 위험 — 잦은 재생성 환경엔 ESO
- 판단 기준: **"이 저장소만으로 클러스터를 복구할 수 있는가"**
