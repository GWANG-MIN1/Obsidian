# EC2 인스턴스 생성 과정

EC2 인스턴스 생성은 다음 순서로 진행된다.

---

## 1. 이름(Name)

- 생성할 인스턴스의 이름 지정
- 여러 인스턴스를 관리하기 쉽도록 용도를 포함하여 작성

**예시**

- WebServer
- Nginx-Server
- Test-EC2

---

## 2. AMI 선택 (Amazon Machine Image)

AMI는 인스턴스를 생성하기 위한 템플릿이다.

포함 내용:

- 운영체제(OS)
- 스토리지 설정
- CPU 아키텍처(x86, ARM)
- 권한 설정
- 설치된 소프트웨어

**예시**

- Amazon Linux
- Ubuntu
- Windows Server

---

## 3. 인스턴스 타입 선택

인스턴스의 CPU와 메모리 성능을 결정한다.

**예시**

- t2.micro
- t3.micro
- c5.large

→ 웹 서버의 성능과 비용에 직접적인 영향을 준다.

---

## 4. Key Pair 설정

EC2 인스턴스에 안전하게 접속하기 위한 인증 방식이다.

- **Public Key** : EC2 서버에 저장
- **Private Key (.pem)** : 사용자가 보관

SSH 접속 시 사용된다.

```
사용자 PC (Private Key)
          ↓
EC2 인스턴스 (Public Key)
```

---

## 5. 네트워크 설정

- VPC 설정
- 서브넷 설정
- 보안 그룹(Security Group) 설정

주로 다음 포트를 허용한다.

- SSH (22)
- HTTP (80)
- HTTPS (443)

---

## 6. 스토리지 설정

인스턴스에 연결할 디스크(EBS) 용량을 설정한다.

예시:

- 8GB
- 30GB
- 100GB

---

# User Data 설정

User Data는 인스턴스가 처음 부팅될 때 자동으로 실행되는 스크립트이다.

예를 들어 Nginx 설치 코드를 입력하면,

1. EC2 생성
2. 서버 부팅
3. 자동으로 Nginx 설치 및 실행

이 이루어진다.

---

# 인스턴스 실행 과정

```
EC2 콘솔 이동  
↓  
인스턴스 생성  
↓  
AMI 선택  
↓  
인스턴스 타입 선택  
↓  
Key Pair 생성  
↓  
네트워크 설정  
↓  
스토리지 설정  
↓  
User Data 실행  
↓  
Launch  
↓  
Public IP 확인  
↓  
웹 브라우저 접속  
↓  
자체 웹 서버 확인
```

---

## 한 줄 정리

> EC2 인스턴스는 AMI, 인스턴스 타입, Key Pair, 네트워크, 스토리지를 설정한 후 실행하며, User Data를 이용하면 웹 서버(Nginx 등)를 자동으로 설치하고 즉시 사용할 수 있다.