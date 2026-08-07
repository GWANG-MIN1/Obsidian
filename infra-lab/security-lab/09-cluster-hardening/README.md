# 09 클러스터 하드닝 & kube-bench

**CIS Kubernetes Benchmark**는 클러스터 설정의 업계 표준 점검 항목이고, **kube-bench**는 그걸 자동으로 검사해주는 도구다.  
다만 관리형 클러스터(EKS)에서는 **애초에 조치할 수 없는 항목**이 많다. 그걸 구분하는 게 이 장의 절반이다.

---

## kube-bench 실행

```yaml
# 온디맨드 Job — GitOps 로 관리하지 않는다
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: kube-bench
spec:
  template:
    spec:
      hostPID: true                     # 노드 프로세스를 봐야 한다
      restartPolicy: Never
      containers:
        - name: kube-bench
          image: docker.io/aquasec/kube-bench:v0.15.6
          command:
            ["kube-bench", "run",
             "--targets", "node,policies,managedservices,controlplane",
             "--benchmark", "eks-1.5.0"]
          volumeMounts:
            - { name: var-lib-kubelet, mountPath: /var/lib/kubelet, readOnly: true }
            - { name: etc-systemd,     mountPath: /etc/systemd,     readOnly: true }
            - { name: etc-kubernetes,  mountPath: /etc/kubernetes,  readOnly: true }
      volumes:
        - { name: var-lib-kubelet, hostPath: { path: "/var/lib/kubelet" } }
        - { name: etc-systemd,     hostPath: { path: "/etc/systemd" } }
        - { name: etc-kubernetes,  hostPath: { path: "/etc/kubernetes" } }
```

```bash
kubectl apply -f security/kube-bench/job-eks.yaml
kubectl -n kube-bench wait --for=condition=complete job/kube-bench --timeout=120s
kubectl -n kube-bench logs job/kube-bench
kubectl delete -f security/kube-bench/job-eks.yaml   # 재실행하려면 삭제 먼저
```

> **GitOps로 관리하지 않는 이유:** 이건 서비스가 아니라 **한 번 실행되고 완료되는 Job**이다. ArgoCD 아래 두면 끝난 Job을 계속 돌보게 되고, `Completed` 상태를 어떻게 볼지 계속 시비가 붙는다.  
> **"GitOps로 관리할 것"과 "온디맨드로 돌릴 것"의 경계**를 보여주는 예다. → `../cicd-lab/06-app-of-apps-applicationset/`

### 🔧 검사 도구가 정책을 위반한다

```
kube-bench 는 노드를 검사해야 하므로
  hostPID: true + hostPath 마운트가 기능상 필수
      ↓
그런데 이건 restricted 정책이 금지하는 바로 그것들
      ↓
그래서 kube-bench 네임스페이스를 Kyverno 정책에서 제외한다
```

> 보안 도구가 보안 정책을 위반하는 역설이다. **정당한 예외이므로 네임스페이스를 분리하고 이유를 매니페스트 주석에 남긴다.** → `04-kyverno/` `03-pod-security-standards/`

---

## 리포트 읽는 법

```
[INFO] 3 Worker Node Security Configuration
[PASS] 3.1.1 kubelet 설정 파일 권한이 644 이하
[FAIL] 3.2.x ...
[WARN] 3.1.2 kubelet 설정 파일 소유자가 root:root
[INFO] 3.2.x 수동 확인 필요

== Summary ==
N checks PASS
N checks FAIL
N checks WARN
N checks INFO
```

| 상태 | 의미 | 대응 |
|---|---|---|
| **PASS** | 통과 | — |
| **FAIL** | 위반 | 원인 확인 후 조치 판단 |
| **WARN** | 자동 판정 불가 | 수동 확인 |
| **INFO** | 참고 | 읽어본다 |

### ⚠️ EKS에서 조치할 수 없는 것들

```
--targets 중
  controlplane / managedservices  → AWS 가 관리하는 영역
                                     kube-apiserver 플래그, etcd 암호화 설정 등
                                     우리가 바꿀 수 없다
  node / policies                 → 우리가 실제로 조치 가능한 부분
```

