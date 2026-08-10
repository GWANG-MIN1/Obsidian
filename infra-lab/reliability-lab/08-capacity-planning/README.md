# 08 용량 계획

용량 부족은 **가장 예측 가능한 장애 원인**이다. 트래픽은 하루아침에 10배가 되지 않는다.  
그런데도 용량 장애가 자주 나는 이유는 **아무도 추세를 안 보고 있어서**다.

---

## 무엇을 재고 무엇을 예측하는가

```
수요 (Demand)          →  요청 수, 동시 사용자, 데이터 증가량
공급 (Capacity)        →  파드 수, 노드 자원, DB 커넥션, 대역폭
여유 (Headroom)        →  공급 - 수요
```

```
용량 계획 = "언제 여유가 바닥나는가" 를 미리 아는 것
```

| 자원 | 한계 지점 |
|---|---|
| CPU·메모리 | 노드 자원 |
| **노드당 파드 수** | **t3.medium = max-pods 17** (ENI 기반) |
| DB 커넥션 | 커넥션 풀·max_connections |
| 디스크 | PV 용량, 로그 증가 |
| 네트워크 | 대역폭, NAT 처리량 |
| API 서버 | 요청 레이트 (대규모 클러스터) |

> ⚠️ **CPU·메모리가 남는데 파드가 Pending이면 max-pods를 의심한다.** 작은 인스턴스에서는 이게 먼저 걸린다.  
> DaemonSet(CNI·kube-proxy·promtail·node-exporter)이 5~6개를 먹으므로 실제 가용 슬롯은 더 적다. → `../cost-lab/07-rightsizing/`

---

## 추세로 예측하기

```promql
# 4시간 뒤 디스크가 찰 것인가
predict_linear(node_filesystem_avail_bytes[6h], 4*3600) < 0

# 7일 뒤 노드 CPU requests 가 allocatable 을 넘는가
predict_linear(sum(kube_pod_container_resource_requests{resource="cpu"})[7d:1h], 7*86400)
  > sum(kube_node_status_allocatable{resource="cpu"})

# 현재 클러스터 여유율
1 - (sum(kube_pod_container_resource_requests{resource="cpu"})
     / sum(kube_node_status_allocatable{resource="cpu"}))
```

```yaml
- alert: DiskWillFillIn4Hours
  expr: predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs"}[6h], 4*3600) < 0
  for: 15m
  labels: { severity: warning }
  annotations:
    summary: "{{ $labels.instance }} 디스크가 4시간 내 가득 찰 예정"
```

> ⭐ **`predict_linear`가 용량 알림의 핵심이다.** "이미 90%다"가 아니라 **"이대로면 4시간 뒤 찬다"** 를 알려서 대응할 시간을 준다. → `../observability-lab/03-promql/`  
> 단, 선형 예측이므로 **급격한 변화는 못 잡는다.** 추세 감시용이지 스파이크 대응용이 아니다.

### 성장 패턴

```
선형 성장   사용자 증가에 비례        → 예측이 쉽다
계단 성장   신규 고객 온보딩          → 일정을 미리 안다
계절성      프로모션·월말·연말        → 캘린더로 대비
지수 성장   바이럴                   → 예측 불가, 오토스케일링에 의존
```

> **계절성과 이벤트는 예측 가능한데도 놓치는 경우가 많다.** 프로모션 일정이 배포·인프라 팀에 공유되지 않는 게 전형적인 기여 요인이다. → `04-postmortem/`

---

## 오토스케일링 3계층

```
① HPA   파드 수를 늘린다            (초~분)
② VPA   파드 크기를 바꾼다          (재시작 필요)
③ CA / Karpenter  노드를 늘린다     (분)
```

```
트래픽 증가
  → HPA 가 파드를 늘린다
    → 노드 자원 부족 → 파드 Pending
      → Karpenter 가 노드를 띄운다 (1~2분)
        → 파드 스케줄 → 이미지 pull → 기동 → readiness
  ─────────────────────────────────────────
  총 2~5분. 그동안의 트래픽은 기존 파드가 받아야 한다
```

