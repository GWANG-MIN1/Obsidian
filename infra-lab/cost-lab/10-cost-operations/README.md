# 10 비용 운영

일회성 최적화는 **3개월이면 원상복구된다.** 새 서비스가 뜨고, 실험용 리소스가 남고, 아무도 안 지운다.  
FinOps의 Operate 단계는 **줄이는 것이 아니라 줄어든 상태를 유지하는 것**이다.

---

## 예산과 알림

```json
// budget.json
{
  "BudgetName": "monthly-total",
  "BudgetLimit": { "Amount": "1000", "Unit": "USD" },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": { "TagKeyValue": ["user:Environment$prod"] }
}
```

```json
// notifications.json — 실제로 쓸모 있는 조합
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "team@example.com" }]
  },
  {
    "Notification": {
      "NotificationType": "FORECASTED",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 100,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{ "SubscriptionType": "EMAIL", "Address": "team@example.com" }]
  }
]
```

| 알림 종류 | 언제 유용한가 |
|---|---|
| **FORECASTED > 100%** | **월 초에 미리 안다** ← 가장 중요 |
| ACTUAL > 80% | 여유 있을 때 경고 |
| ACTUAL > 100% | 이미 늦었다 (기록용) |

> **`ACTUAL > 100%`만 걸어두면 항상 사후 통보다.** 월말에 "예산을 넘었습니다"를 받아봐야 할 수 있는 게 없다.  
> **예측 기반 알림이 유일하게 행동 가능한 알림**이다. 관측성의 `predict_linear`와 같은 발상이다. → `../observability-lab/03-promql/`

---

## 이상 탐지

```bash
aws ce get-anomaly-monitors
aws ce get-anomalies --date-interval StartDate=2026-07-01,EndDate=2026-08-01
```

```
Cost Anomaly Detection = 과거 패턴을 학습해 급증을 자동 감지
  → 예산 임계값과 달리 '평소와 다름' 을 잡는다
  → 총액이 예산 안이어도 특정 서비스가 튀면 알려준다
```

> **예산 알림은 총액만 본다.** EC2가 줄고 데이터 전송이 폭증해도 총액이 비슷하면 아무 알림이 없다.  
> 이상 탐지가 그 빈틈을 메운다. 무료이므로 안 켤 이유가 없다.

### 흔한 급증 원인

| 원인 | 징후 |
|---|---|
| 실험용 리소스를 안 지움 | 특정 태그·계정에서 지속 증가 |
| 로그·메트릭 폭증 | CloudWatch·S3 급증 |
| 무한 재시도 루프 | NAT 처리량·API 호출 급증 |
| 잘못된 오토스케일 설정 | EC2 급증 |
| 데이터 전송 (크로스 AZ) | 전송료만 급증 |
| **암호화폐 채굴 (침해)** | EC2 급증 + CPU 100% |

> ⚠️ **비용 급증이 보안 사고의 첫 신호인 경우가 있다.** 탈취된 자격증명으로 GPU 인스턴스를 대량으로 띄우는 게 전형적인 패턴이다.  
> 비용 알림을 보안 관점에서도 읽는다 — 관측성 지표(CPU 급등)와 함께 보면 더 확실하다. → `../security-lab/10-runtime-detection/`

---

## 낭비 정리 루틴

**분기 1회 정해진 날에 돌린다.** "생각날 때"는 안 돌아간다.

```bash
# ① 미연결 EBS 볼륨
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Created:CreateTime}' --output table

# ② 미연결 Elastic IP (연결 안 되면 과금)
aws ec2 describe-addresses --query 'Addresses[?AssociationId==`null`].[PublicIp]' --output table

# ③ 정지된 인스턴스 (EBS 는 계속 과금)
aws ec2 describe-instances --filters Name=instance-state-name,Values=stopped \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,LaunchTime]' --output table

# ④ 타겟 없는 로드밸런서
aws elbv2 describe-target-groups --query 'TargetGroups[?length(LoadBalancerArns)==`0`].TargetGroupName'

# ⑤ 오래된 스냅샷
aws ec2 describe-snapshots --owner-ids self \
  --query 'sort_by(Snapshots,&StartTime)[:20].[SnapshotId,VolumeSize,StartTime]' --output table

# ⑥ 보존 기간 없는 로그 그룹
aws logs describe-log-groups \
  --query 'logGroups[?retentionInDays==`null`].[logGroupName,storedBytes]' --output table

# ⑦ 태그 없는 리소스 (주인 없는 것)
aws resourcegroupstaggingapi get-resources --region ap-northeast-2 \
  | jq -r '.ResourceTagMappingList[] | select((.Tags|length)==0) | .ResourceARN'
```

