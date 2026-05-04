# 04. 네트워크

## 도커 네트워크 구조

도커는 컨테이너 간 통신을 위해 가상 네트워크 인터페이스를 사용한다.  
컨테이너를 생성하면 기본적으로 `docker0` 브리지에 연결되며, 각 컨테이너는 고유한 IP를 할당받는다.

```
호스트
└── docker0 (브리지, 172.17.0.1)
    ├── container1 (veth ↔ eth0, 172.17.0.2)
    └── container2 (veth ↔ eth0, 172.17.0.3)
```

### 네트워크 목록 조회
```
docker network ls
```

### 네트워크 상세 정보
```
docker network inspect 네트워크명
```

---

## 네트워크 드라이버 종류

| 드라이버 | 설명 |
|---|---|
| bridge | 기본값. 호스트의 가상 브리지를 통해 컨테이너끼리 연결 |
| host | 컨테이너가 호스트 네트워크를 직접 공유 |
| none | 네트워크 인터페이스 없음 (완전 격리) |
| container | 다른 컨테이너의 네트워크 네임스페이스를 공유 |
| macvlan | 컨테이너에 MAC 주소를 직접 부여해 물리 네트워크처럼 동작 |

---

## 브리지 네트워크 (Bridge)

도커 기본 네트워크. 컨테이너 생성 시 자동으로 `docker0` 브리지에 연결된다.

```bash
# 기본 브리지로 컨테이너 실행
docker run -d --name web nginx

# 사용자 정의 브리지 네트워크 생성
docker network create --driver bridge my-bridge

# 사용자 정의 브리지에 컨테이너 연결
docker run -d --name app --network my-bridge nginx

# 실행 중인 컨테이너를 네트워크에 추가 연결
docker network connect my-bridge 컨테이너명

# 컨테이너를 네트워크에서 분리
docker network disconnect my-bridge 컨테이너명
```

> 기본 `bridge`(docker0)는 컨테이너 이름으로 DNS 해석이 안 된다.  
> 사용자 정의 브리지는 컨테이너 이름으로 자동 DNS 해석이 된다.

---

## 호스트 네트워크 (Host)

컨테이너가 호스트의 네트워크 스택을 그대로 사용한다.  
포트 매핑(`-p`) 없이 호스트 포트를 직접 점유한다.

```bash
docker run -d --network host nginx
```

- 성능이 가장 좋다 (NAT 없음)
- 포트 충돌에 주의 (컨테이너가 80 포트 사용 시 호스트 80도 점유)
- Linux 전용. macOS/Windows Docker Desktop에서는 동작 방식이 다름

---

## 논 네트워크 (None)

네트워크 인터페이스를 완전히 제거한다. `lo`(루프백)만 존재.

```bash
docker run -it --network none ubuntu bash

# 컨테이너 내부 확인
ip addr  # lo만 보임
```

- 외부 통신이 전혀 필요 없는 보안 격리 작업에 사용
- 파일 처리, 암호화 연산 등 네트워크 접근을 차단해야 하는 경우

---

## 컨테이너 네트워크 (Container)

다른 컨테이너의 네트워크 네임스페이스를 공유한다.  
두 컨테이너가 같은 IP와 포트 공간을 사용한다.

```bash
# 기준 컨테이너 실행
docker run -d --name base-container nginx

# base-container 의 네트워크를 공유하는 컨테이너 실행
docker run -it --network container:base-container ubuntu bash

# 확인: 두 컨테이너의 IP가 동일하게 출력됨
docker inspect base-container | grep IPAddress
docker inspect 새컨테이너명 | grep IPAddress
```

- 사이드카 패턴에서 자주 사용 (로그 수집기, 프록시 등)
- 같은 `localhost`를 공유하므로 포트 충돌 주의

---

## 브리지 네트워크와 --net-alias

사용자 정의 브리지 네트워크에서 `--net-alias`를 사용하면  
여러 컨테이너에 동일한 DNS 이름을 붙여 라운드로빈 로드밸런싱처럼 동작시킬 수 있다.

```bash
# 사용자 정의 네트워크 생성
docker network create my-net

# 같은 alias로 컨테이너 여러 개 실행
docker run -d --name web1 --network my-net --net-alias webserver nginx
docker run -d --name web2 --network my-net --net-alias webserver nginx
docker run -d --name web3 --network my-net --net-alias webserver nginx

# 같은 네트워크의 컨테이너에서 DNS 조회
docker run --rm --network my-net ubuntu:24.04 bash -c \
  "apt-get install -y dnsutils -qq && nslookup webserver"
# → web1, web2, web3 의 IP가 모두 반환됨

# curl로 반복 요청 시 각기 다른 컨테이너가 응답
docker run --rm --network my-net curlimages/curl \
  sh -c 'for i in $(seq 5); do curl -s webserver; done'
```

> `--net-alias`는 사용자 정의 브리지에서만 동작한다. 기본 `bridge`(docker0)에서는 사용 불가.

---

## MacVLAN 네트워크

컨테이너에 가상 MAC 주소를 부여해 물리 네트워크의 독립 장치처럼 보이게 한다.  
컨테이너가 물리 스위치로부터 직접 IP를 할당받을 수 있다.

```bash
# MacVLAN 네트워크 생성
# parent: 실제 호스트 네트워크 인터페이스 (eth0 등)
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \
  my-macvlan

# MacVLAN 네트워크에 컨테이너 연결 (IP 수동 지정)
docker run -d \
  --name macvlan-container \
  --network my-macvlan \
  --ip 192.168.1.100 \
  nginx
```

### MacVLAN 802.1q 트렁크 모드 (VLAN 서브인터페이스)

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.10.0/24 \
  --gateway 192.168.10.1 \
  --opt parent=eth0.10 \
  macvlan-vlan10
```

> 호스트 NIC가 `Promiscuous Mode`를 지원해야 한다.  
> 클라우드 환경(AWS, GCP 등)에서는 기본적으로 MacVLAN 사용이 제한된다.

### MacVLAN vs Bridge 비교

| 항목 | Bridge | MacVLAN |
|---|---|---|
| 네트워크 방식 | 가상 브리지(docker0) 경유 | 물리 NIC에 직접 연결 |
| IP 할당 | Docker 내부 IPAM | 외부 DHCP 또는 수동 |
| 호스트↔컨테이너 통신 | 가능 | 기본적으로 불가 (별도 설정 필요) |
| 성능 | 보통 | 높음 (NAT 없음) |
| 사용 사례 | 일반 앱 배포 | 레거시 시스템 연동, 물리 네트워크 통합 |

---

## 네트워크 정리

```bash
# 사용하지 않는 네트워크 삭제
docker network rm 네트워크명

# 미사용 네트워크 일괄 삭제
docker network prune
```

---

## 배운 점

- 기본 `bridge`(docker0)는 컨테이너 이름 DNS 해석 불가 → 사용자 정의 브리지를 써야 이름으로 통신 가능
- `host` 네트워크는 성능이 좋지만 포트 충돌 위험이 있고 Linux 전용
- `none` 네트워크는 보안 격리가 필요한 컨테이너에 사용
- `container` 네트워크는 사이드카 패턴 구현에 활용
- `--net-alias`는 동일 alias에 여러 컨테이너를 묶어 DNS 기반 로드밸런싱 구현 가능
- MacVLAN은 컨테이너를 물리 네트워크의 독립 장치처럼 만들 때 사용
