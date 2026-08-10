# 07 백업과 재해 복구

**백업은 있는데 복구를 해본 적이 없다면 백업이 없는 것과 같다.**  
이 장의 핵심은 백업 도구가 아니라 **RPO/RTO를 정하고, 복구를 실제로 리허설하는 것**이다.

---

## RPO와 RTO

```
       장애 발생
          │
  ────────┼────────────────────▶ 시간
     ◀────┤          ├────▶
      RPO │          │ RTO
  마지막 백업       복구 완료

RPO (Recovery Point Objective)  얼마만큼의 데이터 손실을 감수할 것인가
RTO (Recovery Time Objective)   얼마 만에 복구할 것인가
```

| RPO | 필요한 것 |
|---|---|
| 24시간 | 일 1회 백업 |
| 1시간 | 시간별 백업 또는 증분 |
| 5분 | 지속적 백업 (PITR) |
| **0** | 동기 복제 (비용·지연 대가) |

| RTO | 필요한 것 |
|---|---|
| 며칠 | 백업에서 수동 복구 |
| 몇 시간 | 자동화된 복구 절차 |
| 몇 분 | 대기(standby) 환경 |
| **0에 가까움** | 액티브-액티브 (가장 비쌈) |

> **RPO/RTO는 기술 결정이 아니라 사업 결정이다.** "얼마나 잃어도 되는가"는 엔지니어가 혼자 정할 수 없다.  
> ⚠️ **둘 다 0을 요구하면 비용이 폭발한다.** 데이터 종류별로 다르게 정하는 게 현실적이다 — 결제 기록은 RPO 0, 추천 캐시는 재생성 가능.

---

## 무엇을 백업할 것인가

```
Kubernetes 클러스터
├── 클러스터 리소스 정의     → ⭐ Git 이 이미 백업이다 (GitOps)
├── PersistentVolume 데이터  → 백업 필요
├── etcd (관리형은 AWS)      → EKS 는 신경 안 써도 된다
├── 시크릿                   → 외부 저장소(SSM)에 원본
└── 클러스터 자체            → Terraform 이 재생성
```

| 대상 | 백업 수단 | 비고 |
|---|---|---|
| **매니페스트·설정** | **Git** | GitOps면 이미 완료 |
| **인프라** | **Terraform 코드 + state** | state 백업이 중요 |
| PV 데이터 | Velero, EBS 스냅샷 | |
| **DB** | RDS 자동 백업 + PITR | 가장 중요 |
| 객체 스토리지 | S3 버전 관리 + 복제 | |
| 시크릿 | SSM·Secrets Manager | Git에는 참조만 |

> ⭐ **GitOps와 IaC를 쓰면 백업의 절반이 이미 끝나 있다.** 클러스터가 통째로 날아가도 `terraform apply` + ArgoCD 부트스트랩으로 재구축된다. → `../terraform-lab/09-aws-vpc-eks/` `../cicd-lab/06-app-of-apps-applicationset/`  
> 남는 건 **상태(데이터)** 다. 그게 백업의 진짜 대상이다.

### 매일 파괴하는 클러스터의 함의

```
dev 클러스터를 매일 destroy → 다음 날 apply
      ↓
클러스터 재구축을 매일 리허설하고 있는 셈이다
      ↓
"복구 절차가 동작하는가" 에 대한 답을 이미 갖고 있다
```

> **이게 IaC + GitOps의 숨은 이점이다.** 재해 복구 훈련을 따로 할 필요 없이 일상이 훈련이 된다.  
> 다만 **데이터가 없는 환경이므로 데이터 복구는 검증되지 않는다.** 그 부분은 별도 리허설이 필요하다.

---

## Terraform state — 잊기 쉬운 백업 대상

```
state 를 잃으면
  → Terraform 이 기존 리소스를 모르게 된다
  → apply 하면 전부 새로 만들려고 한다  💥
  → 수십 개 리소스를 import 로 되찾아야 한다
```

```hcl
terraform {
  backend "s3" {
    bucket       = "my-tfstate-bucket"
    key          = "dev/terraform.tfstate"
    encrypt      = true
    use_lockfile = true
  }
}
```

```bash
aws s3api put-bucket-versioning --bucket my-tfstate-bucket \
  --versioning-configuration Status=Enabled     # ⭐ 필수
```

> ⭐ **S3 버전 관리가 state 복구의 유일한 수단이다.** 깨졌을 때 이전 버전으로 되돌린다.  
> 부트스트랩 버킷에 `prevent_destroy = true`를 거는 이유도 같다. → `../terraform-lab/03-state/`

---

## Velero — Kubernetes 백업

```bash
velero install --provider aws --bucket my-backup-bucket \
  --backup-location-config region=ap-northeast-2 \
  --plugins velero/velero-plugin-for-aws:v1.10.0
```