| 항목 | EKS에서 |
|---|---|
| API 서버 플래그 (`--anonymous-auth` 등) | ❌ AWS 관리 |
| etcd 설정·암호화 | ❌ AWS 관리 (KMS 봉투 암호화는 클러스터 생성 시 선택) |
| 스케줄러·컨트롤러 매니저 | ❌ AWS 관리 |
| **kubelet 설정·파일 권한** | ✅ 노드 AMI·부트스트랩으로 조치 |
| **RBAC·NetworkPolicy·PSA** | ✅ 우리 영역 |
| **감사 로그 활성화** | ✅ EKS 설정 |

> **FAIL 개수를 0으로 만들려 애쓰지 않는다.** 컨트롤 플레인 항목은 조치가 불가능하고, 그걸 붙들면 시간만 쓴다.  
> **리포트는 node와 policies 항목을 중심으로 읽는다.** 그게 우리가 바꿀 수 있는 부분이다.  
> 이건 관측성에서 EKS 컨트롤 플레인 스크랩 타겟을 끄는 것과 같은 판단이다 — **조치 불가능한 빨간불은 신호가 아니라 소음**이다. → `../observability-lab/02-prometheus-architecture/`

---

## 실제로 조치 가능한 하드닝

### ① IMDSv2 강제 + hop limit

가장 효과가 큰 단일 조치다.

```hcl
# Terraform — 노드 그룹
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required"   # IMDSv2 (토큰 필수) → SSRF 방어
  http_put_response_hop_limit = 1            # 컨테이너에서 IMDS 접근 차단
}
```

```bash
# 파드에서 확인 — 접근되면 안 된다
kubectl run test --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s --max-time 3 http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

> **IMDS 접근이 되면 파드가 노드 IAM 역할의 자격증명을 그대로 얻는다.** IRSA로 파드 권한을 최소화한 의미가 사라진다.  
> `hop_limit = 1`이 컨테이너 네임스페이스에서 나가는 요청을 TTL로 차단한다. → `08-network-security/`

### ② 노드 IAM 역할 최소화

```
노드 역할은 '노드가 되기 위한 최소 권한'만 가져야 한다
  AmazonEKSWorkerNodePolicy, AmazonEC2ContainerRegistryReadOnly, AmazonEKS_CNI_Policy
      ↓
애플리케이션 권한은 IRSA 로 파드마다
```

> **노드 역할에 S3 접근을 붙이면 그 노드의 모든 파드가 S3를 쓸 수 있다.** IRSA를 쓰는 이유가 이것이다. → `05-rbac-least-privilege/`

### ③ 감사 로그

```bash
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["audit","authenticator"],"enabled":true}]}'
```

```
audit          누가 무엇을 했는가 (API 서버 요청 전부)
authenticator  IAM ↔ K8s 신원 매핑
api            API 서버 로그
controllerManager / scheduler
```

> **감사 로그가 없으면 사고 후 재구성이 불가능하다.** 비용이 들지만 `audit`과 `authenticator`는 켜둘 가치가 있다. → `10-runtime-detection/`

### ④ API 엔드포인트 제한

```hcl
cluster_endpoint_public_access       = true
cluster_endpoint_public_access_cidrs = ["203.0.113.0/24"]   # 사무실·VPN 만
cluster_endpoint_private_access      = true
```

> 데모라면 `0.0.0.0/0`이어도 되지만, **그 밖에서는 반드시 좁힌다.** 인증이 있어도 공격 표면을 열어둘 이유가 없다.

### ⑤ 시크릿 암호화 (KMS 봉투 암호화)

```hcl
cluster_encryption_config = {
  provider_key_arn = aws_kms_key.eks.arn
  resources        = ["secrets"]
}
```

> **클러스터 생성 시에만 설정 가능하다.** 나중에 후회하지 않으려면 처음에 켠다.  
> 이게 없으면 etcd에 Secret이 평문으로 저장된다. → `06-secrets-management/`

### ⑥ 노드 하드닝

| 항목 | 조치 |
|---|---|
| 최신 AMI | Bottlerocket 또는 최신 AL2023, 정기 교체 |
| SSH 접근 | **비활성화** — SSM Session Manager 사용 |
| 노드 수명 | 짧게 (교체가 곧 패치) |
| 불필요한 패키지 | Bottlerocket은 애초에 최소 구성 |

```
Bottlerocket = 컨테이너 전용 최소 OS
  패키지 매니저·셸 없음, 읽기 전용 루트, 자동 업데이트
  → 노드 공격 표면이 크게 줄어든다
