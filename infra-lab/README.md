# infra-lab

Linux → Docker → Kubernetes로 이어지는 DevOps 학습 실습 저장소.  
각 장은 **개념 정리 + 실행 가능한 명령 + 배운 점**으로 구성하고, 트랙마다 `commands.md` 레퍼런스를 따로 둔다.

---

## 학습 경로

컨테이너는 결국 **리눅스 커널 기능(namespace·cgroups·capabilities) 위에서 도는 프로세스**다.  
그래서 리눅스를 먼저 다지고, 컨테이너 → 오케스트레이션 순으로 쌓아 올린다.

```
Linux (커널·프로세스·권한·네트워크)
  └─ Docker (컨테이너·이미지·네트워크·볼륨·보안)
       └─ 오케스트레이션 개념 (Swarm)
            └─ Kubernetes (Pod·Controller·Service·Ingress·Storage·RBAC·스케줄링)
                 └─ Helm + GitOps (ArgoCD)
                      └─ 관리형 K8s (EKS) + IaC (Terraform)
                           └─ 관측성 (Prometheus/Grafana) + DevSecOps
```

---

## 진행 현황

| 트랙 | 범위 | 상태 |
|---|---|---|
| [linux-scripts](linux-scripts/) | 01~10 | ✅ 완료 |
| [docker-labs](docker-labs/) | 01~10 | ✅ 완료 |
| [k8s-manifests](k8s-manifests/) | 01~10 | ✅ 완료 |
| [terraform-lab](terraform-lab/) | 01~10 | ✅ 완료 |
| [troubleshooting](troubleshooting/) | 상시 기록 | 🚧 진행 예정 |

---

## 트랙별 목차

### [linux-scripts](linux-scripts/) — 리눅스 기초부터 서버 운영까지

| # | 주제 | 키워드 |
|---|---|---|
| [01](linux-scripts/01-basics/) | 리눅스 기초 | FHS, 셸, 탐색·조작 |
| [02](linux-scripts/02-file-permissions/) | 파일 권한 & 소유권 | rwx, chmod/chown, umask, 특수권한 |
| [03](linux-scripts/03-process-management/) | 프로세스 관리 | ps/top, 시그널, 잡 제어, nice |
| [04](linux-scripts/04-text-processing/) | 텍스트 처리 | 파이프·리다이렉션, grep/sed/awk |
| [05](linux-scripts/05-users-groups/) | 사용자 & 그룹 | useradd/usermod, sudo, passwd·shadow |
| [06](linux-scripts/06-package-management/) | 패키지 관리 | apt, dnf/yum, 저장소·의존성 |
| [07](linux-scripts/07-systemd-service/) | systemd & 서비스 관리 | 유닛, systemctl, journalctl, 타이머 |
| [08](linux-scripts/08-networking/) | 네트워킹 | ip/ss, 방화벽, DNS, SSH |
| [09](linux-scripts/09-shell-scripting/) | 셸 스크립팅 (Bash) | 변수·조건·반복·함수, 안전 옵션, cron |
| [10](linux-scripts/10-monitoring-performance/) | 모니터링 & 성능 · 트러블슈팅 | 4대 자원, USE 방법론, df/du/vmstat |

📎 [commands.md](linux-scripts/commands.md) — Linux 명령어 레퍼런스

### [docker-labs](docker-labs/) — 컨테이너 만들기부터 보안·오케스트레이션까지

| # | 주제 | 키워드 |
|---|---|---|
| [01](docker-labs/01-container-basics/) | 컨테이너 기초 | namespace/cgroups, 이미지 vs 컨테이너, 라이프사이클, exec |
| [02](docker-labs/02-networking/) | 네트워킹 & 포트 노출 | `-p` vs EXPOSE, iptables DNAT, 환경변수 |
| [03](docker-labs/03-volumes/) | 볼륨 | volume/bind/tmpfs, `--volumes-from`, 권한, 백업 |
| [04](docker-labs/04-network/) | 네트워크 | bridge·host·none·container·macvlan, `--net-alias` |
| [05](docker-labs/05-logging/) | 컨테이너 로깅 | json-file, syslog, fluentd, awslogs |
| [06](docker-labs/06-resource-limit/) | 자원 할당 제한 | `--memory`, `--cpus`, cpu-shares/cpuset |
| [07](docker-labs/07-image/) | 도커 이미지 | Dockerfile, build, 레이어, save/load, 레지스트리 |
| [08](docker-labs/08-compose/) | Docker Compose | 멀티 컨테이너, depends_on, healthcheck, 스케일링 |
| [09](docker-labs/09-security/) | 컨테이너 보안 | capabilities, seccomp, 비-root, 읽기전용 FS, 이미지 스캔 |
| [10](docker-labs/10-orchestration/) | 컨테이너 오케스트레이션 | 선언형·셀프힐링, Swarm, K8s로 가는 다리 |

📎 [commands.md](docker-labs/commands.md) — Docker 명령어 레퍼런스

