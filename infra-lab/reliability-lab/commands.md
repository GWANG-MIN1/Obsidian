# 신뢰성 명령어 레퍼런스

## SLO · 에러 버짓 (PromQL)
```
# SLI: 성공률 (가용성)
sum(rate(http_requests_total{status!~"5.."}[30d]))
  / sum(rate(http_requests_total[30d]))

# SLI: 지연 준수율 (300ms 이내 비율)
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[30d]))
  / sum(rate(http_request_duration_seconds_count[30d]))

# 에러 버짓 소진율 (0=미사용, 1=전부 소진)
(1 - (sum(rate(http_requests_total{status!~"5.."}[30d]))
      / sum(rate(http_requests_total[30d]))))
  / (1 - 0.999)

# 번레이트 (에러율 / 허용 에러율)
(sum(rate(http_requests_total{status=~"5.."}[5m]))
 / sum(rate(http_requests_total[5m]))) / (1 - 0.999)

# 남은 에러 버짓 (분)
(1 - ((1 - (sum(rate(http_requests_total{status!~"5.."}[30d]))
            / sum(rate(http_requests_total[30d])))) / (1 - 0.999)))
  * 30 * 24 * 60
```

## 장애 진단 — 첫 5분
```
kubectl get pods -A --field-selector status.phase!=Running
kubectl get nodes
kubectl get events -A --sort-by=.lastTimestamp | tail -30

kubectl -n myapp describe pod <POD>                # Events 섹션이 핵심
kubectl -n myapp logs <POD> --tail=100
kubectl -n myapp logs <POD> --previous             # 재시작 전 로그
kubectl -n myapp logs -l app=myapp --tail=50 --all-containers

kubectl top nodes
kubectl top pods -A --sort-by=memory

# 최근 변경 확인 — 장애의 대부분은 변경에서 온다
kubectl -n argocd get applications
kubectl -n myapp rollout history deploy/myapp
git -C . log --oneline -10
```

## 파드 상태별 원인
```
# Pending — 스케줄 불가
kubectl -n myapp describe pod <POD> | grep -A10 Events
kubectl describe node <NODE> | grep -A8 "Allocated resources"

# CrashLoopBackOff
kubectl -n myapp logs <POD> --previous
kubectl -n myapp get pod <POD> -o jsonpath='{.status.containerStatuses[0].lastState}'

# OOMKilled 확인
kubectl get pods -A -o json | jq -r '.items[]
  | select(.status.containerStatuses[]?.lastState.terminated.reason=="OOMKilled")
  | "\(.metadata.namespace)/\(.metadata.name)"'

# ImagePullBackOff
kubectl -n myapp describe pod <POD> | grep -A5 "Failed to pull"
```

## 완화 조치
```
kubectl -n myapp rollout undo deploy/myapp                # 직전 버전으로
kubectl -n myapp rollout undo deploy/myapp --to-revision=3
kubectl -n myapp scale deploy/myapp --replicas=10         # 용량 확보
kubectl -n myapp rollout restart deploy/myapp             # 재기동

kubectl cordon <NODE>                                     # 신규 스케줄 차단
kubectl drain <NODE> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <NODE>

# GitOps 환경의 정식 롤백
git revert <COMMIT> && git push
# 응급 시 자동 동기화 일시 중지
argocd app set myapp --sync-policy none
```

## PDB · 우선순위
```
kubectl get pdb -A
kubectl -n myapp describe pdb myapp-pdb        # ALLOWED DISRUPTIONS 확인
kubectl get priorityclass
kubectl get pods -A -o custom-columns=\
NS:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priorityClassName
```

## Velero — 백업·복구
```
velero install --provider aws --bucket my-backup-bucket \
  --backup-location-config region=ap-northeast-2 \
  --plugins velero/velero-plugin-for-aws:v1.10.0

velero backup create myapp-backup --include-namespaces myapp
velero backup create full --exclude-namespaces kube-system
velero backup describe myapp-backup --details
velero backup logs myapp-backup
velero backup get

velero schedule create daily --schedule="0 2 * * *" --ttl 720h
velero schedule get

velero restore create --from-backup myapp-backup
velero restore create --from-backup myapp-backup --namespace-mappings myapp:myapp-restore
velero restore describe <RESTORE> --details
velero restore logs <RESTORE>
```

## etcd (자체 관리 클러스터)
```
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%F).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl snapshot status /backup/etcd.db --write-out=table
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db --data-dir /var/lib/etcd-restore
# EKS 는 컨트롤 플레인이 관리형이라 etcd 를 직접 다루지 않는다
```

## AWS 백업
```
aws rds describe-db-snapshots --db-instance-identifier mydb \
  --query 'sort_by(DBSnapshots,&SnapshotCreateTime)[-5:].[DBSnapshotIdentifier,SnapshotCreateTime]'
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mydb --target-db-instance-identifier mydb-restore \
  --restore-time 2026-08-09T10:00:00Z

aws ec2 describe-snapshots --owner-ids self --filters Name=tag:Backup,Values=daily
aws backup list-backup-jobs --by-state COMPLETED
aws backup list-recovery-points-by-backup-vault --backup-vault-name Default
```

## 부하 테스트
```
# k6
k6 run --vus 50 --duration 5m script.js
k6 run --stage 2m:100,5m:100,2m:0 script.js       # 램프업

# vegeta
echo "GET https://api.example.com/health" | vegeta attack -rate=100 -duration=60s \
  | vegeta report

# hey
hey -z 60s -c 50 https://api.example.com/

# 클러스터 내부에서
kubectl run loadtest --rm -it --image=williamyeh/wrk --restart=Never -- \
  -t4 -c100 -d60s http://myapp.myapp.svc/
```

## 카오스 실험
```
# 수동 — 가장 먼저 해볼 것
kubectl -n myapp delete pod <POD>                          # 파드 하나
kubectl drain <NODE> --ignore-daemonsets                   # 노드 하나
kubectl -n myapp scale deploy/dependency --replicas=0      # 의존성 제거

# Chaos Mesh
kubectl apply -f podchaos.yaml
kubectl get podchaos -A
kubectl describe podchaos <NAME>

# LitmusChaos
kubectl apply -f chaosengine.yaml
kubectl get chaosresult -A
```

## 신뢰성 지표 (PromQL)
```
# 파드 재시작
increase(kube_pod_container_status_restarts_total[1h]) > 0

# 원하는 레플리카를 못 채운 Deployment
kube_deployment_spec_replicas != kube_deployment_status_replicas_available

# 노드 NotReady
kube_node_status_condition{condition="Ready", status="true"} == 0

# 파드 축출
increase(kube_pod_status_reason{reason="Evicted"}[1h])

# CPU 스로틀링 (라이트사이징이 과했는지)
sum by (namespace,pod) (rate(container_cpu_cfs_throttled_seconds_total[5m]))

# PDB 로 인해 축출 불가한 상태
kube_poddisruptionbudget_status_pod_disruptions_allowed == 0
```
