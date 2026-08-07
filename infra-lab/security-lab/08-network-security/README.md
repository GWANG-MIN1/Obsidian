# 08 네트워크 보안

Kubernetes의 기본 네트워크 모델은 **"모든 파드가 모든 파드와 통신할 수 있다"** 이다.  
즉 파드 하나가 뚫리면 **클러스터 안의 모든 것에 도달할 수 있다.** NetworkPolicy는 그 횡이동(lateral movement)을 끊는 도구다.

---

## 기본값이 위험한 이유

```
[ NetworkPolicy 없음 ]
frontend ──▶ backend ──▶ database
   │                        ▲
   └────────────────────────┘   ← 프론트가 DB에 직접 접근 가능
   └──▶ 다른 팀 네임스페이스의 모든 파드
   └──▶ 인터넷 (이그레스 제한 없음)
```

> **"방화벽이 없는 평평한 네트워크"** 가 클러스터 기본 상태다. VPC 보안그룹은 노드 단위라 파드 간 통신을 제어하지 못한다.  
> 공격자가 취약한 프론트엔드 파드 하나를 잡으면, 거기서 DB·시크릿 저장소·메타데이터 서비스까지 전부 스캔할 수 있다.

---

## NetworkPolicy 구조

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: myapp
spec:
  podSelector:                    # 이 정책이 적용될 파드
    matchLabels:
      app: backend
  policyTypes: [Ingress, Egress]  # 어느 방향을 제어할지

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend       # frontend 파드에서만
      ports:
        - protocol: TCP
          port: 8080

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

### 세 가지 셀렉터

```yaml
from:
  - podSelector:          # 같은 네임스페이스의 파드
      matchLabels: { app: frontend }

  - namespaceSelector:    # 특정 네임스페이스의 모든 파드
      matchLabels: { kubernetes.io/metadata.name: monitoring }

  - namespaceSelector:    # 특정 네임스페이스의 특정 파드 (AND)
      matchLabels: { name: other-team }
    podSelector:
      matchLabels: { app: client }

  - ipBlock:              # CIDR
      cidr: 10.0.0.0/16
      except: [10.0.5.0/24]
```

> ⚠️ **YAML 들여쓰기가 의미를 바꾼다.** 위 세 번째처럼 `namespaceSelector`와 `podSelector`가 **같은 리스트 항목 안에 있으면 AND**, 별개 항목(`-`)으로 나뉘면 **OR**다. 가장 흔한 실수다.

```yaml
# AND — other-team 네임스페이스의 client 파드만
- namespaceSelector: {...}
  podSelector: {...}

# OR — other-team 의 모든 파드 또는 아무 네임스페이스의 client 파드
- namespaceSelector: {...}
- podSelector: {...}
```

---

## Default Deny — 시작점

```yaml
# 네임스페이스 안의 모든 파드에 대해 모든 인그레스·이그레스 차단
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: myapp
spec:
  podSelector: {}                 # 모든 파드
  policyTypes: [Ingress, Egress]
  # ingress/egress 규칙 없음 = 전부 차단
```

```
NetworkPolicy 의 동작 원리
  ① 어떤 정책도 파드를 선택하지 않으면 → 전부 허용 (기본)
  ② 하나라도 선택하면 → 그 정책들이 허용한 것만 통과 (화이트리스트)
  ③ 여러 정책은 OR 로 합쳐진다 (거부 규칙은 없다)
```

> **NetworkPolicy에는 "거부" 규칙이 없다.** 정책은 오직 허용만 표현하고, 정책에 선택된 파드는 명시되지 않은 통신이 자동으로 막힌다.  
> 그래서 **default-deny를 먼저 깔고 필요한 것만 열어가는** 순서가 유일하게 안전한 방법이다.

### ⚠️ DNS를 반드시 열어야 한다

