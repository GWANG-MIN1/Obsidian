# 02 Pod

Pod는 Kubernetes에서 **배포 가능한 가장 작은 단위**다.  
컨테이너를 직접 배포하지 않고, 컨테이너를 감싼 Pod를 배포한다. Pod 안에는 1개 이상의 컨테이너가 들어간다.

---

## Pod란

- **1개 이상의 컨테이너 + 공유 리소스**의 묶음
- 같은 Pod 안 컨테이너들은 **네트워크(같은 IP·포트 공간)** 와 **스토리지(볼륨)** 를 공유
- `localhost`로 서로 통신, 같은 볼륨을 마운트 가능
- Pod는 **일회용(ephemeral)** — 죽으면 되살아나지 않고, 컨트롤러가 **새 Pod로 교체**한다(IP도 바뀜)

```
┌─────────── Pod (IP: 10.1.0.5) ───────────┐
│  ┌────────────┐      ┌────────────┐      │
│  │ 컨테이너 A │      │ 컨테이너 B │      │
│  └─────┬──────┘      └─────┬──────┘      │
│        └── localhost 통신 ──┘             │
│        ┌──────────────────┐              │
│        │  공유 볼륨        │              │
│        └──────────────────┘              │
└──────────────────────────────────────────┘
```

> 대부분의 경우 **Pod 1개 = 컨테이너 1개**. 여러 컨테이너는 "보조 프로세스가 메인과 생명주기를 함께해야 할 때"만 쓴다(사이드카 패턴).

---

## Pod 매니페스트 기본 구조

모든 Kubernetes 오브젝트는 4개 최상위 필드를 갖는다.

```yaml
apiVersion: v1          # 오브젝트의 API 그룹/버전
kind: Pod               # 오브젝트 종류
metadata:               # 이름·레이블·네임스페이스 등 식별 정보
  name: nginx-pod
  labels:
    app: nginx
spec:                   # 원하는 상태 (핵심)
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:       # 최소 보장 자원
          cpu: "100m"
          memory: "128Mi"
        limits:         # 최대 상한
          cpu: "500m"
          memory: "256Mi"
```

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod nginx-pod     # 이벤트·상태 확인
kubectl logs nginx-pod
kubectl exec -it nginx-pod -- bash
kubectl delete -f pod.yaml
```

| 필드 | 의미 |
|------|------|
| `apiVersion` | 오브젝트가 속한 API 버전 (Pod은 `v1`) |
| `kind` | 오브젝트 종류 (Pod, Deployment, Service…) |
| `metadata` | 이름·레이블·어노테이션 등 식별 정보 |
| `spec` | **원하는 상태** — 무엇을 실행할지 |
| `status` | 현재 상태 (시스템이 채움, 직접 작성 X) |

---

## Pod 라이프사이클

```
Pending → Running → Succeeded / Failed
   │         │
   │         └─ 컨테이너가 정상 실행 중
   └─ 스케줄링·이미지 pull 대기
```

| Phase | 의미 |
|-------|------|
| `Pending` | 스케줄 대기 또는 이미지 다운로드 중 |
| `Running` | Node에 배치되어 컨테이너가 실행 중 |
| `Succeeded` | 모든 컨테이너가 정상 종료(exit 0) |
| `Failed` | 하나 이상 컨테이너가 비정상 종료 |
| `CrashLoopBackOff` | 반복 크래시 → 재시작 간격을 점점 늘리는 상태(디버깅 신호) |

### restartPolicy

```yaml
spec:
  restartPolicy: Always   # 기본값. OnFailure / Never 선택 가능
```

| 정책 | 용도 |
|------|------|
| `Always` | 항상 재시작 (기본, 상시 서비스용) |
| `OnFailure` | 비정상 종료 시에만 (Job에 적합) |
| `Never` | 재시작 안 함 |

---

## 멀티 컨테이너 패턴

같은 Pod 안 컨테이너는 네트워크·볼륨을 공유한다. 이를 이용한 대표 패턴.

| 패턴 | 설명 |
|------|------|
| **Sidecar** | 메인 컨테이너 보조 (로그 수집기, 프록시 등) |
| **Ambassador** | 외부 연결을 대리하는 프록시 |
| **Adapter** | 출력 포맷을 표준화 (모니터링 규격 맞추기) |

```yaml
spec:
  containers:
    - name: app
      image: myapp
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: log-shipper          # 사이드카: 로그를 읽어 전송
      image: fluent/fluent-bit
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

---

## Init Container

메인 컨테이너보다 **먼저 순차 실행**되고, 완료되어야 메인이 시작된다. 초기화·의존성 대기에 사용.

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z db 5432; do sleep 2; done']
  containers:
    - name: app
      image: myapp
```

---

## Health Probe (상태 점검)

kubelet이 컨테이너 건강을 주기적으로 검사한다. Docker의 healthcheck보다 세분화되어 있다.

| Probe | 실패 시 동작 | 용도 |
|-------|-------------|------|
| **livenessProbe** | 컨테이너 **재시작** | 데드락·행 감지 → 살아있나? |
| **readinessProbe** | Service에서 **트래픽 제외** | 준비됐나? (아직 초기화 중이면 대기) |
| **startupProbe** | 시작 유예 (느린 앱 보호) | 느린 시작 앱이 완전히 뜰 때까지 |

```yaml
spec:
  containers:
    - name: app
      image: myapp
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
```

프로브 방식: `httpGet`(HTTP 200~399), `tcpSocket`(포트 연결), `exec`(명령 exit 0).

> **readiness와 liveness를 구분하라.** readiness 실패 = 아직 트래픽 받지 마(하지만 살아있음). liveness 실패 = 망가졌으니 재시작. 이 둘을 혼동하면 초기화 중인 앱을 계속 죽이는 재시작 루프에 빠진다.

---

## 배운 점

- Pod = 최소 배포 단위, 컨테이너 + 공유 네트워크/볼륨
- Pod는 **일회용** — 직접 관리하지 말고 컨트롤러(Deployment)로 관리한다(다음 장)
- 매니페스트 4대 필드: `apiVersion / kind / metadata / spec`
- 멀티 컨테이너는 사이드카처럼 **생명주기를 함께할 때만**
- **liveness(재시작) vs readiness(트래픽 제외)** 구분이 안정 운영의 핵심
- `kubectl describe` + `kubectl logs`가 Pod 디버깅의 두 축
