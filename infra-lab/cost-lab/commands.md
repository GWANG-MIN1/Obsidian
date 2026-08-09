# 비용 명령어 레퍼런스

## Cost Explorer (aws ce)
```
# 월별 서비스별 비용
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# 태그별 비용 (비용 할당 태그 활성화 필요)
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=Project

# 일별 추이 (급증 찾기)
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity DAILY --metrics UnblendedCost

# 특정 서비스만 필터
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity DAILY --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}'

# 태그가 없는 리소스 비용 (낭비 후보)
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=Project \
  | jq -r '.ResultsByTime[].Groups[] | select(.Keys[0]=="Project$") | .Metrics'

aws ce get-cost-forecast \
  --time-period Start=2026-08-10,End=2026-09-01 \
  --metric UNBLENDED_COST --granularity MONTHLY

aws ce get-rightsizing-recommendation --service AmazonEC2
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP --term-in-years ONE_YEAR --payment-option NO_UPFRONT
```

## 예산 · 이상 탐지
```
aws budgets describe-budgets --account-id 123456789012
aws budgets create-budget --account-id 123456789012 --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json

aws ce get-anomalies --date-interval StartDate=2026-07-01,EndDate=2026-08-01
aws ce get-anomaly-monitors
aws ce create-anomaly-monitor --anomaly-monitor file://monitor.json
```

## 태그 점검
```
aws ce list-cost-allocation-tags --status Active
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active

# 태그 없는 리소스 찾기
aws resourcegroupstaggingapi get-resources --region ap-northeast-2 \
  | jq -r '.ResourceTagMappingList[] | select((.Tags|length)==0) | .ResourceARN'

# 특정 태그가 빠진 리소스
aws resourcegroupstaggingapi get-resources --region ap-northeast-2 \
  | jq -r '.ResourceTagMappingList[]
           | select([.Tags[].Key] | index("Project") | not) | .ResourceARN'
```

## 유휴 자원 찾기 (낭비 정리)
```
# 미연결 EBS 볼륨
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}' --output table

# 미연결 Elastic IP (연결 안 되면 과금)
aws ec2 describe-addresses --query 'Addresses[?AssociationId==`null`].[PublicIp,AllocationId]' --output table

# 오래된 스냅샷
aws ec2 describe-snapshots --owner-ids self \
  --query 'sort_by(Snapshots,&StartTime)[:20].{ID:SnapshotId,Size:VolumeSize,Date:StartTime}' --output table

# 사용 안 하는 로드밸런서
aws elbv2 describe-target-groups \
  --query 'TargetGroups[?length(LoadBalancerArns)==`0`].TargetGroupName'

# 정지된 인스턴스 (EBS 는 계속 과금된다)
aws ec2 describe-instances --filters Name=instance-state-name,Values=stopped \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,LaunchTime]' --output table

# gp2 → gp3 전환 대상
aws ec2 describe-volumes --filters Name=volume-type,Values=gp2 \
  --query 'Volumes[].{ID:VolumeId,Size:Size}' --output table
```

## S3 비용
```
aws s3api list-buckets --query 'Buckets[].Name' --output text
aws s3 ls s3://my-bucket --recursive --summarize | tail -3

# 수명주기 정책 확인 (없으면 비용이 계속 쌓인다)
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket

# 스토리지 클래스별 분포
aws s3api list-objects-v2 --bucket my-bucket \
  --query 'Contents[].StorageClass' --output text | sort | uniq -c

aws s3api put-bucket-lifecycle-configuration --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

## Kubernetes 리소스 사용량
```
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pods -A --sort-by=cpu

# 노드별 requests/limits 총량 (오버커밋 확인)
kubectl describe node <NODE> | grep -A8 "Allocated resources"

# requests 미설정 파드 찾기 (스케줄링·비용 배분 불가)
kubectl get pods -A -o json | jq -r '.items[]
  | select(.spec.containers[].resources.requests == null)
  | "\(.metadata.namespace)/\(.metadata.name)"'

# 네임스페이스별 requests 합계
kubectl get pods -A -o json | jq -r '.items[]
  | .metadata.namespace as $ns
  | .spec.containers[].resources.requests.cpu // "0"
  | "\($ns)\t\(.)"' | sort | uniq -c

kubectl get resourcequota -A
kubectl get limitrange -A
kubectl get hpa -A
```

## 실사용 대비 요청량 (PromQL)
```
# CPU: 실제 사용량 / requests — 낮으면 과대 요청
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!=""}[7d]))
  / sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu"})

# 메모리 최대 사용량 대비 requests
max_over_time(container_memory_working_set_bytes{container!=""}[7d])
  / on(namespace,pod,container) kube_pod_container_resource_requests{resource="memory"}

# 노드 유휴율 (allocatable 대비 requests)
1 - (sum(kube_pod_container_resource_requests{resource="cpu"})
     / sum(kube_node_status_allocatable{resource="cpu"}))

# Spot 노드 비율
count(kube_node_labels{label_eks_amazonaws_com_capacity_type="SPOT"})
  / count(kube_node_info)
```

## OpenCost / Kubecost
```
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost -n opencost --create-namespace

kubectl -n opencost port-forward svc/opencost 9003:9003
curl -s "http://localhost:9003/allocation/compute?window=7d&aggregate=namespace" | jq
curl -s "http://localhost:9003/allocation/compute?window=1d&aggregate=label:team" | jq

# Kubecost UI
kubectl -n kubecost port-forward svc/kubecost-cost-analyzer 9090:9090
```

## Karpenter
```
kubectl get nodepool
kubectl get ec2nodeclass
kubectl get nodeclaim
kubectl describe nodeclaim <NAME>
kubectl -n karpenter logs deploy/karpenter -f

# consolidation 동작 확인
kubectl -n karpenter logs deploy/karpenter | grep -i "consolidat\|disrupt"

kubectl get nodes -L karpenter.sh/nodepool,node.kubernetes.io/instance-type,karpenter.sh/capacity-type
```

## Spot
```
# 현재 Spot 가격
aws ec2 describe-spot-price-history --instance-types t3.medium \
  --product-descriptions "Linux/UNIX" --max-items 5 \
  --query 'SpotPriceHistory[].[AvailabilityZone,SpotPrice,Timestamp]' --output table

# 중단 빈도·절감률 (Spot Placement Score)
aws ec2 get-spot-placement-scores --instance-types t3.medium m5.large \
  --target-capacity 10 --region-names ap-northeast-2

# 노드의 capacity type
kubectl get nodes -L eks.amazonaws.com/capacityType
kubectl get nodes -L karpenter.sh/capacity-type
```

## 자주 하는 점검
```
# 이번 달 급증한 서비스 찾기
aws ce get-cost-and-usage --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  | jq -r '.ResultsByTime[].Groups[] | [.Keys[0], .Metrics.UnblendedCost.Amount] | @tsv' \
  | sort -k2 -rn | head -10

# NAT Gateway 데이터 처리량 (숨은 비용 1순위)
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination --start-time 2026-08-01T00:00:00Z \
  --end-time 2026-08-09T00:00:00Z --period 86400 --statistics Sum \
  --dimensions Name=NatGatewayId,Value=nat-0abc123
```
