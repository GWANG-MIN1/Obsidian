# interview-prep

MSP(메가존클라우드·베스핀글로벌·kt cloud 등) 기술면접에서 실제로 나오는 **리눅스 + 네트워크 기본기**를 정리한다.

새로 배우는 게 아니라 **"말로 설명할 수 있는 상태"로 만드는 작업**이다.
명령어 레퍼런스는 [infra-lab/linux-scripts](../infra-lab/linux-scripts/)에 이미 있으니, 여기서는 **왜 그렇게 동작하는지**와 **면접에서 어떻게 말할지**에 집중한다.

---

## 목차

| #                              | 주제              | 핵심 키워드                                           |
| ------------------------------ | --------------- | ------------------------------------------------ |
| [01](01-process-signal.md)     | 프로세스와 시그널       | fork/exec, 좀비·고아, SIGTERM vs SIGKILL, 컨테이너 PID 1 |
| [02](02-filesystem.md)         | 파일시스템           | FHS, inode, 하드링크·심볼릭링크, 디스크 풀 장애                 |
| [03](03-permission.md)         | 퍼미션             | rwx, 8진수, 디렉토리 권한, umask, 특수권한                   |
| [04](04-systemd.md)            | systemd         | PID 1, 유닛 파일, start vs enable, journalctl        |
| [05](05-tcp-handshake.md)      | TCP 핸드셰이크       | 3-way, 4-way, TIME_WAIT, refused vs timeout      |
| [06](06-subnet-routing-nat.md) | 서브넷 · 라우팅 · NAT | CIDR 계산, 게이트웨이, SNAT/DNAT, IGW vs NAT GW         |
| [07](07-dns.md)                | DNS             | 재귀 조회, 레코드 타입, TTL, CNAME vs Alias               |

---

## 공부 방법

1. 노트를 한 번 읽는다
2. **노트를 덮고** 각 노트 끝의 "면접에서 이렇게 말한다" 질문에 소리 내서 답해본다 (30초~1분)
3. 막힌 부분만 다시 본다
4. 명령어는 실제 리눅스(EC2, Docker 컨테이너 등)에서 직접 쳐본다

> 면접관이 보는 건 암기량이 아니라 **"이 사람이 장애 상황에서 원인을 좁혀갈 수 있나"** 이다.
> 그래서 각 노트에 "이런 장애가 나면?" 형태의 질문을 같이 넣었다.

---

## 종합 질문 — 전부 엮어서 나오는 단골

**Q. 브라우저에 `www.example.com`을 치면 무슨 일이 일어나나요?**

1. **DNS**: 도메인을 IP로 바꾼다. 브라우저 캐시 → OS 캐시 → `/etc/hosts` → DNS 리졸버 순서로 찾는다 → [07](07-dns.md)
2. **라우팅**: 목적지 IP가 내 서브넷 밖이니까 기본 게이트웨이(공유기)로 보낸다 → [06](06-subnet-routing-nat.md)
3. **NAT**: 공유기가 내 사설 IP를 공인 IP로 바꿔서 인터넷으로 내보낸다 → [06](06-subnet-routing-nat.md)
4. **TCP**: 서버의 443 포트와 3-way 핸드셰이크로 연결을 맺는다 → [05](05-tcp-handshake.md)
5. **TLS**: HTTPS니까 인증서 확인하고 암호화 키를 정한다
6. **HTTP**: 요청을 보내고 응답(HTML)을 받는다
7. **서버 쪽**: systemd가 띄워 둔 nginx 같은 웹서버 **프로세스**가 요청을 받아 처리한다 → [04](04-systemd.md), [01](01-process-signal.md)

> 이 질문은 "어디까지 깊게 들어가는지"를 보는 질문이다. 먼저 위 흐름을 1분 안에 쭉 말하고,
> 면접관이 파고드는 단계(보통 DNS나 TCP)에서 자세히 들어가면 된다.

---

## 같이 보면 좋은 곳

- [infra-lab/linux-scripts](../infra-lab/linux-scripts/) — 리눅스 명령어 실습 정리
- [infra-lab/linux-scripts/commands.md](../infra-lab/linux-scripts/commands.md) — 명령어 레퍼런스
- [AWS Cloud Practitioner Essentials/Module 5](../AWS%20Cloud%20Practitioner%20Essentials/Module%205/) — VPC 네트워킹
