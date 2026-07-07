# 03 워크로드 컨트롤러

Pod를 직접 만들면 죽었을 때 아무도 되살리지 않는다.  
**컨트롤러**는 "원하는 상태"를 선언받아 Pod를 자동으로 생성·복구·확장한다. 셀프힐링과 스케일링의 실체.

---

## 컨트롤러 계층

```
Deployment ──manages──▶ ReplicaSet ──manages──▶ Pod × N
   (버전·롤아웃)           (복제 수 유지)          (실행 단위)
```

| 컨트롤러 | 용도 |
|----------|------|
| **ReplicaSet** | Pod를 지정 개수만큼 유지 (보통 직접 안 씀) |
| **Deployment** | ReplicaSet을 관리 + 롤링 업데이트·롤백 (무상태 앱 표준) |
| **StatefulSet** | 상태·고유 ID가 필요한 앱 (DB, 큐) |
| **DaemonSet** | 모든 Node에 Pod 하나씩 (로그·모니터링 에이전트) |
| **Job** | 완료가 목표인 일회성 작업 (배치) |
| **CronJob** | 스케줄에 따라 Job 반복 실행 |

---

## ReplicaSet

Pod를 지정한 복제 수(replicas)로 유지한다. **레이블 셀렉터**로 자신이 관리할 Pod를 식별한다.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web          # 이 레이블을 가진 Pod를 관리
  template:             # 만들 Pod의 틀 (= Pod spec)
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

> 실무에서 ReplicaSet을 직접 만드는 일은 드물다. **Deployment가 ReplicaSet을 대신 관리**해주기 때문. 하지만 셀렉터·template 구조는 모든 컨트롤러의 공통 뼈대다.

---

## Deployment (무상태 앱의 표준)

가장 많이 쓰는 컨트롤러. ReplicaSet 위에서 **버전 관리·무중단 배포·롤백**을 제공한다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # 목표보다 최대 몇 개 더 띄울지
      maxUnavailable: 0   # 동시에 최대 몇 개 내릴지 (0 = 무중단)
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

### 롤링 업데이트 & 롤백

```bash
kubectl apply -f deployment.yaml
kubectl set image deployment/web nginx=nginx:1.28   # 이미지 교체 → 롤링 업데이트
kubectl rollout status deployment/web               # 진행 상태
kubectl rollout history deployment/web              # 리비전 이력
kubectl rollout undo deployment/web                 # 직전 버전으로 롤백
kubectl rollout undo deployment/web --to-revision=2 # 특정 리비전으로
kubectl scale deployment web --replicas=5           # 스케일
kubectl rollout restart deployment/web              # 이미지 그대로 재시작 (설정 반영)
```

**롤링 업데이트 원리**: 새 ReplicaSet의 Pod를 하나씩 늘리고, 준비되면(readiness) 옛 ReplicaSet의 Pod를 하나씩 줄인다. `maxSurge`/`maxUnavailable`로 속도·무중단 정도를 조절.

| 배포 전략 | 설명 |
|-----------|------|
| **RollingUpdate** | 점진 교체 (기본, 무중단) |
| **Recreate** | 전부 내리고 새로 올림 (짧은 다운타임, 호환성 문제 회피) |

---

## StatefulSet (상태 있는 앱)

DB·메시지 큐처럼 **고유 identity·안정적 스토리지·순서**가 필요한 앱용.

| Deployment | StatefulSet |
|------------|-------------|
| Pod 이름 랜덤 (`web-a1b2c3`) | 순번 고정 (`db-0`, `db-1`, `db-2`) |
| Pod 교체 가능·동등 | 각 Pod가 고유 identity 유지 |
| 볼륨 공유/무상태 | Pod마다 전용 PVC (`volumeClaimTemplates`) |
| 순서 무관 | 순차 생성·역순 삭제 |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless    # headless Service 필요 (안정 DNS)
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:       # Pod마다 개별 PVC 자동 생성
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

---

## DaemonSet (노드마다 하나)

**모든(또는 특정) Node에 Pod를 정확히 하나씩** 배치한다. 노드가 추가되면 자동으로 그 노드에도 배치.

- 로그 수집기 (Fluent Bit), 모니터링 (node-exporter), 네트워크 플러그인, 스토리지 데몬

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter
```

---

## Job & CronJob

**완료가 목표**인 작업. 상시 실행이 아니라 끝나면 종료된다.

### Job — 일회성 배치

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate
spec:
  completions: 1        # 성공해야 할 Pod 수
  parallelism: 1        # 동시 실행 수
  backoffLimit: 4       # 실패 시 재시도 한도
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myapp:migrate
          command: ["python", "manage.py", "migrate"]
```

### CronJob — 스케줄 반복

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup
spec:
  schedule: "0 2 * * *"          # 매일 02:00 (cron 표현식)
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: myapp:backup
              command: ["/backup.sh"]
```

```
┌ 분 (0-59)
│ ┌ 시 (0-23)
│ │ ┌ 일 (1-31)
│ │ │ ┌ 월 (1-12)
│ │ │ │ ┌ 요일 (0-6, 0=일)
│ │ │ │ │
0 2 * * *   → 매일 새벽 2시
```

---

## 컨트롤러 선택 가이드

| 상황 | 컨트롤러 |
|------|----------|
| 일반 무상태 웹/API | **Deployment** |
| DB, 메시지 큐 등 상태·순서 필요 | **StatefulSet** |
| 노드마다 에이전트 하나 | **DaemonSet** |
| 한 번 실행하고 끝 (마이그레이션·배치) | **Job** |
| 정기 실행 (백업·리포트) | **CronJob** |

---

## 배운 점

- Pod는 직접 만들지 말고 **컨트롤러로 관리** → 셀프힐링·스케일링
- Deployment → ReplicaSet → Pod 계층, 셀렉터·template이 공통 뼈대
- **RollingUpdate**로 무중단 배포, `rollout undo`로 즉시 롤백
- 상태 있는 앱은 StatefulSet(고유 ID·전용 PVC·headless Service)
- 노드 단위 에이전트는 DaemonSet, 일회성/정기 작업은 Job/CronJob
