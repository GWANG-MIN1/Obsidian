# 04 Service & 네트워킹

Pod는 일회용이라 죽으면 IP가 바뀐다. Pod IP로 직접 통신하면 안 되는 이유다.  
**Service**는 변하지 않는 안정적 진입점(가상 IP + DNS 이름)을 제공하고, 뒤의 Pod들에 트래픽을 분산한다.

---

## 왜 Service가 필요한가

```
문제: Pod는 언제든 교체 → IP 계속 바뀜 → 어디로 요청?
해결: Service = 고정 이름/IP → 레이블로 Pod 묶음 → 로드밸런싱
```

- Service는 **레이블 셀렉터**로 대상 Pod를 고른다 (컨트롤러와 느슨하게 결합)
- Pod가 교체돼도 Service의 이름·ClusterIP는 그대로
- 여러 Pod에 자동 **로드밸런싱** (kube-proxy가 규칙 관리)

```
                  ┌─────────────┐
   요청 ─────────▶│  Service    │  (고정 IP + DNS: web.default.svc)
                  │  selector:  │
                  │  app=web    │
                  └──────┬──────┘
             ┌───────────┼───────────┐
             ▼           ▼           ▼
        ┌────────┐  ┌────────┐  ┌────────┐
        │ Pod    │  │ Pod    │  │ Pod    │  (app=web 레이블)
        └────────┘  └────────┘  └────────┘
```

---

## Service 타입

| 타입 | 노출 범위 | 용도 |
|------|-----------|------|
| **ClusterIP** | 클러스터 **내부**만 (기본) | 내부 서비스 간 통신 |
| **NodePort** | 각 Node의 고정 포트로 외부 노출 | 개발·테스트 외부 접근 |
| **LoadBalancer** | 클라우드 LB로 외부 노출 | 프로덕션 외부 서비스 |
| **ExternalName** | 외부 DNS로 CNAME 매핑 | 클러스터 밖 서비스 참조 |

### ClusterIP (기본)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web            # 이 레이블 Pod로 트래픽 전달
  ports:
    - port: 80          # Service가 노출하는 포트
      targetPort: 8080  # Pod(컨테이너)의 실제 포트
```

클러스터 안에서 `web` 또는 `web.네임스페이스.svc.cluster.local`로 접근.

### NodePort

```yaml
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # 30000-32767 범위 (생략 시 자동 할당)
```

`<노드IP>:30080`으로 외부에서 접근. 모든 Node의 해당 포트가 열린다.

### LoadBalancer

```yaml
spec:
  type: LoadBalancer   # 클라우드가 외부 LB 프로비저닝 (EKS→ELB)
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

클라우드 환경(EKS·GKE)에서 실제 로드밸런서와 외부 IP가 자동 생성된다. 온프레미스에선 MetalLB 같은 구현이 필요.

```
ClusterIP ⊂ NodePort ⊂ LoadBalancer
(LoadBalancer는 내부적으로 NodePort와 ClusterIP를 포함한다)
```

---

## 포트 3종 구분

| 필드 | 의미 |
|------|------|
| `port` | **Service**가 노출하는 포트 (클러스터 내부에서 접근) |
| `targetPort` | **Pod(컨테이너)** 의 실제 리슨 포트 |
| `nodePort` | **Node**에 열리는 외부 포트 (NodePort/LoadBalancer) |

```
외부 ─(nodePort 30080)→ Node ─(port 80)→ Service ─(targetPort 8080)→ Pod
```

---

## DNS 서비스 디스커버리

클러스터 내부 DNS(CoreDNS)가 Service마다 이름을 자동 등록한다. **IP 대신 이름으로 통신**.

```
<서비스명>.<네임스페이스>.svc.cluster.local

# 예시
web                          # 같은 네임스페이스 → 짧게
web.default                  # 다른 네임스페이스 명시
web.default.svc.cluster.local  # 전체 FQDN
```

```bash
# app Pod 안에서 db Service 호출
curl http://db:5432
curl http://db.database.svc.cluster.local:5432
```

> 애플리케이션 설정에 Pod IP를 하드코딩하지 말고 **Service 이름**을 쓴다. 이것이 마이크로서비스 간 결합을 느슨하게 유지하는 핵심.

---

## Endpoints

Service는 셀렉터에 맞는 Pod들의 IP 목록(**Endpoints**)을 실시간으로 관리한다. 디버깅 시 유용.

```bash
kubectl get endpoints web        # Service가 실제로 가리키는 Pod IP 목록
kubectl get endpointslices       # 최신 방식 (대규모 확장)
```

> **Service가 응답 없으면 Endpoints부터 확인하라.** Endpoints가 비어 있으면 셀렉터가 Pod 레이블과 안 맞거나, Pod가 readiness 실패로 제외된 것이다.

---

## Headless Service

`clusterIP: None`으로 지정하면 로드밸런싱 없이 **각 Pod의 DNS를 직접** 제공한다. StatefulSet과 함께 쓴다.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None       # headless
  selector:
    app: db
  ports:
    - port: 5432
```

→ `db-0.db-headless`, `db-1.db-headless`처럼 **개별 Pod에 안정적 DNS**로 접근 (DB 리더/팔로워 구분 등).

---

## 배운 점

- Pod IP는 불안정 → **Service가 고정 진입점 + 로드밸런싱** 제공
- 타입 4종: ClusterIP(내부)·NodePort(노드포트)·LoadBalancer(클라우드LB)·ExternalName
- `port`(서비스)/`targetPort`(파드)/`nodePort`(노드) 3종을 구분
- 통신은 IP가 아니라 **Service DNS 이름**으로 → 느슨한 결합
- Service가 안 되면 **Endpoints**부터 확인 (셀렉터/레이블 불일치 진단)
- StatefulSet엔 개별 Pod DNS를 주는 **headless Service**
