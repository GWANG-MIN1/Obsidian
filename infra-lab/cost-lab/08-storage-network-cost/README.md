# 08 스토리지·네트워크 비용

컴퓨트를 최적화하고 나면 **"왜 EC2 말고 이런 것들이 이렇게 나오지"** 하는 항목들이 남는다.  
데이터 전송료와 NAT Gateway가 대표적이다 — **아무도 인스턴스처럼 세어보지 않아서 조용히 쌓인다.**

---

## EBS

| 타입 | 특징 | 비용 |
|---|---|---|
| **gp3** | 기본 3000 IOPS·125MB/s 포함, 성능을 용량과 독립 설정 | **gp2보다 ~20% 저렴** |
| gp2 | IOPS가 용량에 비례 (3 IOPS/GB) | 레거시 |
| io1/io2 | 고IOPS 보장 | 비싸다 |
| st1/sc1 | 처리량형·콜드 (HDD) | 저렴, 랜덤 IO 부적합 |

```bash
# gp2 → gp3 전환 대상 찾기
aws ec2 describe-volumes --filters Name=volume-type,Values=gp2 \
  --query 'Volumes[].{ID:VolumeId,Size:Size,AZ:AvailabilityZone}' --output table

# 무중단 전환 (온라인 변경)
aws ec2 modify-volume --volume-id vol-0abc123 --volume-type gp3
```

> **gp2 → gp3는 무중단이고 대부분의 워크로드에서 더 싸고 빠르다.** 가장 손쉬운 절감 항목 중 하나다.  
> Karpenter의 `EC2NodeClass`나 노드 그룹 launch template에서 **기본값을 gp3로** 해두면 새 노드부터 자동 적용된다. → `06-karpenter/`

### 유휴 EBS — 조용한 낭비

```
인스턴스를 종료해도 deleteOnTermination=false 인 볼륨은 남는다
PVC 를 지워도 reclaimPolicy: Retain 이면 PV 가 남는다
      ↓
아무도 안 쓰는 볼륨이 계속 과금된다
```

```bash
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Created:CreateTime}' --output table
```

```yaml
# StorageClass 기본값
reclaimPolicy: Delete       # PVC 삭제 시 PV·EBS 도 삭제
allowVolumeExpansion: true
```

> **`reclaimPolicy: Retain`은 데이터 보호에는 좋지만 정리 부담을 만든다.** 운영 DB라면 Retain, 그 외에는 Delete가 합리적이다. → `../k8s-manifests/06-storage/`  
> **정지된 EC2 인스턴스의 EBS는 계속 과금된다.** "껐으니 안 나가겠지"는 틀렸다.

### 스냅샷

```bash
aws ec2 describe-snapshots --owner-ids self \
  --query 'sort_by(Snapshots,&StartTime)[:20].{ID:SnapshotId,Size:VolumeSize,Date:StartTime}' \
  --output table
```

> 자동 백업 스냅샷은 **보존 정책이 없으면 무한히 쌓인다.** Data Lifecycle Manager로 보존 기간을 건다.

---

## S3

| 클래스 | 용도 | 검색 |
|---|---|---|
| **Standard** | 자주 접근 | 즉시 |
| **Intelligent-Tiering** | **접근 패턴을 모를 때 (권장 기본값)** | 즉시 |
| Standard-IA | 월 1회 미만 접근 | 즉시 (검색 요금) |
| Glacier Instant | 분기 1회 | 즉시 (검색 요금↑) |
| Glacier Flexible / Deep Archive | 아카이브 | 분~시간 단위 |

```json
// 수명주기 정책
{
  "Rules": [
    {
      "ID": "tier-and-expire",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/" },
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 90,  "StorageClass": "GLACIER_IR" }
      ],
      "Expiration": { "Days": 365 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    }
  ]
}
```

> ⚠️ **`AbortIncompleteMultipartUpload`를 빼먹지 않는다.** 실패한 멀티파트 업로드 조각은 목록에 안 보이는데 **계속 과금된다.** 오래된 버킷에서 수십 GB가 나오는 경우가 흔하다.  
> **Intelligent-Tiering이 무난한 기본값이다.** 접근 패턴을 분석해 자동으로 계층을 옮긴다(소액의 모니터링 수수료).  
> 버전 관리를 켠 버킷은 **이전 버전도 과금된다** — `NoncurrentVersionExpiration`을 함께 건다.