```bash
# Kubernetes 쪽
kubectl get pvc -A                                  # 안 쓰는 PVC
kubectl get svc -A --field-selector spec.type=LoadBalancer   # LB 개수
kubectl get pods -A --field-selector status.phase=Failed
kubectl get jobs -A -o json | jq -r '.items[]
  | select(.status.succeeded==1) | "\(.metadata.namespace)/\(.metadata.name)"'  # 완료된 Job
```

> **⑦ 태그 없는 리소스가 정리의 출발점이다.** 주인을 모르는 리소스는 지워도 되는지 판단할 수 없어서 영원히 남는다.  
> `ManagedBy` 태그가 없다 = IaC 밖에서 만들어졌다 = 코드에 없다. → `02-cost-visibility/`

---

## 비운영 환경 스케줄링

**가장 절감률이 높고 위험이 낮은 조치다.**

```
주 168시간 중 실제 사용 40~50시간
  24시간 가동  → 100%
  업무시간만   → ~30%
```

```yaml
# ① Kubernetes — 야간 스케일 다운
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-night
  namespace: platform
spec:
  schedule: "0 20 * * 1-5"          # 평일 20시
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: scaler
          restartPolicy: OnFailure
          containers:
            - name: kubectl
              image: bitnami/kubectl:latest
              command:
                - sh
                - -c
                - kubectl scale deploy --all --replicas=0 -n dev
```

```hcl
# ② 노드 그룹 자체를 줄인다
node_min_size     = 0     # 야간에는 0 까지
node_desired_size = 0
```

```bash
# ③ 클러스터를 통째로 파괴 (가장 극단적, 가장 저렴)
terraform destroy -auto-approve
```

> **이 프로젝트의 dev 클러스터는 매일 파괴한다.** IaC가 갖춰져 있으면 재생성 비용이 거의 0이므로, 안 쓸 때 켜둘 이유가 없다.  
> 이 전제가 다른 결정들(PVC 없음, 7일 보존, Spot)을 정당화한다는 점이 중요하다 — **하나의 아키텍처 결정이 여러 비용 결정을 연쇄적으로 가능하게 만든다.** → `04-compute-purchasing/`

---

## 비용을 코드로

```hcl
# Terraform — 기본값이 조직 전체 비용을 결정한다
variable "single_nat_gateway" {
  description = "Use a single NAT gateway to save cost in non-prod."
  type        = bool
  default     = true
}

variable "node_capacity_type" {
  description = "Node capacity type: SPOT (cheap, interruptible) or ON_DEMAND."
  type        = string
  default     = "SPOT"
}
```

> **변수 `description`에 비용 트레이드오프를 적어두면 다음 사람이 판단할 수 있다.** "왜 SPOT인가"를 코드가 설명한다.  
> 플랫폼 팀의 역할은 감시가 아니라 **기본값을 효율적으로 만드는 것**이다. → `01-finops-basics/`

### 정책으로 강제

```yaml
# Kyverno — 비용 관련 라벨·리소스 강제
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-cost-controls
spec:
  rules:
    - name: require-team-label
      match:
        any: [{ resources: { kinds: [Deployment, StatefulSet] } }]
      validate:
        failureAction: Audit
        message: "비용 배분을 위해 team 라벨이 필요합니다."
        pattern:
          metadata:
            labels:
              team: "?*"
```

```yaml
# ResourceQuota — 팀별 상한
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    services.loadbalancers: "2"      # LB 개수 제한
    persistentvolumeclaims: "10"
```

> **`services.loadbalancers` 쿼터가 의외로 유용하다.** `type=LoadBalancer` 남발로 LB 고정비가 쌓이는 걸 구조적으로 막는다. → `08-storage-network-cost/`  
> 정책은 **Audit부터 시작**한다 — 보안 정책과 같은 원칙이다. → `../security-lab/04-kyverno/`

### PR 단계에서 비용 보기

```yaml
# Infracost — Terraform plan 의 예상 비용을 PR 에 코멘트
- name: Infracost
  run: |
    infracost breakdown --path terraform/environments/prod \
      --format json --out-file /tmp/infracost.json
    infracost comment github --path /tmp/infracost.json \
      --repo $GITHUB_REPOSITORY --pull-request ${{ github.event.number }} \
      --github-token ${{ secrets.GITHUB_TOKEN }}
```

