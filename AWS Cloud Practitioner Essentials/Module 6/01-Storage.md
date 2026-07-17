# AWS 스토리지 서비스

## 1. 블록 스토리지 (Block Storage)

> 물리적인 HDD/SSD처럼 EC2 인스턴스에 직접 연결하여 사용하는 스토리지
>
> **특징**
> - 블록(Block) 단위로 데이터 저장
> - 낮은 지연 시간(Low Latency)
> - 높은 성능 제공
> - 운영체제, 데이터베이스 등에 적합
> - EC2에 연결하여 사용

### 종류

- **Amazon EC2 Instance Store**
  - EC2 서버 내부의 임시 디스크
  - 매우 빠른 성능
  - EC2 종료(Stop/Terminate) 시 데이터 유실 가능
  - 임시 데이터 저장에 적합

- **Amazon Elastic Block Store (EBS)**
  - EC2에 연결하는 영구 스토리지
  - 데이터가 지속적으로 유지됨(Persistent)
  - 스냅샷 생성 가능
  - 운영체제, 데이터베이스 등에 주로 사용

---

## 2. 객체 스토리지 (Object Storage)

> 데이터를 **객체(Object)** 형태로 저장하는 스토리지

### 특징
- 파일을 객체(Object)로 저장
- 무제한에 가까운 확장성
- 인터넷을 통해 어디서든 접근 가능
- 이미지, 동영상, 백업 파일 저장에 적합

### 종류

- **Amazon Simple Storage Service (Amazon S3)**
  - 완전관리형 객체 스토리지 서비스
  - 원하는 만큼 데이터 저장 및 검색 가능
  - 높은 내구성(99.999999999%)
  - 백업, 정적 웹사이트, 로그 저장 등에 활용

---

## 3. 파일 스토리지 (File Storage)

> 여러 사용자와 애플리케이션이 동시에 접근 가능한 **공유 파일 시스템** 제공

### 특징
- 파일 및 폴더 구조 사용
- 네트워크를 통해 공유
- 여러 EC2에서 동시에 접근 가능

### 종류

- **Amazon Elastic File System (EFS)**
  - Linux용 완전관리형 파일 시스템
  - 여러 EC2에서 동시에 공유 가능
  - 자동으로 용량 확장

- **Amazon FSx**
  - 다양한 파일 시스템을 관리형으로 제공
  - Windows File Server
  - Lustre
  - NetApp ONTAP
  - OpenZFS 지원

---

# 추가 스토리지 관련 서비스

## Amazon Storage Gateway

> 온프레미스(On-Premises) 환경과 AWS 스토리지를 연결하는 하이브리드 스토리지 서비스

### 특징
- 로컬 스토리지와 AWS 연동
- 백업 및 아카이브 용도
- 온프레미스 → AWS 데이터 이전 지원

---

## AWS Elastic Disaster Recovery (AWS DRS)

> 서버를 AWS로 빠르게 복구할 수 있도록 지원하는 재해 복구(Disaster Recovery) 서비스

### 특징
- 서버를 지속적으로 복제
- 장애 발생 시 AWS에서 신속하게 복구
- 낮은 RPO/RTO 제공
- 비즈니스 연속성(BCP) 확보