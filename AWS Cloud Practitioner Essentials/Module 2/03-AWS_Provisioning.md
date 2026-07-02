# AWS API 접근 방법

AWS 서비스는 API를 통해 제어되며, 사용자는 여러 방법으로 AWS API를 호출할 수 있다.

---

## API (Application Programming Interface)

- 애플리케이션끼리 서로 통신할 수 있도록 해주는 인터페이스
- AWS의 모든 서비스는 API를 통해 동작
- 콘솔, CLI, SDK 모두 결국 AWS API를 호출하는 방식

---

## 1. AWS Management Console

- 웹 브라우저에서 사용하는 그래픽 기반 관리 도구
- 직관적이고 사용하기 쉬움

**활용**

- 테스트 환경 구성
- AWS 요금 및 청구서 확인
- 모니터링 및 리소스 상태 확인
- 비기술 사용자도 쉽게 사용 가능

**단점**

- 수동 작업(Manual Provisioning)이 많음
- 반복 작업 시 실수나 오류가 발생할 가능성이 높음

---

## 2. AWS CLI (Command Line Interface)

- 터미널에서 명령어를 통해 AWS를 제어
- AWS API를 간접적으로 호출

**장점**

- 스크립트를 통한 자동화 가능
- 반복 작업을 빠르게 수행 가능
- 배포 및 운영 자동화에 많이 사용됨

**예시**

```
aws ec2 describe-instances
```

→ EC2 인스턴스 정보를 조회

---

## 3. AWS SDK (Software Development Kit)

- 프로그래밍 언어에서 AWS 서비스를 제어할 수 있는 라이브러리
- Python, Java, JavaScript 등 다양한 언어 지원

**활용**

- 애플리케이션 내부에서 AWS 서비스 사용
- 코드로 AWS 자원 관리
- 자동화 프로그램 개발

**예시**

- 현재 리전의 EC2 인스턴스 목록 조회
- S3 파일 업로드
- Lambda 함수 실행

---

## 접근 방법 비교

|방식|특징|장점|단점|
|---|---|---|---|
|Management Console|웹 GUI|쉽고 직관적|수동 작업으로 오류 가능|
|AWS CLI|명령어 기반|자동화 가능|명령어 학습 필요|
|AWS SDK|프로그래밍 기반|애플리케이션에 통합 가능|개발 지식 필요|

---

## 한 줄 정리

> AWS Management Console은 쉽지만 수동 작업이 많고, CLI와 SDK는 자동화를 통해 효율적이고 안정적인 클라우드 운영을 가능하게 한다.