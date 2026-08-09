# 02 비용 가시성

**"이 리소스가 누구 것인가"에 답할 수 없으면 아무것도 줄일 수 없다.** 지워도 되는지 모르니 아무도 안 지운다.  
가시성의 핵심은 화려한 대시보드가 아니라 **태그**다. 태그가 없으면 어떤 도구를 붙여도 "미분류" 덩어리만 커진다.

---

## 태그 전략

```
비용 배분의 최소 단위 = 태그
  → 태그가 없는 리소스는 영원히 "누구 것인지 모르는 비용"
```

### 최소 태그 세트

| 태그 | 값 예시 | 용도 |
|---|---|---|
| **Project** | `eks-gitops-platform` | 프로젝트별 배분 |
| **Environment** | `dev` / `stg` / `prod` | 환경별 배분, 정리 대상 판단 |
| **Owner** / **Team** | `platform` | 책임 주체 |
| **ManagedBy** | `terraform` | **수동 생성 리소스 식별** |
| CostCenter | `CC-1234` | 회계 배분 |

> **`ManagedBy`가 의외로 중요하다.** 이 태그가 없는 리소스 = 콘솔에서 손으로 만든 것 = 코드에 없어서 아무도 모르는 것 = 지워지지 않는 것.  
> 유휴 자원 정리의 첫 단추가 "IaC 밖에서 만들어진 리소스 찾기"다. → `../terraform-lab/07-data-import/`

### Terraform에서 일괄 부착

```hcl
provider "aws" {
  region = var.region

  default_tags {                    # 이 프로바이더로 만드는 모든 리소스에 자동 부착
    tags = {
      Project     = "eks-gitops-platform"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}
```

> **`default_tags`는 태그 누락을 구조적으로 없앤다.** 리소스마다 `tags`를 쓰면 반드시 빠뜨린다.  
> 다만 **모든 리소스가 태그를 지원하지는 않는다.** 또 일부 리소스는 `default_tags`와 자체 `tags`가 충돌해 perpetual diff를 만들기도 한다 — plan에 매번 태그 변경이 뜨면 이걸 의심한다.

### Kubernetes 쪽 라벨

```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/part-of: platform
    team: backend                      # 비용 배분 기준
    environment: prod
```

> AWS 태그는 **노드·볼륨·LB** 수준까지만 간다. **파드 단위 비용은 Kubernetes 라벨로 배분**해야 한다. → `03-kubernetes-cost-allocation/`

---

## ⚠️ 비용 할당 태그 활성화

태그를 붙이기만 하면 Cost Explorer에 나오지 않는다. **콘솔에서 명시적으로 활성화**해야 한다.

```bash
aws ce list-cost-allocation-tags --status Active

aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active TagKey=Environment,Status=Active
```

```
⚠️ 활성화는 소급 적용되지 않는다
   오늘 활성화하면 오늘 이후 데이터부터 태그별로 보인다
   → 프로젝트 시작 시점에 켜두는 게 맞다
```

> **이걸 몰라서 "태그를 다 붙였는데 Cost Explorer에 안 나온다"로 헤매는 경우가 많다.**  
> 활성화 후에도 데이터가 반영되기까지 24시간 정도 걸린다.

---

## 태그 강제

붙이는 걸 사람의 성실함에 맡기면 반드시 빠진다.

```hcl
# ① Terraform — default_tags (가장 확실)
```

```yaml
# ② Kyverno — 라벨 없는 워크로드 차단
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-cost-labels
spec:
  rules:
    - name: require-team-label
      match:
        any:
          - resources:
              kinds: [Deployment, StatefulSet]
      validate:
        failureAction: Audit          # 익숙해지면 Enforce
        message: "비용 배분을 위해 team 라벨이 필요합니다."
        pattern:
          metadata:
            labels:
              team: "?*"
```

```json
// ③ AWS Tag Policy (Organizations) — 태그 값 형식 강제
{
  "tags": {
    "Environment": {
      "tag_key": { "@@assign": "Environment" },
      "tag_value": { "@@assign": ["dev", "stg", "prod"] }
    }
  }
}
```

> **Kyverno 정책도 Audit부터 시작한다.** 태그 정책을 Enforce로 바로 켜면 기존 워크로드 배포가 전부 막힌다. → `../security-lab/04-kyverno/`

---

## Cost Explorer

```bash
# 서비스별 (가장 먼저 보는 뷰)
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# 태그별
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=Project

# 일별 — 급증 시점 찾기
aws ce get-cost-and-usage \
  --time-period Start=2026-07-01,End=2026-08-01 \
  --granularity DAILY --metrics UnblendedCost
```

### 비용 지표 종류

| 지표 | 의미 |
|---|---|
| **UnblendedCost** | 실제 청구액 (**기본으로 이걸 본다**) |
| BlendedCost | Organizations 평균 단가 적용 |
| AmortizedCost | 선결제 RI·SP를 기간에 나눠 배분 |
| NetUnblendedCost | 할인 적용 후 |
| UsageQuantity | 사용량 (시간·GB) |

> **선결제 커밋이 있으면 `AmortizedCost`를 본다.** `UnblendedCost`는 결제한 달에 몰려 찍혀서 월별 추세가 왜곡된다.