---

## ⚠️ NAT Gateway — 숨은 비용 1순위

```
NAT Gateway 비용 = 시간당 요금 + 처리 데이터 GB당 요금
                     (고정)         (변동, 이게 크다)
```

```
프라이빗 서브넷의 파드가 인터넷으로 나갈 때마다 NAT 를 거친다
  ├─ 컨테이너 이미지 pull        ← 매 배포·매 노드 생성
  ├─ 패키지 다운로드
  ├─ 외부 API 호출
  └─ S3·ECR·DynamoDB 접근       ← ⭐ 이건 NAT 없이 갈 수 있다
```

### VPC 엔드포인트로 우회

```hcl
# Gateway 엔드포인트 — 무료
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids
}

# Interface 엔드포인트 — 시간당 요금이 있지만 NAT 처리료보다 싼 경우가 많다
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  private_dns_enabled = true
}
```

| 엔드포인트 | 대상 | 요금 |
|---|---|---|
| **Gateway** | S3, DynamoDB | **무료** |
| Interface (PrivateLink) | ECR, STS, SSM, CloudWatch 등 | 시간당 + 데이터 처리 |

> **S3·DynamoDB Gateway 엔드포인트는 무료다.** 안 쓸 이유가 없다.  
> **ECR Interface 엔드포인트는 계산이 필요하다** — 이미지 pull 트래픽이 크면 NAT 처리료보다 싸다. 노드가 자주 교체되는(Spot·Karpenter) 환경이면 특히 유리하다.

### 단일 NAT vs AZ별 NAT

```hcl
single_nat_gateway = true    # dev: NAT 1개 — 비용 1/N
                             # prod: false — AZ 마다 1개
```

| | 단일 NAT | AZ별 NAT |
|---|---|---|
| 비용 | **1/N** | N배 |
| 가용성 | **그 AZ 장애 시 전체 아웃바운드 중단** | AZ 독립 |
| 크로스 AZ 전송료 | **발생** (다른 AZ → NAT AZ) | 없음 |

> **단일 NAT는 비용을 줄이지만 크로스 AZ 전송료를 만든다.** 트래픽이 많으면 절감이 상쇄될 수 있다.  
> dev처럼 트래픽이 적고 가용성 요구가 낮은 환경에서만 명확한 이득이다 — 그래서 `single_nat_gateway = true`가 dev 기본값이고 prod에서는 뒤집는다. → `../terraform-lab/09-aws-vpc-eks/`

```bash
# NAT 처리량 확인 — 비용의 근거
aws cloudwatch get-metric-statistics --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --start-time 2026-08-01T00:00:00Z --end-time 2026-08-09T00:00:00Z \
  --period 86400 --statistics Sum \
  --dimensions Name=NatGatewayId,Value=nat-0abc123
```

---

## 데이터 전송료

```
같은 AZ 내부                    무료
크로스 AZ (같은 리전)            양방향 각각 과금  ← ⭐ 자주 놓친다
리전 간                         비싸다
인터넷 아웃바운드                가장 비싸다 (CloudFront 경유 시 저렴)
인터넷 인바운드                  무료
```

### 크로스 AZ가 조용히 쌓이는 경로

```
파드 A (AZ-a) ──▶ Service ──▶ 파드 B (AZ-c)     크로스 AZ 과금
앱 (AZ-a) ──▶ RDS 프라이머리 (AZ-c)             크로스 AZ 과금
Prometheus (AZ-a) ──▶ 모든 노드 스크랩           크로스 AZ 과금
```

> **Kubernetes Service는 기본적으로 AZ를 신경 쓰지 않고 라우팅한다.** 파드가 3 AZ에 흩어져 있으면 요청의 2/3이 크로스 AZ로 나간다.

```yaml
# 토폴로지 인식 라우팅 — 같은 AZ 를 우선한다
apiVersion: v1
kind: Service
metadata:
  name: backend
  annotations:
    service.kubernetes.io/topology-mode: Auto
```

```yaml
# 또는 trafficDistribution (1.31+)
spec:
  trafficDistribution: PreferClose
```

