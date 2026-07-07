# 07 Ingress

Service의 `LoadBalancer` 타입은 서비스마다 로드밸런서를 하나씩 만든다 → 비용·관리 부담.  
**Ingress**는 **하나의 진입점**에서 호스트·경로 규칙으로 여러 서비스에 트래픽을 라우팅하는 L7(HTTP/HTTPS) 게이트웨이다.

---

## 왜 Ingress인가

```
LoadBalancer만 쓰면              Ingress를 쓰면
┌────┐┌────┐┌────┐              ┌──────────────┐
│LB 1││LB 2││LB 3│              │  Ingress LB  │  (하나)
└─┬──┘└─┬──┘└─┬──┘              └──────┬───────┘
  ▼     ▼     ▼             경로/호스트 규칙으로 분기
 svc1  svc2  svc3            /api→svc1  /web→svc2  shop.x→svc3
(LB 3개 = 비용 3배)          (LB 1개 + L7 라우팅)
```

- **경로 기반 라우팅**: `/api` → api 서비스, `/web` → web 서비스
- **호스트 기반 라우팅**: `shop.example.com` → shop, `blog.example.com` → blog
- **TLS 종료**: HTTPS 인증서를 Ingress에서 한 곳에 관리
- LB 하나로 다수 서비스 노출 → 비용 절감

---

## Ingress = 규칙 + Controller

Ingress는 **두 부분**으로 나뉜다. 이 구분이 핵심.

| 구성 | 역할 |
|------|------|
| **Ingress 리소스** | 라우팅 **규칙** (YAML). "이 경로는 저 서비스로" |
| **Ingress Controller** | 규칙을 실제로 **실행하는 프록시** (nginx, Traefik, AWS ALB 등) |

> **Ingress 리소스만 만들면 아무 일도 안 일어난다.** 규칙을 읽고 트래픽을 처리할 Ingress Controller가 클러스터에 설치돼 있어야 한다. 이걸 빼먹는 게 가장 흔한 실수.

```bash
# minikube에서 nginx ingress controller 활성화
minikube addons enable ingress

# 또는 Helm으로 설치
helm install ingress-nginx ingress-nginx/ingress-nginx
```

| Controller | 특징 |
|------------|------|
| **ingress-nginx** | 가장 대중적, 온프레미스·클라우드 공통 |
| **Traefik** | 자동 설정·대시보드 |
| **AWS ALB Ingress** | EKS에서 ALB 자동 생성 (aws-load-balancer-controller) |

---

## 경로 기반 라우팅

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /web
            pathType: Prefix
            backend:
              service:
                name: web-svc
                port:
                  number: 80
```

| pathType | 매칭 |
|----------|------|
| `Prefix` | 경로 접두사 매칭 (`/api` → `/api/users` 포함) |
| `Exact` | 정확히 일치 |
| `ImplementationSpecific` | 컨트롤러 구현에 위임 |

---

## 호스트 기반 라우팅 (가상 호스트)

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
    - host: blog.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: blog-svc
                port:
                  number: 80
```

같은 IP·LB로 들어와도 `Host` 헤더에 따라 다른 서비스로 분기.

---

## TLS 종료 (HTTPS)

인증서를 Secret(`kubernetes.io/tls`)에 저장하고 Ingress에 연결하면, Ingress가 HTTPS를 복호화(종료)해 백엔드엔 HTTP로 전달한다.

```bash
kubectl create secret tls shop-tls --cert=tls.crt --key=tls.key
```

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - shop.example.com
      secretName: shop-tls        # TLS 인증서 Secret
  rules:
    - host: shop.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop-svc
                port:
                  number: 80
```

> 실무에선 **cert-manager**를 설치해 Let's Encrypt 인증서를 자동 발급·갱신한다. 인증서 만료로 인한 장애를 막는 사실상 표준 조합(cert-manager + ingress-nginx).

---

## 요청 흐름 정리

```
사용자
  │  https://shop.example.com/checkout
  ▼
Ingress Controller (LB)  ── TLS 복호화, Host/Path 규칙 매칭
  ▼
Service (shop-svc, ClusterIP)  ── 로드밸런싱
  ▼
Pod (shop)
```

Ingress는 Service를 **대체하지 않는다**. Ingress → Service → Pod로 이어지며, 각 계층의 역할이 분리돼 있다.

---

## Ingress vs Gateway API

Ingress는 HTTP/HTTPS 위주로 기능이 제한적이라, 최신 표준으로 **Gateway API**가 등장했다(더 유연한 라우팅·역할 분리·L4 지원). 신규 프로젝트라면 Gateway API도 함께 검토할 가치가 있다. 다만 현재도 Ingress가 가장 널리 쓰인다.

---

## 배운 점

- Ingress = **하나의 L7 진입점**에서 호스트·경로로 다수 서비스 라우팅 (LB 비용 절감)
- **규칙(Ingress 리소스) + 실행자(Ingress Controller)** 는 별개 → Controller 설치 필수
- Prefix/Exact 경로 매칭, `host`로 가상 호스트 분기
- TLS는 Ingress에서 종료, **cert-manager로 자동 발급·갱신**이 실무 표준
- 흐름: Ingress → Service → Pod (Service를 대체하지 않음)