---

## 계정 분리 — 가장 강력한 배분 수단

```
태그 기반 배분: 한 계정 안에서 나눈다 (누락·오타 위험)
계정 분리:     애초에 섞이지 않는다  (배분이 아니라 격리)
```

| | 태그 | 계정 분리 |
|---|---|---|
| 배분 정확도 | 태그 품질에 의존 | **100%** |
| 실수 격리 | 없음 | **prod와 dev가 물리적으로 분리** |
| 권한 경계 | IAM 정책 | 계정 경계 |
| 운영 부담 | 낮음 | 높음 (Organizations·SSO 필요) |

```
AWS Organizations
├── prod 계정        ← 청구서가 따로 나온다
├── stg 계정
├── dev 계정
└── shared 계정 (ECR, Route53)
```

> **환경별 계정 분리는 비용 배분과 보안 격리를 동시에 해결한다.** dev에서 실수로 `terraform destroy`를 쳐도 prod가 안전하다.  
> Terraform의 환경 분리 전략(디렉터리 + 백엔드 분리)이 계정 분리와 자연스럽게 맞물린다. → `../terraform-lab/08-workspace-environment/`

---

## CUR (Cost and Usage Report)

Cost Explorer로 안 되는 분석이 필요할 때 쓴다.

```
Cost Explorer : UI·API, 최대 13개월, 집계된 뷰
CUR           : S3 로 내려오는 원시 데이터, 시간 단위·리소스 단위
                → Athena·QuickSight 로 임의 분석
```

```sql
-- Athena 예시: 태그별 일별 비용
SELECT
  line_item_usage_start_date AS day,
  resource_tags_user_project AS project,
  SUM(line_item_unblended_cost) AS cost
FROM cur_table
WHERE line_item_usage_start_date >= DATE '2026-07-01'
GROUP BY 1, 2
ORDER BY 3 DESC;
```

> **CUR은 처음부터 켜둔다.** 나중에 필요해졌을 때 과거 데이터를 소급해서 받을 수 없다. S3 저장 비용은 미미하다.

---

## 대시보드 설계

```
┌────────────────────────────────────────┐
│ 1행: 이번 달 예상 / 지난 달 대비        │  ← 3초
├────────────────────────────────────────┤
│ 2행: 서비스별 상위 5개 (막대)           │  ← 어디에 쓰나
├────────────────────────────────────────┤
│ 3행: 팀·환경별 추이 (시계열)            │  ← 누가 쓰나
├────────────────────────────────────────┤
│ 4행: 단위 비용 (요청당·MAU당)           │  ← 효율은?
├────────────────────────────────────────┤
│ 5행: 미분류(태그 없음) 비율             │  ← 가시성 품질
└────────────────────────────────────────┘
```

> **"미분류 비용 비율"을 지표로 삼는 게 실질적이다.** 이 값이 10%를 넘으면 배분 논의가 무의미해진다.  
> 관측성 대시보드와 같은 원칙 — 위에서 아래로 좁혀지게, 한 화면에. → `../observability-lab/04-grafana/`

### 보여주는 주기

| 대상 | 주기 | 형태 |
|---|---|---|
| 엔지니어 | 주 1회 | Slack 요약 (자기 팀 비용) |
| 팀 리드 | 월 1회 | 추세 + 단위 비용 |
| 경영 | 월 1회 | 총액 + 예측 + 단위 경제성 |

> **엔지니어에게 안 보여주면 비용을 만드는 사람이 결과를 모른다.** 주간 Slack 요약 하나가 대시보드 열 개보다 효과가 크다.

---

## 배운 점

- **"누구 것인가"에 답할 수 없으면 아무것도 못 줄인다** — 지워도 되는지 모르니 안 지운다
- 최소 태그: **Project / Environment / Owner / ManagedBy**
- **`ManagedBy` 태그가 IaC 밖에서 만들어진 리소스를 드러낸다** (정리의 첫 단추)
- Terraform **`default_tags`** 로 태그 누락을 구조적으로 없앤다
- 일부 리소스는 `default_tags`와 자체 `tags`가 충돌해 perpetual diff를 만든다
- AWS 태그는 노드·볼륨까지 — **파드 단위는 Kubernetes 라벨**로 배분
- ⚠️ **태그를 붙여도 "비용 할당 태그"를 활성화하지 않으면 Cost Explorer에 안 나온다**
- ⚠️ **활성화는 소급 적용되지 않는다** — 프로젝트 시작 시점에 켠다
- 태그 강제는 **Terraform default_tags + Kyverno + AWS Tag Policy** 3중으로
- 기본 지표는 **`UnblendedCost`**, 선결제 커밋이 있으면 **`AmortizedCost`**
- **계정 분리는 배분이 아니라 격리** — 정확도 100%, 보안 격리까지
- **CUR은 처음부터 켜둔다** — 과거 데이터는 소급해서 못 받는다
- **"미분류 비용 비율"을 지표로** 삼는다 (10% 넘으면 배분 논의가 무의미)
- 엔지니어에게 **주간 Slack 요약** 하나가 대시보드 열 개보다 효과적이다