### [k8s-manifests](k8s-manifests/) — 로컬 클러스터에서 GitOps까지

| # | 주제 | 키워드 |
|---|---|---|
| [01](k8s-manifests/01-cluster-basics/) | 클러스터 기초 | Control Plane/Node, minikube·kind, kubectl |
| [02](k8s-manifests/02-pod/) | Pod | 매니페스트, 라이프사이클, 멀티 컨테이너, probe |
| [03](k8s-manifests/03-controllers/) | 워크로드 컨트롤러 | ReplicaSet, Deployment, DaemonSet, StatefulSet, Job |
| [04](k8s-manifests/04-service/) | Service & 네트워킹 | ClusterIP/NodePort/LoadBalancer, DNS, Endpoints |
| [05](k8s-manifests/05-config/) | ConfigMap & Secret | env·volume 주입, 설정과 이미지 분리 |
| [06](k8s-manifests/06-storage/) | 스토리지 | PV/PVC, StorageClass, 동적 프로비저닝 |
| [07](k8s-manifests/07-ingress/) | Ingress | Controller, 경로·호스트 라우팅, TLS 종료 |
| [08](k8s-manifests/08-namespace-rbac/) | Namespace & RBAC | Role/RoleBinding, ServiceAccount, Quota |
| [09](k8s-manifests/09-scheduling/) | 스케줄링 & 오토스케일링 | affinity, taint·toleration, requests/limits, HPA |
| [10](k8s-manifests/10-helm-gitops/) | Helm & GitOps | 차트·values, ArgoCD, EKS로 가는 다리 |

📎 [commands.md](k8s-manifests/commands.md) — kubectl 명령어 레퍼런스

### [terraform-lab](terraform-lab/) — 인프라를 코드로

| # | 주제 | 키워드 |
|---|---|---|
| [01](terraform-lab/01-iac-basics/) | IaC 기초 | 선언형 vs 명령형, 도구 비교, init/plan/apply |
| [02](terraform-lab/02-provider-resource/) | Provider & Resource | HCL 문법, 버전 제약, 자격증명, 참조·의존성 |
| [03](terraform-lab/03-state/) | State | tfstate, S3 백엔드·잠금, state 명령, 드리프트 |
| [04](terraform-lab/04-variables-outputs/) | 변수 & 출력 | variable·locals·output, validation, 주입 우선순위 |
| [05](terraform-lab/05-module/) | 모듈 | 루트/자식 모듈, source, 레지스트리, 설계 원칙 |
| [06](terraform-lab/06-meta-arguments/) | 메타 인수 & 표현식 | count vs for_each, lifecycle, dynamic, for·splat |
| [07](terraform-lab/07-data-import/) | Data Source & 임포트 | data, import·moved·removed 블록 |
| [08](terraform-lab/08-workspace-environment/) | 환경 분리 전략 | workspace vs 디렉터리 분리, backend-config, Terragrunt |
| [09](terraform-lab/09-aws-vpc-eks/) | 실습: VPC + EKS | 부트스트랩, 모듈 조립, IRSA, kubeconfig |
| [10](terraform-lab/10-cicd-policy/) | CI/CD & 정책 검사 | OIDC, plan-on-PR, tfsec·checkov, OPA, Atlantis |

📎 [commands.md](terraform-lab/commands.md) — Terraform 명령어 레퍼런스

### [troubleshooting](troubleshooting/) — 실제로 막혔던 것들

학습 노트와 별개로, **직접 겪은 장애·삽질을 기록**한다. 다음 형식을 따른다.

```
## 현상       무엇이 어떻게 안 됐는가 (에러 메시지 원문 포함)
## 환경       OS/버전/구성 — 재현에 필요한 최소 정보
## 원인       왜 그랬는가 (추측이 아니라 확인한 근거)
## 해결       실제로 통한 명령·설정
## 재발 방지   다음에 안 겪으려면 무엇을 바꿔야 하는가
```

---

## 문서 규칙

- 파일명은 `NN-주제/README.md` — 번호로 학습 순서를 고정한다
- 문서는 `# NN 제목` → 도입부 → 섹션(`---`로 구분) → **`## 배운 점`** 순서
- 명령은 실제로 실행해본 것만 적는다. 옵션은 주석으로 의미를 남긴다
- 함정·주의사항은 `>` 인용으로 눈에 띄게 분리한다
- 다른 장과 이어지는 내용은 화살표와 폴더명으로 참조를 남긴다 (예: → `09-security/`)

---

## 다음 단계

| 트랙 | 다룰 내용 |
|---|---|
| `observability-lab/` | Prometheus·PromQL, Grafana, Alertmanager, Loki, OpenTelemetry |

> 실제 적용 결과는 별도 저장소 [eks-gitops-platform](https://github.com/GWANG-MIN1/eks-gitops-platform)에서 EKS 위에 GitOps·관측성·DevSecOps로 이어진다.
