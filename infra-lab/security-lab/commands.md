# 보안 명령어 레퍼런스

## RBAC 점검
```
kubectl auth can-i --list                             # 내가 할 수 있는 것 전부
kubectl auth can-i create pods -n myapp
kubectl auth can-i delete secrets --all-namespaces
kubectl auth can-i '*' '*'                            # 클러스터 관리자인지

# 특정 주체 흉내내기 (impersonate)
kubectl auth can-i list secrets --as system:serviceaccount:myapp:default -n myapp
kubectl auth can-i --list --as system:serviceaccount:myapp:default -n myapp

kubectl auth whoami                                   # 현재 신원 (1.26+)

kubectl get clusterrolebindings -o wide
kubectl get rolebindings -A -o wide
kubectl describe clusterrole cluster-admin

# cluster-admin 을 가진 주체 찾기
kubectl get clusterrolebindings -o json \
  | jq -r '.items[] | select(.roleRef.name=="cluster-admin")
           | {name:.metadata.name, subjects:.subjects}'

# 위험 권한(secrets 조회) 보유자 찾기
kubectl get clusterroles -o json \
  | jq -r '.items[] | select(.rules[]? | select((.resources[]? == "secrets")
           and (.verbs[]? | test("get|list|\\*")))) | .metadata.name'
```

## ServiceAccount
```
kubectl get sa -A
kubectl -n myapp get sa default -o yaml
kubectl create sa myapp-sa -n myapp
kubectl -n myapp annotate sa myapp-sa \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/myapp-irsa

# 토큰 자동 마운트 끄기
kubectl -n myapp patch sa default -p '{"automountServiceAccountToken": false}'

# 파드 안에서 SA 토큰 확인
kubectl -n myapp exec -it <POD> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

## Pod Security Admission
```
kubectl label ns myapp pod-security.kubernetes.io/enforce=restricted
kubectl label ns myapp pod-security.kubernetes.io/enforce-version=v1.30
kubectl label ns myapp pod-security.kubernetes.io/warn=restricted
kubectl label ns myapp pod-security.kubernetes.io/audit=restricted

kubectl get ns --show-labels | grep pod-security

# 적용 전에 무엇이 걸릴지 확인 (warn 만 먼저)
kubectl label ns myapp pod-security.kubernetes.io/warn=restricted --overwrite
kubectl -n myapp rollout restart deploy    # 경고를 확인한다
```

## Kyverno
```
kubectl get clusterpolicy
kubectl get clusterpolicy -o wide
kubectl describe clusterpolicy disallow-latest-tag

# 정책 위반 리포트
kubectl get policyreport -A
kubectl get clusterpolicyreport
kubectl -n myapp get policyreport -o wide
kubectl -n myapp get policyreport -o json \
  | jq -r '.items[].results[] | select(.result=="fail") | "\(.policy)/\(.rule): \(.message)"'

# 정책 테스트 (클러스터 없이)
kyverno apply security/kyverno/policies/ --resource bad-pod.yaml
kyverno apply security/kyverno/policies/ --resource manifests/ --policy-report
kyverno test ./tests                                  # 정책 단위 테스트

kubectl -n kyverno logs deploy/kyverno-admission-controller -f
kubectl get validatingwebhookconfigurations | grep kyverno
```

## Trivy
```
trivy image myapp:1.0                                 # 이미지 취약점
trivy image --severity CRITICAL --ignore-unfixed myapp:1.0
trivy image --format cyclonedx -o sbom.json myapp:1.0
trivy image --scanners vuln,secret,misconfig myapp:1.0

trivy config .                                        # IaC 미스컨피그
trivy fs --scanners vuln,secret .                     # 소스 코드·시크릿
trivy k8s --report summary cluster                    # 클러스터 전체 스캔
trivy k8s --namespace myapp --report all
```

## cosign — 서명·검증
```
cosign generate-key-pair
cosign sign --key cosign.key myapp:1.0
cosign verify --key cosign.pub myapp:1.0

# keyless (OIDC 신원 기반, 키 관리 불필요)
cosign sign --yes myapp@sha256:...
cosign verify myapp@sha256:... \
  --certificate-identity-regexp "https://github.com/GWANG-MIN1/.*" \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com

cosign attach sbom --sbom sbom.json myapp:1.0
cosign tree myapp:1.0                                 # 첨부된 서명·SBOM 확인
```

## External Secrets
```
kubectl get clustersecretstore                        # STATUS 가 Valid 여야 한다
kubectl get secretstore -A
kubectl describe clustersecretstore aws-ssm

kubectl get externalsecret -A
kubectl -n observability describe externalsecret grafana-admin
# STATUS: SecretSynced 여야 정상

kubectl -n observability get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d
kubectl -n external-secrets logs deploy/external-secrets -f

# SSM 쪽
aws ssm put-parameter --type SecureString \
  --name /eks-gitops/dev/grafana/admin-password --value "$(openssl rand -base64 24)"
aws ssm get-parameter --name /eks-gitops/dev/grafana/admin-password --with-decryption
aws ssm describe-parameters --parameter-filters "Key=Name,Option=BeginsWith,Values=/eks-gitops/"
```

## kube-bench (CIS 벤치마크)
```
kubectl apply -f security/kube-bench/job-eks.yaml
kubectl -n kube-bench wait --for=condition=complete job/kube-bench --timeout=120s
kubectl -n kube-bench logs job/kube-bench
kubectl delete -f security/kube-bench/job-eks.yaml    # 재실행하려면 삭제 먼저

# 로컬에서
kube-bench run --targets node,policies --benchmark eks-1.5.0
kube-bench run --json | jq '.Controls[].tests[].results[] | select(.status=="FAIL")'
```

## 감사 로그 (EKS)
```
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["audit","authenticator"],"enabled":true}]}'

# CloudWatch Logs Insights 쿼리
fields @timestamp, user.username, verb, objectRef.resource, responseStatus.code
| filter objectRef.resource = "secrets" and verb in ["get","list"]
| sort @timestamp desc

# exec 사용 추적
fields @timestamp, user.username, objectRef.namespace, objectRef.name
| filter objectRef.subresource = "exec"
```

## 클러스터 상태 점검
```
# privileged 컨테이너 찾기
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.containers[].securityContext.privileged==true)
  | "\(.metadata.namespace)/\(.metadata.name)"'

# hostNetwork / hostPID 사용 파드
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.hostNetwork==true or .spec.hostPID==true)
  | "\(.metadata.namespace)/\(.metadata.name)"'

# root 로 도는 파드
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.securityContext.runAsNonRoot != true)
  | "\(.metadata.namespace)/\(.metadata.name)"'

# :latest 사용 이미지
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}' \
  | grep ':latest'

kubectl get networkpolicy -A                          # 네트워크 정책 유무
```