```yaml
# default-deny 후 가장 먼저 추가할 것
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: myapp
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

> **이그레스 default-deny를 걸면 DNS부터 죽는다.** 앱이 "연결 안 됨"이 아니라 **"호스트를 찾을 수 없음"** 으로 실패하는데, 원인이 NetworkPolicy라는 걸 알아채기 어렵다.  
> 이그레스 정책 도입 시 **DNS 허용을 세트로 함께** 넣는다.

---

## Kyverno로 자동 부착

새 네임스페이스에 default-deny를 자동으로 깔 수 있다.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-networkpolicy
spec:
  rules:
    - name: default-deny
      match:
        any:
          - resources:
              kinds: [Namespace]
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny
        namespace: "{{request.object.metadata.name}}"
        synchronize: true
        data:
          spec:
            podSelector: {}
            policyTypes: [Ingress, Egress]
```

> **안전한 기본값을 조직 차원에서 보장하는 방법이다.** 개발자가 잊어도 깔린다. → `04-kyverno/`

---

## ⚠️ CNI가 지원해야 동작한다

```
NetworkPolicy 는 API 오브젝트일 뿐, 실제 강제는 CNI 플러그인이 한다.
지원하지 않는 CNI 에서는 정책을 만들어도 조용히 무시된다.  💥
```

| CNI | NetworkPolicy | 비고 |
|---|---|---|
| **AWS VPC CNI** | ✅ (v1.14+) | **`ENABLE_NETWORK_POLICY=true` 필요** |
| Calico | ✅ | 확장 정책(GlobalNetworkPolicy)도 제공 |
| Cilium | ✅ | eBPF, L7 정책까지 |
| Flannel | ❌ | 지원 안 함 |

```bash
# EKS 기본 VPC CNI 에서 활성화 확인
kubectl -n kube-system get ds aws-node -o yaml | grep -A2 ENABLE_NETWORK_POLICY

# 정책이 실제로 적용되는지 테스트
kubectl -n myapp run test --rm -it --image=busybox --restart=Never -- \
  wget -qO- --timeout=3 http://backend:8080
```

> **"정책을 만들었는데 여전히 통신된다"의 1순위 원인이 CNI 미지원·미활성화다.** 정책을 믿기 전에 **실제로 차단되는지 테스트**한다.  
> 이건 Kyverno 정책의 대조 실험과 같은 발상이다 — 통과 사례만 보면 아무것도 검증되지 않는다. → `04-kyverno/`

---

## 이그레스 제어 — 데이터 유출 방지

인그레스보다 잊기 쉽지만, **유출을 막는 건 이그레스다.**

```yaml
egress:
  # 클러스터 내부 DB 만
  - to:
      - podSelector:
          matchLabels: { app: database }
    ports: [{ protocol: TCP, port: 5432 }]

  # DNS
  - to:
      - namespaceSelector:
          matchLabels: { kubernetes.io/metadata.name: kube-system }
        podSelector:
          matchLabels: { k8s-app: kube-dns }
    ports: [{ protocol: UDP, port: 53 }]

  # 외부는 특정 대역만
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0
          except:
            - 169.254.169.254/32     # ⭐ IMDS 차단
            - 10.0.0.0/8             # VPC 내부 직접 접근 차단
    ports: [{ protocol: TCP, port: 443 }]
```

### ⭐ IMDS 차단

```
169.254.169.254 = EC2 인스턴스 메타데이터 서비스
파드에서 여기 접근 가능하면 → 노드 IAM 역할의 자격증명을 그대로 얻는다
      ↓
IRSA 로 파드 권한을 최소화한 의미가 사라진다
```

| 대응 | 방법 |
|---|---|
| **IMDSv2 강제** | 토큰 기반 요청만 허용 (SSRF 방어) |
| **hop limit = 1** | 컨테이너에서의 요청이 한 홉을 더 거쳐 차단된다 |
| NetworkPolicy | `169.254.169.254/32`를 except에 |

```hcl
# Terraform — 노드 그룹 메타데이터 옵션
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required"   # IMDSv2 강제
  http_put_response_hop_limit = 1            # 컨테이너에서 접근 차단
}
```

> **hop limit 1이 가장 효과적이다.** 컨테이너 네트워크 네임스페이스에서 나가는 요청은 홉을 하나 더 소비하므로 TTL이 떨어져 도달하지 못한다.  
> 이게 `01-security-basics/`의 위협 모델에서 **"가장 자주 실현되는 경로"** 로 꼽은 그것이다.

---

