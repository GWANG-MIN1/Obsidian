# 08 배포 전략

"새 버전으로 바꾼다"는 한 문장에 **다운타임을 허용할지, 리소스를 두 배 쓸지, 일부 사용자에게 먼저 보여줄지**가 전부 들어있다.  
전략 선택은 취향이 아니라 **무엇을 감당할 수 있는가**의 문제다.

---

## 한눈에 비교

| 전략 | 다운타임 | 추가 리소스 | 롤백 속도 | 두 버전 공존 | 난이도 |
|---|---|---|---|---|---|
| **Recreate** | **있다** | 없음 | 느림 | ❌ | 매우 쉬움 |
| **RollingUpdate** | 없음 | 소량 | 보통 | ✅ (혼재) | 쉬움 (기본값) |
| **Blue-Green** | 없음 | **2배** | **즉시** | ✅ (분리) | 보통 |
| **Canary** | 없음 | 소량 | 빠름 | ✅ (비율) | 어려움 |
| **A/B 테스트** | 없음 | 소량 | 빠름 | ✅ (규칙) | 어려움 |

> Canary와 A/B는 비슷해 보이지만 목적이 다르다. **Canary는 "안전한가"를 묻고(기술적 검증), A/B는 "어느 쪽이 나은가"를 묻는다(제품 실험).**

---

## Recreate

```
v1 ████████  →  (전부 종료)  →  v2 ████████
                    ↑
                 다운타임
```

```yaml
spec:
  strategy:
    type: Recreate
```

| 쓸 때 | |
|---|---|
| 두 버전이 **동시에 뜨면 안 될 때** | DB 스키마가 호환되지 않는 마이그레이션 |
| 단일 인스턴스 리소스 | RWO 볼륨을 점유하는 워크로드 |
| 배치·내부 도구 | 짧은 다운타임이 문제되지 않는 곳 |

> **의외로 필요한 경우가 있다.** RWO(ReadWriteOnce) PVC를 쓰는 파드는 새 파드가 볼륨을 못 붙어 롤링이 멈춘다. → `../k8s-manifests/06-storage/`

---

## RollingUpdate — Kubernetes 기본값

```
v1 ████████
v1 ██████ v2 ██          maxSurge:       기존 대비 몇 개까지 더 띄울지
v1 ████ v2 ████          maxUnavailable: 몇 개까지 없어도 되는지
v1 ██ v2 ██████
       v2 ████████
```

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # 개수 또는 "25%"
      maxUnavailable: 0      # 용량을 유지하려면 0
  minReadySeconds: 10        # Ready 후 이만큼 더 기다린다
  progressDeadlineSeconds: 600   # 이 시간 넘으면 실패 처리
