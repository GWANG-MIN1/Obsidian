# AWS 클라우드 연결(Connection)
| 서비스                      | 역할                            | 언제 사용?              | 핵심 키워드              |
| ------------------------ | ----------------------------- | ------------------- | ------------------- |
| **AWS Client VPN**       | 사용자가 인터넷을 통해 AWS VPC에 안전하게 접속 | 재택근무, 원격 접속         | 원격 액세스, VPN, 완전 관리형 |
| **AWS Site-to-Site VPN** | 회사 네트워크와 AWS를 VPN으로 연결        | 온프레미스 ↔ AWS 연결      | 고가용성, 보안, 프라이빗 연결   |
| **AWS PrivateLink**      | 인터넷을 거치지 않고 AWS 서비스 간 안전하게 연결 | VPC 간 서비스 공유        | 프라이빗 연결, 트래픽 보호     |
| **AWS Direct Connect**   | 전용 회선을 이용하여 AWS와 직접 연결        | 대용량 데이터, 낮은 지연시간 필요 | 전용선, 고대역폭, 저지연      |

## AWS Direct Connect를 사용하는 경우

- 지연 시간(Latency)에 민감한 애플리케이션
- 대규모 데이터 마이그레이션 및 데이터 전송
- 하이브리드 클라우드(Hybrid Cloud) 환경 구축
- 인터넷 대신 전용 회선을 사용하여 안정적인 통신


## AWS Gateway 서비스

|Gateway|역할|한 줄 설명|
|---|---|---|
|**AWS Transit Gateway**|여러 VPC와 온프레미스를 하나의 게이트웨이로 연결|네트워크 허브 역할|
|**NAT Gateway**|Private Subnet에서 인터넷으로 나갈 수 있도록 지원|아웃바운드 인터넷 연결|
|**Amazon API Gateway**|API 생성·관리·보안 제공|API의 출입문 역할|
## 한눈에 이해하기
```
사용자
   │
   ├── AWS Client VPN
   │
회사 네트워크
   │
   ├── Site-to-Site VPN
   │
전용 회선
   │
   ├── AWS Direct Connect
   │
VPC ↔ AWS 서비스
   │
   ├── AWS PrivateLink
```