> ⚠️ **AZ 분산(가용성)과 크로스 AZ 전송료(비용)는 정면으로 충돌한다.**  
> `topology-mode: Auto`는 같은 AZ에 엔드포인트가 충분할 때만 동작한다 — 부족하면 자동으로 전체 라우팅으로 돌아간다. 그래서 안전한 편이지만, **AZ당 레플리카가 충분해야 효과가 있다.**  
> 가용성을 깎아서 전송료를 아끼는 건 위험하다. → `01-finops-basics/`

### 인터넷 아웃바운드

```
S3 → 인터넷 직접        비싸다
S3 → CloudFront → 인터넷  더 싸다 (오리진→CloudFront 전송 무료)
```

> **정적 파일·미디어를 직접 서빙하고 있다면 CloudFront를 앞에 두는 것만으로 전송료가 준다.** 캐시 히트만큼 오리진 요청도 줄어든다.

---

## 로드밸런서

```
Service type=LoadBalancer 를 만들 때마다 NLB/CLB 가 하나씩 생긴다
  → 서비스 10개 = LB 10개 = 고정비 10배
```

```yaml
# Ingress 하나로 여러 서비스를 묶는다 → ALB 1개
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/group.name: shared    # 여러 Ingress 가 ALB 를 공유
```

> **`type=LoadBalancer`를 남발하면 LB 고정비가 쌓인다.** 내부 통신은 ClusterIP로 충분하고, 외부 노출은 Ingress로 묶는다.  
> ALB Ingress Controller의 `group.name`을 쓰면 **여러 Ingress가 ALB 하나를 공유**한다. → `../k8s-manifests/07-ingress/`

---

## 점검 순서

```
1. Cost Explorer 서비스별 → EC2 다음으로 큰 게 뭔가
2. 데이터 전송 항목이 크다  → NAT? 크로스 AZ? 인터넷 아웃바운드?
3. NAT 처리량 확인          → VPC 엔드포인트 대상 식별
4. 유휴 EBS·EIP·스냅샷 정리
5. gp2 → gp3 전환
6. S3 수명주기 정책
7. LB 개수 확인
```

```bash
# 미연결 Elastic IP (연결 안 되면 과금)
aws ec2 describe-addresses --query 'Addresses[?AssociationId==`null`].[PublicIp,AllocationId]' --output table

# 타겟 없는 LB
aws elbv2 describe-target-groups --query 'TargetGroups[?length(LoadBalancerArns)==`0`].TargetGroupName'
```

---

## 배운 점

- 컴퓨트 다음으로 큰 항목은 대개 **데이터 전송료와 NAT Gateway**
- **gp2 → gp3는 무중단이고 더 싸고 빠르다** — 가장 손쉬운 절감
- Karpenter·launch template의 **기본 볼륨 타입을 gp3로** 해둔다
- **정지된 인스턴스의 EBS는 계속 과금된다**
- `reclaimPolicy: Retain`은 안전하지만 **유휴 볼륨을 만든다**
- 스냅샷은 보존 정책이 없으면 무한히 쌓인다 (DLM)
- S3 기본값은 **Intelligent-Tiering**이 무난하다
- ⚠️ **`AbortIncompleteMultipartUpload`를 빼먹지 않는다** — 안 보이는데 과금된다
- 버전 관리 버킷은 **이전 버전도 과금** (`NoncurrentVersionExpiration`)
- ⚠️ **NAT Gateway가 숨은 비용 1순위** — 시간당 + 처리 GB당
- **S3·DynamoDB Gateway 엔드포인트는 무료** — 안 쓸 이유가 없다
- ECR Interface 엔드포인트는 **이미지 pull이 잦으면 NAT보다 싸다**
- **단일 NAT는 크로스 AZ 전송료를 만든다** — 트래픽이 많으면 절감이 상쇄된다
- **크로스 AZ는 양방향 각각 과금** — Service가 기본적으로 AZ를 안 가린다
- `topology-mode: Auto` / `trafficDistribution: PreferClose`로 같은 AZ 우선
- ⚠️ **AZ 분산(가용성)과 전송료(비용)는 정면 충돌** — 가용성을 깎지 않는다
- 인터넷 아웃바운드는 **CloudFront 경유가 더 싸다**
- **`type=LoadBalancer` 남발은 LB 고정비를 곱한다** — Ingress로 묶는다