```

| 설정 | 효과 |
|---|---|
| `maxUnavailable: 0` + `maxSurge: 1` | **용량을 절대 줄이지 않는다** (느리지만 안전) |
| `maxUnavailable: 25%` | 빠르지만 순간 용량이 준다 |
| `minReadySeconds` | Ready 직후 죽는 파드를 거르는 완충 |

### 롤링의 전제 — probe가 정확해야 한다

```yaml
readinessProbe:              # 트래픽을 받을 준비가 됐는가
  httpGet: { path: /readyz, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5

livenessProbe:               # 살아있는가 (실패 시 재시작)
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 15
  periodSeconds: 10
```

> **readinessProbe가 없거나 부정확하면 롤링 업데이트는 무의미하다.** 파드가 Running이 되자마자 트래픽을 받는데 아직 준비가 안 됐으면 그 사이 요청이 전부 실패한다.  
> `/healthz`가 항상 200을 반환하는 껍데기라면 **깨진 버전이 그대로 전량 배포된다.** 롤링의 안전성은 전적으로 probe에 달려 있다. → `../k8s-manifests/02-pod/`

### 우아한 종료

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]     # 엔드포인트 전파를 기다린다
```

```
파드 종료 시작
  ├─ Endpoints 에서 제거 (전파에 시간이 걸린다)
  └─ SIGTERM 전송
        ↑ 이 둘이 동시에 일어나서, 이미 라우팅된 요청이 끊길 수 있다
        → preStop 으로 몇 초 버티면 대부분 해결된다
```

> **배포 중 5xx가 조금씩 나는 원인 1순위가 이것이다.** 앱이 SIGTERM을 받고 진행 중인 요청을 마저 처리하도록 구현하는 것도 필요하다.

---

## Blue-Green

```
        ┌── Service (selector: version=blue)
        │
  Blue  ████████  (v1, 트래픽 100%)
  Green ████████  (v2, 트래픽 0% — 배포는 끝나 있음)

        ↓ 검증 후 Service selector 를 green 으로 전환

        ┌── Service (selector: version=green)
        │
  Blue  ████████  (v1, 대기 — 롤백용)
  Green ████████  (v2, 트래픽 100%)
```

```yaml
# Service 의 selector 한 줄을 바꾸는 것이 전환
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: green        # blue → green
  ports:
    - port: 80
      targetPort: 8080
```

| 장점 | 단점 |
|---|---|
| **롤백이 즉시** (selector 되돌리기) | **리소스 2배** |
| 전환 전에 완전히 검증 가능 | DB 스키마 호환이 필수 |
| 두 버전이 섞이지 않는다 | 비용이 크다 |

> **가장 큰 이점은 롤백 속도다.** 문제가 생기면 selector 한 줄로 즉시 돌아간다 — 이미지를 다시 당길 필요가 없다.  
> 전환 전에 `preview` Service로 내부 검증(스모크 테스트)을 돌릴 수 있다는 것도 실질적인 장점이다.

---

## Canary

일부 트래픽만 새 버전에 보내고, 지표를 보며 점진적으로 늘린다.

```
v2에 5%  ──지표 정상──▶ 20% ──정상──▶ 50% ──정상──▶ 100%
             │
        지표 악화 → 즉시 0% (롤백)
```

### 방법 1 — 레플리카 비율 (Service만으로)

```
v1 파드 9개 + v2 파드 1개, 같은 Service selector
  → 대략 10% 트래픽이 v2 로
```

| 장점 | 단점 |
|---|---|
| 추가 도구 불필요 | **비율이 부정확** (레플리카 수로만 조절) |
| | 파드 1개 = 최소 단위, 세밀한 제어 불가 |

### 방법 2 — 트래픽 라우팅 (Ingress/서비스메시)

```yaml
# NGINX Ingress 카나리 어노테이션
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"       # 10%
    # 또는 헤더 기반
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
```

```
정확한 비율 제어 가능
  → Argo Rollouts 가 이걸 자동화한다 → 09-argo-rollouts/
```

> **수동 카나리는 사람이 지표를 보고 판단해야 해서 실전에서 잘 안 돌아간다.** 새벽 배포에 누가 그래프를 보고 있을 것인가.  
> 카나리의 진짜 가치는 **자동 분석 + 자동 롤백**과 결합할 때 나온다.

---

## ⚠️ 데이터베이스 호환성 — 진짜 어려운 부분

무중단 배포의 전제는 **두 버전이 동시에 같은 DB를 쓸 수 있어야 한다**는 것이다.

```
롤링/카나리/블루그린 전부
  → v1 과 v2 가 잠시(또는 오래) 공존한다
  → v2 용으로 컬럼을 DROP 하면 그 순간 v1 이 깨진다  💥
```

### Expand-Contract (확장-수축) 패턴

```
1단계 Expand   : 새 컬럼을 추가한다 (nullable, 기본값 있음)
                 → v1 은 무시하고, v2 는 사용한다. 둘 다 동작.
2단계 Migrate  : v2 를 전량 배포하고 데이터를 채운다
3단계 Contract : 아무도 안 쓰게 된 뒤 옛 컬럼을 삭제한다 (다음 릴리스)
```

| 하면 안 되는 것 | 대신 |
|---|---|
| 컬럼 DROP + 새 버전 동시 배포 | 배포 완료 후 다음 릴리스에서 DROP |
| 컬럼 RENAME | 새 컬럼 추가 → 이중 쓰기 → 옛 컬럼 제거 |
| NOT NULL 컬럼 추가 | nullable + 기본값으로 추가 후 채우기 |
| 타입 변경 | 새 컬럼 추가 후 이관 |

> **"마이그레이션은 항상 하위 호환"** 이 무중단 배포의 실질적 제약이다.  
> 이걸 지킬 수 없는 마이그레이션이면 **Recreate + 계획된 다운타임**이 정직한 선택이다. 억지로 무중단을 시도하다 데이터를 잃는 것보다 낫다.

```yaml
# ArgoCD PreSync 훅으로 마이그레이션을 배포 전에 → 05-argocd-advanced/
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync
```

---

## 선택 기준

```
DB 스키마가 하위 호환되지 않는다        →  Recreate (계획된 다운타임)
그냥 안전하게 무중단으로 바꾸고 싶다     →  RollingUpdate (기본값)
롤백을 즉시 해야 한다, 리소스 여유 있다  →  Blue-Green
장애 영향을 최소화하며 검증하고 싶다     →  Canary + 자동 분석
어느 버전이 더 나은지 실험하고 싶다      →  A/B 테스트
```

| 상황 | 전략 |
|---|---|
| 내부 도구, 배치 | Recreate |
| 일반적인 웹 서비스 | RollingUpdate |
| 결제·인증 등 고위험 | Blue-Green 또는 Canary |
| 사용자 많고 지표가 잘 갖춰짐 | Canary + AnalysisTemplate |

> **대부분의 서비스에 RollingUpdate로 충분하다.** Canary는 관측성이 먼저 갖춰져 있어야 의미가 있다 — 판단할 지표가 없으면 자동 분석도 없다. → `../observability-lab/`

---

## 배운 점

- 전략 선택은 **다운타임·리소스·롤백 속도·버전 공존** 네 가지의 트레이드오프
- **Canary는 "안전한가", A/B는 "어느 쪽이 나은가"** — 목적이 다르다
- Recreate는 두 버전이 공존하면 안 될 때 (RWO 볼륨, 비호환 스키마)
- RollingUpdate에서 `maxUnavailable: 0` + `maxSurge: 1`이 용량 보존형
- **롤링의 안전성은 전적으로 readinessProbe에 달려 있다** — 껍데기 probe면 깨진 버전이 전량 나간다
- 배포 중 5xx의 1순위 원인은 **엔드포인트 전파와 SIGTERM의 경합** → `preStop` + 우아한 종료
- Blue-Green의 핵심 가치는 **selector 한 줄로 즉시 롤백**
- Blue-Green은 리소스 2배 — 비용이 대가다
- 레플리카 비율 카나리는 간단하지만 **비율이 부정확**하다
- **수동 카나리는 실전에서 안 돌아간다** — 자동 분석·자동 롤백과 결합해야 의미가 있다
- ⚠️ **무중단 배포의 전제는 두 버전이 같은 DB를 쓸 수 있는 것**
- 마이그레이션은 **Expand-Contract** — 추가는 지금, 삭제는 다음 릴리스
- 하위 호환이 불가능한 마이그레이션이면 **계획된 다운타임이 정직한 선택**
- 마이그레이션은 ArgoCD **PreSync 훅**으로 배포 전에 실행
- **대부분은 RollingUpdate로 충분** — Canary는 관측성이 먼저다
