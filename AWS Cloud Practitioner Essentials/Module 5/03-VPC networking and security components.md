# 서브넷 보안 (Network Security)

서브넷에서 외부와 통신할 때 **트래픽 접근을 제어**하는 기능이다.

- **Network ACL (NACL)** : **서브넷 단위**에서 트래픽 제어
- **Security Group** : **EC2 인스턴스 단위**에서 트래픽 제어

즉,

> **서브넷 = Network ACL**
> **인스턴스 = Security Group**

---

# Network ACL (Network Access Control List)

### 역할

- 서브넷의 **경계(Border)** 에서 트래픽 검사
- 서브넷에 **들어오고 나가는 패킷**을 제어

### 특징

- **Stateless**
    - 요청과 응답을 **별도로 검사**
- **허용(Allow)**, **거부(Deny)** 규칙 모두 지원
- 패킷이 **서브넷 경계**를 지날 때 검사

### 한 줄 요약

> **서브넷의 출입문을 관리하는 방화벽**

---

# Security Group

### 역할

- EC2 인스턴스로 들어오고 나가는 트래픽 제어

### 특징

- **Stateful**
    - 인바운드가 허용되면 **응답(Outbound)은 자동 허용**
- **허용(Allow) 규칙만 설정 가능**
- 인스턴스마다 별도로 적용 가능

### 한 줄 요약

> **EC2 인스턴스의 방화벽**


# Security Group vs Network ACL
| 구분     | Security Group | Network ACL             |
| ------ | -------------- | ----------------------- |
| 적용 범위  | 인스턴스 수준        | 서브넷 수준                  |
| 상태     | **Stateful**   | **Stateless**           |
| 규칙     | 허용(Allow)만 가능  | 허용(Allow) + 거부(Deny) 가능 |
| 반환 트래픽 | 자동 허용          | 별도 규칙 필요                |
| 목적     | EC2를 세밀하게 보호   | 서브넷 전체를 광범위하게 보호        |