```
"이 PR 은 월 $340 를 추가합니다"  ← 머지 전에 보인다
```

> **비용을 코드 리뷰의 대상으로 만드는 가장 직접적인 방법이다.** plan 결과를 PR에 코멘트하는 것과 같은 발상이다. → `../cicd-lab/03-github-actions-advanced/` `../terraform-lab/10-cicd-policy/`

---

## 리포팅

| 대상 | 주기 | 내용 |
|---|---|---|
| **엔지니어** | 주 1회 | Slack — 자기 팀 비용, 전주 대비 |
| 팀 리드 | 월 1회 | 추세, 단위 비용, 상위 항목 |
| 경영 | 월 1회 | 총액, 예측, 단위 경제성 |

```
주간 Slack 요약 예시
  team-a: $1,240 (전주 대비 +12%)  ← 증가 원인: 신규 배치 잡
  team-b: $890  (-3%)
  플랫폼 공통: $2,100 (관측성 $980)
  미분류: $45 (2%)                 ← 이 값이 10% 넘으면 태그를 손본다
```

> **엔지니어에게 안 보여주면 비용을 만드는 사람이 결과를 모른다.** 주간 Slack 요약 하나가 대시보드 열 개보다 효과가 크다. → `02-cost-visibility/`  
> 다만 **비난 도구로 쓰이면 즉시 실패한다.** "team-a가 제일 많이 씀"이 아니라 "team-a의 단위 비용이 개선됨"으로 읽히게 만든다.

---

## 운영 루틴 정리

```
매주
  □ 팀별 비용 Slack 요약
  □ 이상 탐지 알림 확인

매월
  □ 예측 대비 실제 검토
  □ 단위 경제성 지표 (요청당·MAU당)
  □ 상위 5개 서비스 변동 원인

분기
  □ 낭비 정리 루틴 (미연결 EBS·EIP·스냅샷·LB)
  □ 라이트사이징 재검토 (VPA 추천값 대조)
  □ Savings Plans 커버리지·활용률 점검
  □ 태그 미분류 비율 확인
  □ 관측성 카디널리티·보존 재검토
```

```bash
# 커밋 활용률 — 남는 커밋은 그냥 버려지는 돈
aws ce get-savings-plans-utilization \
  --time-period Start=2026-07-01,End=2026-08-01
```

> **커버리지(얼마나 커밋으로 덮었나)와 활용률(산 커밋을 다 썼나)을 둘 다 본다.** 활용률이 낮으면 과매수한 것이다.

---

## 배운 점

- **일회성 최적화는 3개월이면 원상복구된다** — Operate 단계가 본체다
- **`FORECASTED > 100%` 예측 알림만이 행동 가능한 알림**이다 (ACTUAL은 사후 통보)
- **Cost Anomaly Detection은 총액이 아니라 "평소와 다름"** 을 잡는다 — 무료다
- ⚠️ **비용 급증이 보안 사고의 첫 신호**일 수 있다 (탈취된 키로 인스턴스 대량 생성)
- 낭비 정리는 **분기 1회 정해진 날에** — "생각날 때"는 안 돌아간다
- 정리 대상: 미연결 EBS·EIP, 정지 인스턴스, 타겟 없는 LB, 오래된 스냅샷, 보존 없는 로그 그룹
- **태그 없는 리소스가 정리의 출발점** — 주인을 모르면 지울 수 없다
- **비운영 환경 스케줄링이 절감률 최고·위험 최저**
- IaC가 있으면 **재생성 비용이 0에 가까워** 매일 파괴가 가능해진다
- 하나의 아키텍처 결정(매일 파괴)이 **여러 비용 결정을 연쇄적으로 정당화**한다
- Terraform 변수 `description`에 **비용 트레이드오프를 적어둔다**
- 플랫폼 팀의 역할은 감시가 아니라 **기본값을 효율적으로 만드는 것**
- **`services.loadbalancers` 쿼터**로 LB 고정비 누적을 구조적으로 막는다
- 비용 정책도 **Audit부터** 시작한다
- **Infracost로 PR에 예상 비용을 코멘트** — 비용을 코드 리뷰 대상으로
- 주간 Slack 요약 하나가 대시보드 열 개보다 효과적이다
- ⚠️ **리포팅이 비난 도구가 되면 즉시 실패한다** — 단위 비용 개선으로 읽히게 만든다
- 커밋은 **커버리지와 활용률을 둘 다** 본다 (활용률이 낮으면 과매수)
