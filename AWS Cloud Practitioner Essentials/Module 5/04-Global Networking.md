# AWS 글로벌 네트워크 서비스

## 1. Amazon Route 53

### 역할

- **DNS(Domain Name System) 서비스**
- 사람이 읽는 **도메인 이름**을 **IP 주소**로 변환

> 예)
> 
> `www.google.com` → `142.xxx.xxx.xxx`

### 주요 기능

- **지연 시간 기반 라우팅 (Latency-Based Routing)**
    - 사용자와 가장 가까운 서버로 연결
- **지리적 위치 기반 라우팅 (Geolocation Routing)**
    - 사용자 지역에 따라 다른 서버로 연결
- **가중치 기반 라우팅 (Weighted Routing)**
    - 트래픽을 원하는 비율로 분산

### 특징

- 높은 가용성(High Availability)
- 높은 확장성(Scalability)

### 한 줄 요약

> **도메인 이름을 IP 주소로 변환하고, 가장 적절한 서버로 연결하는 DNS 서비스**

---

# 2. Amazon CloudFront

### 역할

- **CDN(Content Delivery Network) 서비스**
- 사용자와 가까운 엣지 로케이션(Edge Location)에서 콘텐츠 제공

### 사용 사례

- 스트리밍 동영상
- 전자상거래(E-Commerce) 웹사이트
- 모바일 애플리케이션
- 이미지, CSS, JavaScript 등 정적 콘텐츠 제공

### 특징

- 콘텐츠 전송 속도 향상
- 지연 시간(Latency) 감소
- 사용자 경험(UX) 향상

### 한 줄 요약

> **가까운 서버에서 콘텐츠를 제공하여 빠르게 전달하는 CDN 서비스**

---

# 3. AWS Global Accelerator

### 역할

- AWS 글로벌 네트워크를 이용해 애플리케이션으로 트래픽 전달

### 특징

- 애플리케이션 가용성 향상
- 네트워크 성능 향상
- 장애 발생 시 빠른 우회(Failover)
- 보안 강화

### 한 줄 요약

> **AWS의 글로벌 백본 네트워크를 이용해 애플리케이션 연결을 최적화하는 서비스**


# 서비스 비교
|서비스|역할|핵심 기능|키워드|
|---|---|---|---|
|**Amazon Route 53**|DNS|도메인 → IP 변환, 다양한 라우팅 정책|DNS, 라우팅|
|**Amazon CloudFront**|CDN|사용자와 가까운 위치에서 콘텐츠 제공|캐싱, 저지연|
|**AWS Global Accelerator**|글로벌 네트워크|최적의 네트워크 경로로 트래픽 전달|저지연, 고가용성|