> ⭐ **오토스케일링은 즉시가 아니다.** 급격한 스파이크는 오토스케일링으로 못 막는다.  
> 그래서 **헤드룸(여유 용량)이 필요**하다 — 스케일이 따라올 때까지 버틸 만큼.

### 헤드룸 확보

```yaml
# 저우선순위 더미 파드로 노드 여유를 미리 확보 (over-provisioning)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1                    # 음수 = 다른 파드에 밀린다
globalDefault: false
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
spec:
  replicas: 2
  template:
    spec:
      priorityClassName: overprovisioning
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
          resources:
            requests: { cpu: "1", memory: 2Gi }
```

```
평소: 더미 파드가 노드 자원을 점유 (실제로는 아무것도 안 함)
급증: 실제 파드가 더미를 선점(preempt) → 즉시 스케줄
      동시에 Karpenter 가 새 노드를 띄워 더미를 복구
```

> **"노드가 뜨기를 기다리는 시간"을 없애는 기법이다.** 비용을 지불하고 응답 속도를 사는 것 — 명시적인 트레이드오프다. → `../cost-lab/06-karpenter/`

### HPA 설정 주의

```yaml
spec:
  minReplicas: 3               # 1 로 두면 스케일 아웃이 느리다
  maxReplicas: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60      # 70~80은 여유가 부족하다
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0   # 확장은 빠르게
    scaleDown:
      stabilizationWindowSeconds: 300 # 축소는 천천히 (플래핑 방지)
```

> ⚠️ **HPA의 사용률은 requests 대비 비율**이다. requests를 과대하게 잡으면 사용률이 항상 낮게 나와 **스케일아웃이 안 된다.**  
> 라이트사이징과 HPA를 함께 봐야 하는 이유다. → `../cost-lab/07-rightsizing/`  
> **축소를 빠르게 하면 플래핑**(늘었다 줄었다)이 생긴다. 확장은 즉시, 축소는 5분 이상.

---

## 부하 테스트

```
목적별로 다른 테스트
  로드 테스트    예상 부하에서 정상 동작하는가
  스트레스 테스트 한계가 어디인가 (언제 무너지는가)
  스파이크 테스트 급증에 견디는가
  소크 테스트    장시간 유지 시 문제(메모리 누수)가 있는가
```

```javascript
// k6 — 램프업으로 한계 찾기
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // 워밍업
    { duration: '5m', target: 100 },   // 유지
    { duration: '2m', target: 300 },   // 증가
    { duration: '5m', target: 300 },
    { duration: '2m', target: 0 },     // 감소
  ],
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://api.example.com/items');
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

> ⚠️ **스테이징 부하가 운영의 1/50이면 부하 문제는 절대 안 드러난다.** 이게 커넥션 풀 고갈 같은 장애가 운영에서만 나는 이유다. → `04-postmortem/`  
> 부하 테스트는 **운영과 비슷한 규모**에서 하거나, 아예 운영에서(카나리로 소량) 한다.

### 무엇을 관찰할 것인가

```
□ 언제 p95 지연이 꺾이는가        ← 여기가 실질적 한계
□ 언제 에러가 나기 시작하는가
□ 병목이 어디인가 (CPU? DB 커넥션? 스레드 풀?)
□ 오토스케일링이 따라오는가
□ 부하 종료 후 정상 복귀하는가    ← 회복도 검증 대상
```

> **"몇 rps까지 되는가"보다 "무엇이 먼저 무너지는가"가 중요하다.** 병목을 알아야 증설 대상을 안다.

---

## 한계를 미리 아는 것들

```
클러스터
  □ 노드당 max-pods (t3.medium = 17)
  □ 서브넷 IP 개수 (VPC CNI 는 파드마다 IP 를 쓴다)
  □ 노드 그룹 max_size

