VPC (Virtual Private Cloud)
: AWS 클라우드 안에 만드는 나만의 논리적으로 격리된 가상 네트워크

- EC2, RDS 등의 리소스를 생성하고 관리하는 공간
- 필요한 네트워크를 직접 설계 가능
- 인터넷 연결 여부에 따라 Public / Private으로 구성 가능

    ├─ Public Subnet
    │   - 인터넷과 직접 통신 가능
    │   - 웹 서버, 로드 밸런서 등을 배치
    │
    └─ Private Subnet
        - 인터넷에서 직접 접근 불가
        - DB 서버, 내부 애플리케이션 서버 등을 배치



구조
AWS Client
   │
Internet
   │
AWS Region
└─ VPC
   ├─ AZ-A
   │  ├─ Public EC2
   │  └─ Private DB
   │
   └─ AZ-B
      ├─ Public EC2
      └─ Private DB

