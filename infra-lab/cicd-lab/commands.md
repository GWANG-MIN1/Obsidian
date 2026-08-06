# CI/CD 명령어 레퍼런스

## gh — GitHub CLI
```
gh auth login                                  # 로그인
gh repo clone OWNER/REPO
gh pr create --fill                            # 커밋 메시지로 PR 생성
gh pr list --state open
gh pr checks                                   # 현재 PR의 CI 상태
gh pr merge --squash --delete-branch

gh run list --workflow=ci.yml --limit 10       # 워크플로 실행 목록
gh run view <RUN_ID>                           # 실행 상세
gh run view <RUN_ID> --log-failed              # 실패한 스텝 로그만
gh run watch                                   # 진행 중인 실행 실시간 추적
gh run rerun <RUN_ID> --failed                 # 실패한 잡만 재실행
gh workflow run deploy.yml -f env=prod         # workflow_dispatch 수동 실행
gh workflow list / disable / enable
```

## GitHub Actions — 로컬 검증
```
act -l                                         # 워크플로 목록 (nektos/act)
act pull_request                               # 로컬에서 실행
actionlint                                     # 워크플로 문법 린트
yamllint .github/workflows/
gh api repos/:owner/:repo/actions/secrets      # 시크릿 목록 (값은 안 나옴)
```

## docker buildx — 이미지 빌드
```
docker buildx create --use --name builder
docker buildx build -t myapp:1.0 .
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:1.0 --push .

# 레이어 캐시 (GitHub Actions 캐시 백엔드)
docker buildx build \
  --cache-from type=gha \
  --cache-to type=gha,mode=max \
  -t myapp:1.0 --push .

docker buildx imagetools inspect myapp:1.0     # 매니페스트·플랫폼 확인
docker history myapp:1.0                       # 레이어별 크기
```

## ECR
```
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin <ACCT>.dkr.ecr.ap-northeast-2.amazonaws.com

aws ecr create-repository --repository-name myapp --image-scanning-configuration scanOnPush=true
aws ecr describe-images --repository-name myapp --query 'sort_by(imageDetails,&imagePushedAt)[-5:]'
aws ecr batch-delete-image --repository-name myapp --image-ids imageTag=old
aws ecr describe-image-scan-findings --repository-name myapp --image-id imageTag=1.0
```

## 이미지 보안
```
trivy image myapp:1.0                                   # 취약점 스캔
trivy image --severity CRITICAL --ignore-unfixed myapp:1.0
trivy image --format sarif -o results.sarif myapp:1.0
trivy image --format cyclonedx -o sbom.json myapp:1.0   # SBOM 생성
syft myapp:1.0 -o spdx-json                             # SBOM (Syft)
grype myapp:1.0                                         # 취약점 (Grype)

cosign sign --key cosign.key myapp:1.0                  # 이미지 서명
cosign verify --key cosign.pub myapp:1.0
cosign sign --yes myapp@sha256:...                      # keyless (OIDC)
```

## argocd CLI
```
argocd login <SERVER> --username admin --password <PW>
argocd login <SERVER> --sso

argocd app list
argocd app get sample-app
argocd app diff sample-app                     # Git vs 클러스터 차이
argocd app sync sample-app                     # 수동 동기화
argocd app sync sample-app --prune --force
argocd app history sample-app                  # 배포 이력(리비전)
argocd app rollback sample-app 3               # 리비전 3으로 롤백
argocd app set sample-app --sync-policy automated
argocd app wait sample-app --health --timeout 300
argocd app delete sample-app --cascade

argocd app manifests sample-app                # 렌더링된 매니페스트
argocd appset list                             # ApplicationSet 목록
argocd proj list
argocd repo add https://github.com/org/repo --username x --password <TOKEN>
argocd cluster list
```

## ArgoCD — kubectl로 다루기
```
kubectl -n argocd get applications
kubectl -n argocd get app sample-app -o yaml
kubectl -n argocd describe app sample-app

# OutOfSync 리소스만 추려보기
kubectl -n argocd get app sample-app -o jsonpath=\
"{range .status.resources[?(@.status=='OutOfSync')]}{.kind}/{.name}{'\n'}{end}"

# 초기 admin 비밀번호
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

kubectl -n argocd port-forward svc/argocd-server 8080:443
kubectl -n argocd logs deploy/argocd-application-controller -f
kubectl -n argocd logs deploy/argocd-repo-server -f

# 강제 새로고침 (캐시된 매니페스트 무시)
kubectl -n argocd annotate app sample-app argocd.argoproj.io/refresh=hard --overwrite

# 삭제가 안 될 때 finalizer 제거
kubectl -n argocd patch app sample-app -p '{"metadata":{"finalizers":null}}' --type merge
```

## Argo Rollouts
```
kubectl argo rollouts list rollouts -n myapp
kubectl argo rollouts get rollout myapp -n myapp --watch    # 진행 상황 실시간
kubectl argo rollouts status myapp -n myapp

kubectl argo rollouts promote myapp -n myapp               # 다음 단계로
kubectl argo rollouts promote myapp -n myapp --full        # 남은 단계 전부 건너뛰기
kubectl argo rollouts abort myapp -n myapp                 # 중단 (안정 버전 유지)
kubectl argo rollouts retry rollout myapp -n myapp
kubectl argo rollouts undo myapp -n myapp                  # 직전 리비전으로
kubectl argo rollouts undo myapp -n myapp --to-revision=3

kubectl argo rollouts set image myapp app=myapp:1.2 -n myapp
kubectl argo rollouts pause myapp -n myapp
kubectl argo rollouts dashboard                            # 로컬 UI (localhost:3100)

kubectl -n myapp get analysisrun                           # 분석 실행 결과
kubectl -n myapp describe analysisrun <NAME>
```

## Helm (파이프라인에서)
```
helm lint ./chart
helm template myapp ./chart -f values-prod.yaml            # 렌더링만 (적용 X)
helm diff upgrade myapp ./chart -f values.yaml             # 변경분 미리보기(플러그인)
helm upgrade --install myapp ./chart -f values.yaml --atomic --timeout 5m
helm rollback myapp 2
helm history myapp
```

## kubectl — 배포 확인·롤백
```
kubectl rollout status deploy/myapp -n myapp --timeout=300s
kubectl rollout history deploy/myapp -n myapp
kubectl rollout undo deploy/myapp -n myapp
kubectl rollout undo deploy/myapp -n myapp --to-revision=2
kubectl rollout restart deploy/myapp -n myapp
kubectl rollout pause deploy/myapp -n myapp

kubectl set image deploy/myapp app=myapp:1.2 -n myapp
kubectl get events -n myapp --sort-by=.lastTimestamp | tail -20
```

## 자주 겪는 상황
```
# ArgoCD가 계속 OutOfSync 인데 diff 가 안 보일 때
argocd app diff sample-app
kubectl -n argocd annotate app sample-app argocd.argoproj.io/refresh=hard --overwrite

# CRD가 커서 client-side apply 가 실패할 때 → Application 에 ServerSideApply=true
kubectl apply --server-side --force-conflicts -f crds.yaml

# Actions 에서 OIDC 인증이 안 될 때 → permissions 확인
#   permissions: { id-token: write, contents: read }

# 배포가 멈췄는지 확인
kubectl rollout status deploy/myapp -n myapp --timeout=60s
kubectl describe deploy/myapp -n myapp | grep -A5 Conditions
```
