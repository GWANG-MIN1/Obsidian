# 네트워크 및 애플리케이션 보호 (Network & Application Protection)

## 서비스 거부 공격 (DoS)

**DoS (Denial of Service)** 공격은 **단일 공격자(단일 소스)** 가 대상 서버에 과도한 요청을 보내 서비스를 사용할 수 없게 만드는 공격입니다.

### 목적

정상 사용자가 서비스를 이용하지 못하도록 만드는 것

---

## 분산 서비스 거부 공격 (DDoS)

**DDoS (Distributed Denial of Service)** 공격은

여러 대의 손상된 컴퓨터(봇넷, Botnet)를 이용하여 동시에 공격하는 방식입니다.

단일 컴퓨터가 아니라 수천~수백만 대의 시스템이 동시에 요청을 보내기 때문에
DoS보다 훨씬 강력합니다.

---

## DDoS 공격 과정

```
공격자

     ↓

수천~수백만 대의 좀비 PC(Botnet)

     ↓

동시에 요청 전송

     ↓

대상 서버

     ↓

과부하 발생

     ↓

정상 사용자 서비스 불가
```

### 결과

- 서버 과부하
- 네트워크 대역폭 소모
- 응답 지연
- 서비스 중단

---

# UDP Flood 공격

UDP Flood는 대표적인 **네트워크 계층 DDoS 공격**입니다.

공격자는 대량의 UDP 패킷을 보내 서버의 자원을 모두 소모시킵니다.

---

## 비유

기상청에

> "오늘 날씨 알려주세요."

라고 요청하면,

정상적으로는 요청한 사람에게만 날씨를 알려줍니다.

하지만 공격자는

- 수많은 좀비 컴퓨터(Bot)를 이용하고,
- 응답 주소(Return Address)를 **공격 대상 서버**로 조작(Spoofing)합니다.

그러면 기상청은

모든 응답을 **피해 서버**로 보내게 되고,

결국 피해 서버는 엄청난 트래픽을 처리하지 못해 다운됩니다.

---

# AWS에서 UDP Flood 대응

## 1. Security Group

Security Group은

허용된 트래픽만 EC2 인스턴스로 전달하는 **가상 방화벽(Virtual Firewall)** 입니다.

허용되지 않은 요청은 차단합니다.

### 특징

- 상태 저장(Stateful)
- 허용(Allow) 규칙만 설정
- EC2에 도달하기 전에 필터링

---

## 2. Elastic Load Balancing (ELB)

AWS에서는 EC2 인스턴스를 직접 인터넷에 노출하기보다

**Elastic Load Balancer(ELB)** 를 앞단에 배치하는 것을 권장합니다.

```
사용자

      ↓

Elastic Load Balancer

      ↓

EC2 여러 대
```

### 장점

- 트래픽 분산
- 특정 서버 과부하 방지
- DDoS 공격 완화
- 고가용성(High Availability)

---

# AWS Shield Standard

AWS Shield Standard는

AWS에서 **기본적으로 무료 제공하는 DDoS 방어 서비스**입니다.

추가 비용 없이 자동 적용됩니다.

### 보호 대상

- Elastic Load Balancing (ELB)
- Amazon CloudFront
- Amazon Route 53
- AWS Global Accelerator

### 특징

- 자동 DDoS 탐지
- Layer 3 / Layer 4 공격 방어
- 별도 설정 없이 기본 활성화

---

# AWS WAF (Web Application Firewall)

AWS WAF는

웹 애플리케이션으로 들어오는 HTTP/HTTPS 요청을 검사하여

악성 요청을 차단하는 서비스입니다.

예시

- SQL Injection
- Cross Site Scripting(XSS)
- 악성 Bot
- 특정 국가 차단
- IP 차단

---

## AWS Shield + AWS WAF

AWS Shield와 AWS WAF를 함께 사용하면 더욱 강력한 보안 환경을 구축할 수 있습니다.

### 장점

- 머신러닝 기반 위협 탐지
- 최신 공격 패턴 대응
- 웹 애플리케이션 보호
- 공격 자동 차단

---

# AWS Shield Advanced

AWS Shield Advanced는

유료 DDoS 보호 서비스입니다.

Shield Standard보다 더 강력한 기능을 제공합니다.

### 추가 기능

- 대규모 DDoS 공격 방어
- 상세 공격 분석
- 실시간 탐지
- 사전 예방(Proactive Protection)
- AWS DDoS Response Team(DRT) 지원

기업이나 대규모 서비스에서 주로 사용합니다.

---

# AWS 인프라 기반 보호

AWS는 자체 인프라를 활용하여 DDoS 공격을 완화합니다.

대표적인 보호 요소

- Security Group
- Elastic Load Balancing (ELB)
- AWS Shield Standard
- AWS 리전(Region) 및 글로벌 네트워크
- Amazon CloudFront
- AWS WAF

이러한 서비스들이 함께 동작하여 대규모 공격에도 안정적인 서비스 운영을 지원합니다.

---

# 핵심 정리

- **DoS**는 단일 소스에서 발생하는 서비스 거부 공격이다.
- **DDoS**는 여러 대의 손상된 컴퓨터(봇넷)를 이용한 분산 서비스 거부 공격이다.
- **UDP Flood**는 대량의 UDP 패킷을 보내 서버를 과부하시키는 네트워크 공격이다.
- **Security Group**은 허용된 네트워크 트래픽만 EC2에 전달하는 가상 방화벽이다.
- **Elastic Load Balancing(ELB)** 은 트래픽을 여러 서버로 분산하여 공격 영향을 줄인다.
- **AWS Shield Standard**는 무료로 제공되는 기본 DDoS 방어 서비스이다.
- **AWS WAF**는 웹 애플리케이션 공격(SQL Injection, XSS 등)을 차단한다.
- **AWS Shield Advanced**는 상세 분석과 사전 예방 기능을 제공하는 유료 DDoS 보호 서비스이다.