# Amazon Elastic File System (Amazon EFS)

> **여러 EC2 인스턴스가 동시에 사용할 수 있는 완전관리형 공유 파일 시스템**

## 특징

- 여러 EC2 인스턴스에서 동시에 읽기/쓰기 가능
- Linux 파일 시스템(NFS) 제공
- 자동으로 용량 확장 및 축소
- 여러 가용 영역(AZ)에 데이터 저장
- 높은 가용성과 내구성 제공

### 장점

- 공유 액세스
- 자동 확장(Elastic)
- 다중 AZ 이중화
- 서버 수가 늘어나도 동일한 파일 시스템 사용 가능

### 사용 사례

- 웹 서버
- 콘텐츠 관리 시스템(CMS)
- 머신러닝
- 컨테이너 환경(EKS/ECS)
- 홈 디렉터리 공유

### 한 줄 요약
> **여러 EC2가 동시에 사용하는 공유 네트워크 드라이브**

---

# Amazon EBS vs Amazon EFS

| 항목    | Amazon EBS       | Amazon EFS       |
| ----- | ---------------- | ---------------- |
| 저장 방식 | 블록 스토리지          | 파일 스토리지          |
| 연결 대상 | 하나의 EC2          | 여러 EC2           |
| 동시 접근 | ❌ 불가능 (기본적으로 1개) | ✅ 가능             |
| 운영체제  | Linux, Windows   | Linux(NFS)       |
| 범위    | 가용 영역(AZ)        | 리전(Region)       |
| 용량    | 직접 확장 필요         | 자동 확장            |
| 대표 용도 | OS, DB           | 공유 파일 시스템        |

---

# EFS 스토리지 클래스

## Standard

- 자주 사용하는 파일 저장
- 여러 AZ에 저장
- 높은 가용성

---

## Standard-IA (Infrequent Access)

- 자주 사용하지 않는 파일
- 저장 비용 절감
- 접근 시 요금 발생

---

## One Zone

- 하나의 AZ에 저장
- Standard보다 저렴
- AZ 장애 시 데이터 손실 가능

---

## One Zone-IA

- 하나의 AZ 저장
- 자주 사용하지 않는 데이터
- 가장 저렴한 EFS 스토리지 클래스

---

# EFS Lifecycle Management

> **파일 접근 빈도에 따라 자동으로 스토리지 클래스를 변경**

### 자동 전환

- Standard
    ↓
- Standard-IA

또는

- One Zone
    ↓
- One Zone-IA

### 특징

- 일정 기간 접근하지 않은 파일 자동 이동
- 다시 접근하면 Standard(또는 One Zone)로 자동 복귀
- 비용 절감 가능