```bash
# 네임스페이스 단위 백업
velero backup create myapp-backup --include-namespaces myapp

# 정기 백업
velero schedule create daily --schedule="0 2 * * *" --ttl 720h

# 복구
velero restore create --from-backup myapp-backup

# 다른 네임스페이스로 복구 (리허설에 유용)
velero restore create --from-backup myapp-backup \
  --namespace-mappings myapp:myapp-restore-test
```

| Velero가 하는 것 | 안 하는 것 |
|---|---|
| 리소스 정의 백업 | 애플리케이션 정합성 보장 |
| PV 스냅샷 (CSI 연동) | DB의 트랜잭션 일관성 |
| 네임스페이스 단위 복구 | 클러스터 밖 리소스 |

> ⚠️ **동작 중인 DB의 볼륨 스냅샷은 정합성이 보장되지 않는다.** 파일시스템 레벨 복사이므로 쓰기 중인 데이터가 깨질 수 있다.  
> DB는 **DB 자체의 백업 메커니즘**(RDS 스냅샷, `pg_dump`)을 쓴다. Velero는 그 앞뒤로 훅을 걸 수 있다.

```yaml
# 백업 전후 훅 — DB 를 잠깐 정지시키거나 flush
metadata:
  annotations:
    pre.hook.backup.velero.io/command: '["/bin/sh","-c","fsfreeze --freeze /data"]'
    post.hook.backup.velero.io/command: '["/bin/sh","-c","fsfreeze --unfreeze /data"]'
```

> **GitOps 환경에서 Velero의 역할은 제한적이다.** 매니페스트는 Git에 있으므로 복구할 필요가 없고, **PV 데이터만이 진짜 대상**이다.  
> 다만 **잘못 지운 리소스를 빠르게 되돌리는 용도**로는 여전히 유용하다.

---

## 데이터베이스

```bash
# RDS — 자동 백업 + PITR (Point-In-Time Recovery)
aws rds modify-db-instance --db-instance-identifier mydb \
  --backup-retention-period 7 --preferred-backup-window "17:00-18:00"

# 특정 시점으로 복구 (실수로 DELETE 한 경우)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier mydb \
  --target-db-instance-identifier mydb-restore \
  --restore-time 2026-08-09T10:00:00Z
```

> ⭐ **PITR이 실무에서 가장 자주 쓰이는 복구 수단이다.** 재해가 아니라 **"방금 잘못된 UPDATE를 돌렸다"** 같은 인적 실수에 쓴다.  
> **복구는 원본을 덮어쓰지 않고 새 인스턴스로** 한다. 원본을 살려둬야 비교·검증이 가능하다.

| 복구 시나리오 | 수단 |
|---|---|
| 실수로 데이터 삭제 | PITR |
| 인스턴스 장애 | Multi-AZ 자동 페일오버 |
| AZ 장애 | Multi-AZ |
| 리전 장애 | 크로스 리전 리드 레플리카 |
| 논리적 손상(버그) | 스냅샷 + PITR |

> ⚠️ **복제(replication)는 백업이 아니다.** `DROP TABLE`은 레플리카에도 즉시 복제된다.  
> 백업은 **시점을 되돌릴 수 있어야** 백업이다.

---

## ⭐ 복구 리허설

```
백업이 있다 ≠ 복구할 수 있다

실제로 실패하는 지점들
  □ 백업이 며칠째 실패하고 있었다 (알림이 없어서 몰랐다)
  □ 백업 파일이 손상돼 있었다
  □ 복구 절차 문서가 오래돼 안 맞는다
  □ 복구 권한이 없다
  □ 복구는 됐는데 앱이 그 데이터를 못 읽는다
  □ 복구에 예상보다 10배 오래 걸린다 (RTO 초과)
```

```
분기 1회 리허설
  1. 실제 백업에서 별도 환경으로 복구
  2. 소요 시간 측정 → RTO 와 비교
  3. 데이터 정합성 검증 (건수·최신 레코드)
  4. 애플리케이션이 실제로 동작하는지 확인
  5. 절차 문서 갱신
```

> ⭐ **"롤백을 실제로 해본 적 있는가"** 와 같은 질문이다. 장애 중에 처음 해보는 절차는 대부분 실패한다. → `03-incident-response/`  
> **리허설은 프로덕션이 아닌 곳에서** 한다. `--namespace-mappings`로 다른 네임스페이스에 복구하면 안전하다.

### 백업 자체를 모니터링한다

```promql
# 백업이 최근에 성공했는가
time() - velero_backup_last_successful_timestamp > 86400 * 2

# 백업 실패
increase(velero_backup_failure_total[1d]) > 0
```

```yaml
- alert: BackupNotRecent
  expr: time() - velero_backup_last_successful_timestamp > 172800
  for: 1h
  labels: { severity: warning }
  annotations:
    summary: "48시간 이상 성공한 백업이 없습니다"
```

