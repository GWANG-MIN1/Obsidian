# 10 Helm & GitOps

매니페스트가 수십 개로 늘고 환경(dev/stg/prod)이 여러 개가 되면 YAML 관리가 지옥이 된다.  
**Helm**은 매니페스트를 패키징·템플릿화하고, **GitOps(ArgoCD)** 는 Git을 유일한 진실의 원천으로 삼아 클러스터를 자동 동기화한다. Kubernetes 학습의 종착점이자 실무 운영의 시작점.

---

## Helm — Kubernetes 패키지 매니저

apt·npm처럼 애플리케이션을 **차트(Chart)** 로 패키징한다. 템플릿 + 값(values)으로 환경별 배포를 하나의 정의로 관리.

### 왜 Helm인가

```
문제: deployment.yaml, service.yaml, ingress.yaml, configmap.yaml...
     환경마다 복붙 → replicas·이미지 태그·도메인만 다른 파일 수십 개
해결: 템플릿 1벌 + values-dev.yaml / values-prod.yaml
```

- **템플릿화**: 변하는 값(이미지·복제수·도메인)을 변수로
- **패키징**: 관련 매니페스트를 하나의 차트로 배포·버전 관리
- **릴리스 관리**: 설치·업그레이드·롤백을 한 명령으로
- **재사용**: bitnami 등 공개 차트로 DB·모니터링을 즉시 설치

### 차트 구조

```
mychart/
├── Chart.yaml          # 차트 메타데이터 (이름·버전)
├── values.yaml         # 기본 값 (변수)
├── templates/          # 템플릿 매니페스트
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── charts/             # 의존 차트
```

### 템플릿 & values

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: "1.0"
```

```yaml
# templates/deployment.yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### 주요 명령

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install myapp bitnami/nginx                    # 공개 차트 설치
helm install myapp ./mychart -f values-prod.yaml    # 로컬 차트 + 환경 값
helm upgrade myapp ./mychart --set replicaCount=5   # 업그레이드
helm rollback myapp 1                               # 리비전 1로 롤백
helm list                                           # 릴리스 목록
helm uninstall myapp                                # 제거
helm template ./mychart                             # 렌더링 결과 확인 (설치 안 함)
```

> **Kustomize**는 Helm의 대안으로, 템플릿 없이 base + overlay(패치)로 환경 차이를 관리한다. `kubectl apply -k`로 내장 지원. Helm(패키징·배포)과 Kustomize(오버레이)는 상호 보완적으로 함께 쓰기도 한다.

---

## GitOps — Git이 유일한 진실의 원천

**클러스터의 원하는 상태를 Git에 선언하고, 에이전트가 Git과 클러스터를 자동으로 일치시킨다.** `kubectl apply`를 사람이 직접 하지 않는다.

```
개발자 ──git push──▶ Git 리포 (원하는 상태)
                         │
                    ArgoCD가 감지
                         │  동기화(sync)
                         ▼
                    Kubernetes 클러스터 (실제 상태)
                    현재 ≠ 원하는 → 자동 조정
```

### Push 방식 vs Pull(GitOps) 방식

| 구분 | 전통 CI/CD (Push) | GitOps (Pull) |
|------|-------------------|---------------|
| 배포 주체 | CI가 클러스터에 `kubectl apply` | 클러스터 내 에이전트가 Git을 당겨 적용 |
| 자격증명 | CI가 클러스터 접근 권한 보유 (외부 노출) | 에이전트가 안에서 처리 (더 안전) |
| 진실의 원천 | 불명확 (CI 로그·수동 변경 혼재) | **Git = 단일 원천** |
| 드리프트 | 수동 변경 감지 어려움 | 자동 감지·복구 |

### GitOps 4원칙

1. **선언형** — 시스템 상태를 선언적으로 기술 (YAML)
2. **버전 관리·불변** — Git에 저장, 모든 변경이 커밋 이력
3. **자동 적용(pull)** — 승인된 변경을 에이전트가 자동 반영
4. **지속적 조정** — 실제 상태를 원하는 상태로 계속 맞춤 (드리프트 자동 복구)

---

## ArgoCD

가장 대중적인 GitOps 컨트롤러. Git 리포를 감시하며 클러스터를 동기화하고, 상태를 시각화한다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/GWANG-MIN1/gitops-repo
    targetRevision: main
    path: apps/myapp            # 매니페스트(또는 Helm 차트) 경로
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true               # Git에서 지운 리소스 자동 삭제
      selfHeal: true            # 수동 변경 시 Git 상태로 자동 복구
```