```

> **"노드를 패치한다"보다 "노드를 교체한다"** 가 클라우드 네이티브의 방식이다. 수명이 짧으면 침해가 지속되지 못한다.

---

## 다른 점검 도구

| 도구 | 대상 |
|---|---|
| **kube-bench** | CIS 벤치마크 (노드·클러스터 설정) |
| **kubescape** | NSA/CISA·MITRE ATT&CK 프레임워크 기준 |
| **Trivy k8s** | 실행 중 워크로드의 취약점·미스컨피그 |
| **Polaris** | 워크로드 모범 사례 (probe·리소스·보안 컨텍스트) |
| **kubeaudit** | 매니페스트 감사 |

```bash
kubescape scan framework nsa
trivy k8s --report summary cluster
polaris audit --audit-path ./manifests
```

> **도구를 늘리는 것보다 하나의 결과를 실제로 조치하는 게 낫다.** 리포트만 쌓이면 아무것도 개선되지 않는다.

---

## 정기 점검 루틴

```
분기 1회
  □ kube-bench 실행 → node/policies FAIL 항목 검토
  □ cluster-admin 보유 주체 목록 확인
  □ 사용되지 않는 ServiceAccount·RoleBinding 정리
  □ NetworkPolicy 없는 네임스페이스 확인
  □ privileged / hostPath 파드 목록 확인
  □ Kyverno Audit 리포트 → Enforce 승격 후보 선별

상시
  □ Trivy CI 스캔 (주간 재스캔 포함)
  □ 감사 로그 이상 징후
  □ Kubernetes 버전 (EOL 전 업그레이드)
```

```bash
# 한 번에 훑기
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.hostNetwork==true or .spec.hostPID==true
           or (.spec.containers[].securityContext.privileged==true))
  | "\(.metadata.namespace)/\(.metadata.name)"'

kubectl version --short          # EOL 확인
```

> **Kubernetes 버전 관리가 의외로 중요한 보안 항목이다.** EOL 버전은 보안 패치를 못 받는다. EKS는 버전별 지원 종료일이 정해져 있다.  
> 또한 클러스터 버전이 낮으면 차트가 요구하는 필드를 지원하지 못해 다른 문제로 번지기도 한다. → `../cicd-lab/05-argocd-advanced/`

---

## 배운 점

- kube-bench는 **온디맨드 Job** — 완료되는 Job은 GitOps로 관리하지 않는다
- 🔧 **검사 도구가 정책을 위반한다** — `hostPID`·hostPath가 기능상 필수라 네임스페이스를 예외로 둔다
- 리포트 상태는 PASS / FAIL / **WARN(수동 확인)** / INFO
- ⚠️ **EKS는 컨트롤 플레인 항목을 조치할 수 없다** — FAIL 0을 목표로 삼지 않는다
- **node·policies 항목을 중심으로** 읽는다 (우리가 바꿀 수 있는 부분)
- 조치 불가능한 빨간불은 **신호가 아니라 소음** (관측성의 EKS 스크랩 타겟과 같은 판단)
- ⭐ **IMDSv2 + hop limit 1**이 가장 효과가 큰 단일 조치
- 노드 IAM 역할은 최소화하고 **애플리케이션 권한은 IRSA로 파드마다**
- **감사 로그가 없으면 사고 후 재구성이 불가능하다** (`audit`, `authenticator`)
- API 엔드포인트 CIDR를 좁힌다 — 인증이 있어도 표면을 열어둘 이유가 없다
- **KMS 시크릿 암호화는 클러스터 생성 시에만** 설정 가능하다
- SSH 대신 **SSM Session Manager**, 노드는 Bottlerocket 같은 최소 OS
- **"노드를 패치한다"보다 "노드를 교체한다"** — 수명이 짧으면 침해가 지속되지 못한다
- 도구를 늘리는 것보다 **하나의 결과를 실제로 조치**하는 게 낫다
- **Kubernetes 버전 관리도 보안 항목** — EOL은 패치를 못 받는다