> ⚠️ **조용히 실패하는 백업이 가장 위험하다.** 필요할 때가 되어서야 안다.  
> 백업 성공 여부는 반드시 알림 대상이다. → `../observability-lab/05-alerting/`

---

## 재해 복구 전략

| 전략 | RTO | 비용 | 설명 |
|---|---|---|---|
| **백업 & 복구** | 시간~일 | 낮음 | 백업만 다른 리전에, 필요 시 구축 |
| **파일럿 라이트** | 수십 분 | 중간 | 핵심(DB)만 대기, 나머지는 필요 시 |
| **웜 스탠바이** | 분 | 높음 | 축소된 전체 스택이 상시 가동 |
| **멀티 사이트 액티브** | ~0 | **매우 높음** | 양쪽에서 트래픽 처리 |

```
IaC + GitOps 가 있으면 '백업 & 복구' 의 RTO 가 크게 줄어든다
  terraform apply (20분) + ArgoCD 부트스트랩 (10분) + 데이터 복구
      ↓
  파일럿 라이트에 가까운 RTO 를 백업 & 복구 비용으로
```

> ⭐ **이게 IaC의 신뢰성 측면 이점이다.** 인프라를 코드로 재현할 수 있으면 DR 비용 구조가 통째로 달라진다.  
> 단, **다른 리전에 실제로 만들어본 적이 있어야** 한다. AMI·인스턴스 타입·서비스 가용성이 리전마다 다르다.

### 멀티 리전의 현실

```
어려운 것
  □ 데이터 동기화 (지연 vs 정합성)
  □ 상태를 가진 것들 (세션·캐시)
  □ DNS 전환 시간 (TTL)
  □ 비용 2배
  □ 평소에 안 쓰는 환경은 반드시 썩는다  ← 가장 큰 문제
```

> ⚠️ **쓰지 않는 대기 환경은 반드시 낡는다.** 설정 드리프트, 버전 차이, 만료된 인증서.  
> 그래서 **정기적으로 트래픽을 흘려보는 것**(또는 아예 액티브-액티브)이 실질적이다.

---

## 백업 위생

```
□ 3-2-1 원칙: 복사본 3개, 매체 2종, 오프사이트 1개
□ 백업을 다른 계정·리전에 (같은 계정이 털리면 백업도 털린다)
□ 백업에 대한 쓰기 권한 최소화 (랜섬웨어 대비)
□ 암호화 (전송·저장)
□ 보존 기간 정의 (무한 보존은 비용)
□ 복구 절차 문서화 + 정기 리허설
```

> ⚠️ **백업을 같은 계정에 두면 계정이 침해됐을 때 백업도 함께 삭제된다.** 별도 계정 + 객체 잠금(Object Lock)이 랜섬웨어 대비의 표준이다. → `../security-lab/10-runtime-detection/`

---

## 배운 점

- ⭐ **복구를 해본 적 없는 백업은 백업이 아니다**
- **RPO(데이터 손실 허용)와 RTO(복구 시간)는 사업 결정** — 데이터 종류별로 다르게
- ⚠️ 둘 다 0을 요구하면 비용이 폭발한다
- ⭐ **GitOps + IaC면 백업의 절반이 이미 끝나 있다** — 남는 건 데이터
- **매일 파괴하는 dev 클러스터는 재구축을 매일 리허설**하는 셈이다
- 단, 데이터가 없으므로 **데이터 복구는 별도 검증**이 필요하다
- ⭐ **Terraform state의 S3 버전 관리가 state 복구의 유일한 수단**
- ⚠️ **동작 중인 DB의 볼륨 스냅샷은 정합성이 보장되지 않는다** — DB 자체 백업을 쓴다
- GitOps 환경에서 Velero는 **PV 데이터와 실수 복구**용으로 제한적
- ⭐ **PITR이 실무에서 가장 자주 쓰이는 복구 수단** (재해가 아니라 인적 실수에)
- 복구는 **원본을 덮어쓰지 않고 새 인스턴스로**
- ⚠️ **복제는 백업이 아니다** — `DROP TABLE`은 레플리카에도 복제된다
- 실패 지점: 백업이 며칠째 실패 / 문서가 오래됨 / RTO 초과 / 앱이 데이터를 못 읽음
- **분기 1회 리허설**하고 소요 시간을 RTO와 비교한다
- ⚠️ **조용히 실패하는 백업이 가장 위험** — 백업 성공 여부를 알림 대상으로
- DR 전략 4단계: 백업&복구 / 파일럿 라이트 / 웜 스탠바이 / 액티브-액티브
- ⭐ **IaC가 있으면 낮은 비용으로 짧은 RTO**를 얻는다 — 단, 다른 리전에 실제로 만들어봐야 한다
- ⚠️ **쓰지 않는 대기 환경은 반드시 낡는다** — 정기적으로 트래픽을 흘린다
- **백업을 다른 계정·리전에** 둔다 (계정 침해 시 백업도 함께 삭제된다)