| 개념 | 의미 |
|------|------|
| **Application** | "이 Git 경로를 이 클러스터/ns에 동기화" 선언 |
| **Sync** | Git 상태를 클러스터에 반영 |
| **prune** | Git에서 삭제된 리소스를 클러스터에서도 제거 |
| **selfHeal** | 클러스터의 수동 변경을 Git 기준으로 되돌림 (드리프트 복구) |
| **App of Apps** | 하나의 Application이 여러 Application을 관리 (계층 배포) |

> ArgoCD 대시보드에서 리소스 트리·동기화 상태·헬스를 한눈에 본다. `OutOfSync`(Git≠클러스터), `Synced`, `Healthy` 상태로 배포를 관찰·관리한다.

---

## Kubernetes → 관리형 클러스터(EKS)로

지금까지는 로컬(minikube)에서 배웠다. 실무는 **관리형 Kubernetes**에서 운영한다 — Control Plane을 클라우드가 관리해주므로 워크로드에 집중할 수 있다.

| 관리형 서비스 | 클라우드 |
|--------------|----------|
| **EKS** | AWS |
| **GKE** | Google Cloud |
| **AKS** | Azure |

### EKS로 확장 시 매핑

| 로컬에서 배운 것 | EKS에서 |
|------------------|---------|
| minikube 클러스터 | EKS 클러스터 (Control Plane 관리형) |
| Service LoadBalancer | AWS ELB/NLB 자동 프로비저닝 |
| Ingress | AWS ALB (aws-load-balancer-controller) |
| StorageClass/PVC | EBS CSI / EFS CSI 드라이버 |
| 노드 스케일링 | Cluster Autoscaler / **Karpenter** |
| RBAC + ServiceAccount | **IRSA** (IAM Role for ServiceAccount) |
| ConfigMap/Secret | + AWS Secrets Manager (External Secrets) |

---

## 전체 스택 연결 (DevOps 여정)

```
컨테이너(Docker)
   └─ 오케스트레이션 개념(Swarm)
        └─ Kubernetes (Pod·Deployment·Service·Ingress·Storage·RBAC·스케줄링)
             └─ 패키징(Helm) + GitOps(ArgoCD)
                  └─ 관리형 K8s(EKS) + IaC(Terraform)
                       └─ 관측성(Prometheus/Grafana) + DevSecOps
```

> 이 저장소(docker-labs → k8s-manifests)는 그 여정의 앞부분이다. 다음 단계는 EKS 위에 ArgoCD로 GitOps 파이프라인을 세우고, Terraform으로 인프라를 코드화하며, 관측성·보안을 얹는 것 — 실제 프로덕션 플랫폼의 모습이다.

---

## 배운 점

- **Helm** = 매니페스트 패키징·템플릿화 → 환경별 values로 재사용·릴리스 관리·롤백
- Kustomize(오버레이)는 Helm의 대안·보완
- **GitOps** = Git이 유일한 진실의 원천, 에이전트가 pull 방식으로 자동 동기화 (Push CI/CD보다 안전·추적 가능)
- **ArgoCD**: Application으로 선언, `prune`·`selfHeal`로 드리프트 자동 복구
- 실무는 **관리형 K8s(EKS)** — Ingress→ALB, PVC→EBS, RBAC→IRSA, 스케일→Karpenter로 매핑
- 여기까지가 컨테이너 → 오케스트레이션 → K8s → GitOps → 클라우드 플랫폼으로 이어지는 DevOps 스택의 뼈대