애플리케이션
  □ DB max_connections
  □ 커넥션 풀 크기 × 파드 수 ≤ DB 한계   ← 자주 놓친다
  □ 스레드 풀 크기
  □ 파일 디스크립터 한계

클라우드
  □ EC2 인스턴스 쿼터 (vCPU 기준)
  □ Elastic IP 개수
  □ API 요청 레이트 리밋
```

> ⭐ **"파드 수 × 커넥션 풀 크기"가 DB 한계를 넘는 게 전형적인 사고다.** HPA로 파드를 30개까지 늘리게 해뒀는데 DB는 100 커넥션만 받는다면, 파드당 풀 10개일 때 10개 파드에서 이미 한계다.  
> **오토스케일링 상한을 정할 때 하류 의존성의 한계를 함께 계산**한다.

```bash
# 서브넷 IP 여유 확인 — 파드가 늘면 IP 도 늘어난다
aws ec2 describe-subnets --subnet-ids subnet-0abc \
  --query 'Subnets[].[SubnetId,AvailableIpAddressCount,CidrBlock]' --output table

# EC2 쿼터
aws service-quotas get-service-quota --service-code ec2 \
  --quota-code L-1216C47A            # Running On-Demand Standard instances
```

---

## 용량 리뷰 루틴

```
주간
  □ 클러스터 여유율 추이
  □ predict_linear 알림 발생 여부

월간
  □ 수요 성장률 vs 용량 증가율
  □ 오토스케일링 상한이 여전히 적절한가
  □ 하류 의존성(DB·외부 API) 한계 대비 여유

분기
  □ 부하 테스트 재실행
  □ 클라우드 쿼터 검토
  □ 성수기·이벤트 일정 반영
```

```promql
# 대시보드에 둘 것
1 - (sum(kube_pod_container_resource_requests{resource="cpu"})
     / sum(kube_node_status_allocatable{resource="cpu"}))        # 여유율

sum(kube_pod_status_phase{phase="Pending"})                      # Pending 파드
count(kube_node_info)                                            # 노드 수 추이
```

> **Pending 파드가 지속되면 용량 부족의 직접 신호다.** 일시적인 건 정상이지만 몇 분 이상 지속되면 확인한다.

---

## 배운 점

- 용량 부족은 **가장 예측 가능한 장애 원인**인데 아무도 추세를 안 봐서 터진다
- 한계는 CPU·메모리만이 아니다 — **max-pods, DB 커넥션, 서브넷 IP, 쿼터**
- ⚠️ **자원이 남는데 Pending이면 max-pods를 의심**한다
- ⭐ **`predict_linear`로 "이대로면 언제 찬다"** 를 알린다 (임계값 알림보다 낫다)
- 선형 예측이므로 **급격한 변화는 못 잡는다** — 추세 감시용
- 계절성·이벤트는 **예측 가능한데도 공유가 안 돼서** 놓친다
- ⭐ **오토스케일링은 즉시가 아니다** (노드 확보까지 2~5분) — 헤드룸이 필요하다
- 저우선순위 **더미 파드(over-provisioning)** 로 노드 여유를 미리 확보할 수 있다
- ⚠️ **HPA 사용률은 requests 대비** — requests가 과대하면 스케일아웃이 안 된다
- **확장은 즉시, 축소는 5분 이상** (플래핑 방지)
- ⚠️ **스테이징 부하가 운영의 1/50이면 부하 문제는 절대 안 드러난다**
- 부하 테스트는 "몇 rps"보다 **"무엇이 먼저 무너지는가"** 를 본다
- **부하 종료 후 정상 복귀하는지**도 검증 대상이다
- ⭐ **"파드 수 × 커넥션 풀 크기"가 DB 한계를 넘는 게 전형적 사고**
- 오토스케일링 상한은 **하류 의존성의 한계를 함께 계산**해서 정한다
- **Pending 파드 지속은 용량 부족의 직접 신호**
