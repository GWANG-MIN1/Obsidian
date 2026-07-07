
# Kubernetes Labs

Kubernetes 학습 실습 및 명령어 정리 저장소

## 구조
- `commands.md` - 자주 쓰는 kubectl 명령어 레퍼런스
- `01-cluster-basics/` - 클러스터 아키텍처 (Control Plane / Node), 로컬 클러스터 (minikube, kind), kubectl 기본
- `02-pod/` - Pod 개념, 매니페스트 구조, 라이프사이클, 멀티 컨테이너, liveness/readiness probe
- `03-controllers/` - ReplicaSet, Deployment (롤링 업데이트·롤백), DaemonSet, StatefulSet, Job, CronJob
- `04-service/` - Service 타입 (ClusterIP, NodePort, LoadBalancer), DNS 서비스 디스커버리, Endpoints, headless
- `05-config/` - ConfigMap, Secret (env·volume 주입), 설정과 이미지 분리
- `06-storage/` - Volume, PersistentVolume, PersistentVolumeClaim, StorageClass, 동적 프로비저닝, 접근 모드
- `07-ingress/` - Ingress, Ingress Controller, 경로·호스트 기반 라우팅, TLS 종료
- `08-namespace-rbac/` - Namespace, RBAC (Role, RoleBinding, ClusterRole), ServiceAccount, ResourceQuota, LimitRange
- `09-scheduling/` - 레이블·셀렉터, nodeSelector, affinity, taint·toleration, requests·limits, HPA 오토스케일링
- `10-helm-gitops/` - Helm 차트 (패키징·템플릿·values), GitOps (ArgoCD 선언형 배포), 관리형 K8s (EKS)로 가는 다리