## 서비스 메시와 mTLS

NetworkPolicy는 **L3/L4**(IP·포트)만 다룬다. 그 이상이 필요하면 서비스 메시다.

| | NetworkPolicy | 서비스 메시 (Istio·Linkerd) |
|---|---|---|
| 계층 | L3/L4 (IP·포트) | **L7 (HTTP 메서드·경로·헤더)** |
| 암호화 | ❌ | ✅ **mTLS 자동** |
| 신원 | 라벨 기반 | **인증서 기반 워크로드 신원** |
| 관측성 | 없음 | 요청 단위 메트릭·트레이스 |
| 비용 | 없음 (CNI 내장) | 사이드카·복잡도·리소스 |

```yaml
# Istio — mTLS 강제
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: myapp
spec:
  mtls:
    mode: STRICT
```

```yaml
# L7 인가 — NetworkPolicy 로는 불가능한 수준
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: backend-policy
  namespace: myapp
spec:
  selector:
    matchLabels: { app: backend }
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/myapp/sa/frontend"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/*"]
```

> **서비스 메시는 공짜가 아니다.** 사이드카가 파드마다 붙어 리소스·지연·운영 복잡도를 늘린다.  
> **NetworkPolicy로 시작해서, L7 인가나 mTLS가 실제로 필요할 때 도입한다.** "보안 강화"라는 막연한 이유로 먼저 깔면 운영 부담만 는다.  
> Cilium은 eBPF로 사이드카 없이 L7 정책 일부를 제공한다 — 중간 선택지다.

---

## 도입 순서

```
1. CNI 가 NetworkPolicy 를 지원하는지 확인·활성화
2. 통신 흐름을 파악한다 (관측성·로그)
3. 인그레스부터 — 네임스페이스별 default-deny + 필요한 것만 허용
4. 실제로 차단되는지 테스트
5. 이그레스 — DNS 허용을 세트로, IMDS 차단
6. 필요하면 mTLS·L7
```

```bash
# 현재 정책 현황
kubectl get networkpolicy -A

# 정책이 없는 네임스페이스 찾기
kubectl get ns -o name | while read ns; do
  n="${ns#namespace/}"
  c=$(kubectl -n "$n" get networkpolicy --no-headers 2>/dev/null | wc -l)
  [ "$c" -eq 0 ] && echo "정책 없음: $n"
done
```

---

## 배운 점

- Kubernetes 기본은 **"모든 파드가 모든 파드와 통신 가능"** — 평평한 네트워크
- VPC 보안그룹은 노드 단위라 **파드 간 통신을 제어하지 못한다**
- **NetworkPolicy에는 거부 규칙이 없다** — 허용만 표현하고, 선택된 파드는 나머지가 자동 차단
- 그래서 **default-deny 먼저, 필요한 것만 열기**가 유일하게 안전한 순서
- ⚠️ **`namespaceSelector`와 `podSelector`가 같은 항목이면 AND, 별개 항목이면 OR** — 가장 흔한 실수
- ⚠️ **이그레스 default-deny를 걸면 DNS부터 죽는다** — DNS 허용을 세트로 넣는다
- 실패가 "연결 안 됨"이 아니라 "호스트를 찾을 수 없음"으로 나와 원인 파악이 어렵다
- Kyverno `generate`로 새 네임스페이스에 **default-deny를 자동 부착**할 수 있다
- ⚠️ **CNI가 지원해야 동작한다** — Flannel은 미지원, AWS VPC CNI는 `ENABLE_NETWORK_POLICY=true`
- **"정책을 만들었는데 통신된다"의 1순위는 CNI 미활성화** — 반드시 차단 테스트를 한다
- 유출을 막는 건 인그레스가 아니라 **이그레스**
- ⭐ **IMDS(169.254.169.254) 차단이 가장 중요한 이그레스 규칙** — 노드 IAM 탈취 경로
- **IMDSv2 + hop limit 1**이 가장 효과적인 대응 (Terraform `metadata_options`)
- NetworkPolicy는 L3/L4 — **L7 인가·mTLS는 서비스 메시** 영역
- **서비스 메시는 공짜가 아니다** — 실제로 필요할 때 도입한다
