## 컨테이너(Container)

> **애플리케이션과 실행에 필요한 모든 요소를 하나로 묶어 어디서나 동일하게 실행할 수 있는 기술**

### 핵심 특징

- **하나의 패키지로 구성**
    - 코드(Code)
    - 런타임(Runtime)
    - 설정(Configuration)
    - 종속성(Dependencies)
- **장점**
    - 일관된 실행 환경 제공
    - 배포가 쉬움
    - 빠른 시작 시간(Start-up Time)
    - 높은 리소스 효율성

---

## 컨테이너 오케스트레이션(Container Orchestration)

> **여러 컨테이너의 생성부터 종료까지 전체 생명주기를 자동으로 관리하는 서비스**

### 주요 기능

- 자동 확장/축소 (Scale Out / Scale In)
- 장애 발생 시 자동 복구
- 애플리케이션 업데이트 자동화
- 컨테이너 수명 주기 관리

---

## AWS 컨테이너 서비스

### 1. Amazon ECS (Elastic Container Service)

- AWS 자체 컨테이너 오케스트레이션 서비스
- **간단하고 사용하기 쉬움**
- AWS 서비스와 높은 통합성
- 일부 설정만으로 사용할 수 있는 **완전 관리형 서비스**

### 2. Amazon EKS (Elastic Kubernetes Service)

- **Kubernetes 기반**의 컨테이너 오케스트레이션 서비스
- ECS보다 설정이 복잡
- **더 높은 제어권과 유연성** 제공

### 3. Amazon ECR (Elastic Container Registry)

- **컨테이너 이미지를 저장·관리하는 완전 관리형 Docker Registry**
- Docker 이미지를 안전하게 저장하고 버전 관리 가능

### 4. AWS Fargate

- **서버리스 컨테이너 실행 서비스**
- 서버를 관리할 필요 없이 컨테이너만 실행
- 인프라 운영 없이 높은 운영 효율성 제공

---

## 컨테이너 배포 과정

```
① 컨테이너 이미지 생성
            │
            ▼
② Amazon ECR에 이미지 업로드
            │
            ▼
③ 오케스트레이션 서비스 선택
   ├─ Amazon ECS
   └─ Amazon EKS
            │
            ▼
④ 실행 환경 선택
   ├─ Amazon EC2
   └─ AWS Fargate
            │
            ▼
⑤ 컨테이너 실행 및 자동 관리
```

### 한 줄 요약

- **ECR** → 컨테이너 이미지를 저장
- **ECS / EKS** → 컨테이너를 관리(오케스트레이션)
- **EC2 / Fargate** → 컨테이너를 실제 실행하는 컴퓨팅 환경
    - **EC2:** 서버를 직접 관리
    - **Fargate:** 서버 관리 없이 컨테이너만 실행