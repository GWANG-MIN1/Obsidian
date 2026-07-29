# AWS 추가 서비스 (Additional AWS Services)

## AWS 추가 서비스란?

AWS는 수백 개의 클라우드 서비스를 제공하며, 대부분의 서비스는 **고객의 요구사항과 피드백을 기반으로 특정 사용 사례(Use Case)** 에 맞게 개발되었다.

이러한 서비스들을 조합하면 다양한 비즈니스 요구사항을 해결할 수 있다.

대표적으로 다음과 같은 분야의 서비스가 있다.

- 개발(Developer Tools)
- 비즈니스 애플리케이션(Business Applications)
- 최종 사용자 컴퓨팅(End User Computing)

---

# 1. 개발(Developer Tools)

개발자가 애플리케이션을 **개발, 테스트, 배포 및 운영**할 수 있도록 지원하는 서비스이다.

---

## AWS CodeBuild

### AWS CodeBuild란?

완전관리형 **빌드(Build) 서비스**이다.

소스 코드를 컴파일하고 테스트를 수행한 뒤 배포 가능한 결과물을 생성한다.

### 주요 기능

- 코드 컴파일
- 자동 테스트
- 빌드 자동화
- 서버 관리 불필요

---

## AWS CodePipeline

### AWS CodePipeline이란?

애플리케이션의 **지속적 통합(CI)** 과 **지속적 전달/배포(CD)** 를 자동화하는 서비스이다.

### 주요 기능

- CI/CD 파이프라인 구축
- 자동 빌드
- 자동 테스트
- 자동 배포

### 사용 예시

```
Code Commit

↓

CodeBuild

↓

Test

↓

Deploy
```

---

## AWS X-Ray

### AWS X-Ray란?

분산 애플리케이션을 **모니터링하고 성능을 분석하며 문제를 추적(Tracing)** 하는 서비스이다.

### 주요 기능

- 애플리케이션 성능 분석
- 요청(Request) 추적
- 병목(Bottleneck) 분석
- 오류 위치 확인
- 디버깅 지원

### 장점

- 장애 원인 파악
- 성능 개선
- 빠른 문제 해결

---

## AWS AppSync

### AWS AppSync란?

**GraphQL API**를 쉽고 빠르게 구축할 수 있도록 지원하는 서비스이다.

프론트엔드 애플리케이션과 백엔드 데이터를 효율적으로 연결할 수 있다.

### 주요 기능

- GraphQL API 생성
- 여러 데이터 소스 통합
- 실시간 데이터 처리
- 모바일 및 웹 애플리케이션 지원

---

## AWS Amplify

### AWS Amplify란?

웹 및 모바일 애플리케이션을 **쉽게 개발, 배포 및 관리**할 수 있도록 지원하는 개발 플랫폼이다.

복잡한 백엔드 구성을 자동으로 처리하여 개발 생산성을 높여준다.

### 주요 기능

- 프론트엔드 개발 지원
- 백엔드 서비스 통합
- 자동 배포
- 호스팅
- 인증 기능 제공

### 장점

- 개발 속도 향상
- 복잡한 설정 자동화
- 우수한 사용자 경험 제공

---

# 2. 비즈니스 애플리케이션(Business Applications)

기업의 비즈니스 업무를 지원하는 서비스이다.

---

## Amazon Connect

### Amazon Connect란?

**AI 기반의 완전관리형 클라우드 고객센터(Contact Center)** 서비스이다.

### 주요 기능

- 전화 상담
- 채팅 상담
- AI 기반 고객 응대
- 옴니채널(Omnichannel) 지원
- 고객 서비스 자동화

---

## Amazon Simple Email Service (Amazon SES)

### Amazon SES란?

대량의 이메일을 안전하고 안정적으로 전송할 수 있는 이메일 서비스이다.

### 주요 기능

- 대량 이메일 전송
- 마케팅 이메일
- 알림 이메일
- 인증 이메일
- 높은 전송 신뢰성

### 사용 사례

- 회원가입 인증 메일
- 비밀번호 재설정 메일
- 뉴스레터 발송
- 마케팅 캠페인

---

# 3. 최종 사용자 컴퓨팅 (End User Computing)

최종 사용자가 **언제 어디서나 업무 환경에 안전하게 접근**할 수 있도록 지원하는 서비스이다.

---

## Amazon WorkSpaces

### Amazon WorkSpaces란?

완전관리형 **가상 데스크톱 인프라(VDI, Virtual Desktop Infrastructure)** 서비스이다.

사용자는 인터넷을 통해 어디서든 자신의 데스크톱 환경에 접속할 수 있다.

### 주요 기능

- 가상 데스크톱 제공
- 원격 근무 지원
- 중앙 집중식 관리
- 보안 강화

---

## Amazon WorkSpaces Secure Browser

### Amazon WorkSpaces Secure Browser란?

웹 브라우저 기반 업무를 위한 **관리형 보안 브라우저 서비스**이다.

별도의 가상 데스크톱 없이 웹 애플리케이션에 안전하게 접근할 수 있다.

### 주요 기능

- 웹 애플리케이션 전용
- 보안 브라우징
- 가벼운 원격 업무 환경
- 중앙 관리

---

# 주요 서비스 요약

| 분야 | 서비스 | 주요 기능 |
|------|---------|-----------|
| **개발** | **AWS CodeBuild** | 코드 빌드 및 테스트 자동화 |
| | **AWS CodePipeline** | CI/CD 파이프라인 구축 및 자동 배포 |
| | **AWS X-Ray** | 애플리케이션 모니터링, 성능 분석 및 디버깅 |
| | **AWS AppSync** | GraphQL API 구축 |
| | **AWS Amplify** | 웹·모바일 애플리케이션 개발 및 배포 |
| **비즈니스 애플리케이션** | **Amazon Connect** | AI 기반 클라우드 고객센터 |
| | **Amazon SES** | 대량 이메일 전송 서비스 |
| **최종 사용자 컴퓨팅** | **Amazon WorkSpaces** | 완전관리형 가상 데스크톱(VDI) |
| | **Amazon WorkSpaces Secure Browser** | 웹 애플리케이션용 보안 브라우저 |

---

# 핵심 정리

- AWS는 **고객의 요구사항과 피드백을 기반으로 특정 사용 사례에 맞는 수백 개의 서비스를 제공**한다.
- **AWS CodeBuild**는 코드 빌드와 테스트를 자동화하고, **AWS CodePipeline**은 **CI/CD(지속적 통합 및 지속적 전달/배포)** 를 지원한다.
- **AWS X-Ray**는 애플리케이션의 요청을 추적하여 성능 분석과 디버깅을 지원한다.
- **AWS AppSync**는 **GraphQL API**를 쉽게 구축할 수 있도록 지원하며, **AWS Amplify**는 웹·모바일 애플리케이션의 개발, 배포 및 관리를 간소화한다.
- **Amazon Connect**는 AI 기반 클라우드 고객센터 서비스이며, **Amazon SES**는 대량 이메일 전송 서비스이다.
- **Amazon WorkSpaces**는 완전관리형 가상 데스크톱(VDI) 서비스이고, **Amazon WorkSpaces Secure Browser**는 웹 애플리케이션에 안전하게 접근할 수 있는 관리형 보안 브라우저 서비스이다.