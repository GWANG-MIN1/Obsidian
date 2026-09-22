# 07 DNS

DNS는 **`www.example.com` 같은 이름을 `93.184.215.14` 같은 IP로 바꿔주는 인터넷의 전화번호부**다.
"도메인을 옮겼는데 일부 사용자만 옛날 서버로 가요", "IP로는 되는데 도메인으로는 안 돼요" 같은 장애가 전부 여기서 나온다.
**"It's always DNS"** 라는 운영 격언이 있을 정도로 장애 원인 1순위 후보다.

---

## 이름을 IP로 바꾸는 순서 — 내 컴퓨터 안에서

브라우저에 도메인을 치면, 가까운 곳부터 찾는다.

1. **브라우저 캐시** — 최근에 찾아본 적 있으면 끝
2. **OS 캐시** — (systemd-resolved 등)
3. **`/etc/hosts`** — 파일에 직접 적어둔 이름
4. **DNS 리졸버** — `/etc/resolv.conf`의 `nameserver`에 적힌 서버에 물어본다

> `/etc/hosts`와 DNS 중 뭘 먼저 볼지는 `/etc/nsswitch.conf`의 `hosts: files dns` 줄이 정한다. 보통 **files(= /etc/hosts)가 먼저**다.
> 그래서 누가 `/etc/hosts`에 옛 IP를 적어두면, DNS를 아무리 고쳐도 그 서버만 옛 IP로 간다.

---

## 리졸버가 답을 찾는 과정 — 재귀 vs 반복

리졸버(통신사 DNS, `8.8.8.8`, VPC의 AmazonProvidedDNS)가 답을 모르면 **맨 위부터 차례로** 물어본다.

```
내 PC ──(1) www.example.com 알려줘──▶ 리졸버 (재귀 질의: "최종 답을 가져와")
                                        │
                                        ├─(2)▶ 루트 서버 (.)      "com은 저쪽에 물어봐"
                                        ├─(3)▶ TLD 서버 (.com)    "example.com은 저쪽 네임서버에"
                                        └─(4)▶ 권한 있는 서버      "www.example.com = 93.184.215.14"
                                                (authoritative, 예: Route 53)
내 PC ◀──(5) 93.184.215.14 ───────────── 리졸버 (받은 답을 TTL 동안 캐시)
```

- **재귀 질의(recursive)** — 내 PC → 리졸버: "알아서 **최종 답**을 가져와줘"
- **반복 질의(iterative)** — 리졸버 → 각 서버: "모르면 **누구한테 물어볼지**라도 알려줘"
- **권한 있는 네임서버(authoritative)** — 그 도메인의 **진짜 기록**을 가진 서버. 우리가 Route 53에 레코드를 등록하면 Route 53이 이 역할을 한다.

> 도메인을 가비아에서 사고 Route 53으로 관리하려면, 가비아에서 **네임서버(NS)를 Route 53이 알려준 4개로 바꿔야** 한다. 이게 위 (3)번 단계가 Route 53을 가리키게 만드는 작업이다.

---

## 레코드 타입

| 타입 | 뜻 | 예 |
|---|---|---|
| `A` | 이름 → **IPv4** | `www → 93.184.215.14` |
| `AAAA` | 이름 → **IPv6** | |
| `CNAME` | 이름 → **다른 이름** (별명) | `blog → myblog.github.io` |
| `MX` | 이 도메인의 **메일 서버** | 우선순위 숫자와 함께 |
| `TXT` | 아무 텍스트 | **도메인 소유 인증**, SPF·DKIM(메일 스푸핑 방지) |
| `NS` | 이 도메인을 **담당하는 네임서버** | Route 53 네임서버 4개 |
| `SOA` | 이 존(zone)의 기본 정보 | 관리자, 시리얼 번호 |
| `PTR` | **IP → 이름** (역방향) | 메일 서버 신뢰도 확인에 사용 |

> ACM에서 인증서를 발급받을 때 "DNS 검증"을 고르면 **CNAME 레코드를 추가하라**고 한다. 도메인 주인만 레코드를 추가할 수 있으니 소유 증명이 되는 원리.

---

## TTL — 캐시 유효기간

모든 레코드에는 **TTL(Time To Live, 초 단위)** 이 붙는다. 리졸버는 답을 받으면 **TTL 동안은 다시 묻지 않고 캐시된 답**을 준다.

| TTL | 장점 | 단점 |
|---|---|---|
| 길게 (86400 = 1일) | 조회가 줄어 빠르고 저렴 | 바꿔도 **최대 하루 동안 옛 값**이 퍼져 있음 |
| 짧게 (60 = 1분) | 변경이 빨리 반영 | 조회가 많아짐 |

**"DNS 전파(propagation)"의 정체**: 전 세계에 뭔가를 퍼뜨리는 게 아니라, **각 리졸버가 들고 있던 캐시가 TTL만큼 지나서 만료되길 기다리는 것**이다.

> **서버 이전 실무 팁**: 이전 **며칠 전에 TTL을 60초로 낮춰두고** → 기존 TTL이 지나 모든 캐시가 짧은 TTL로 바뀐 뒤 → IP 변경 → 안정되면 TTL을 다시 올린다.
> 변경 당일에 TTL을 낮추면 **이미 캐시된 옛 TTL**은 그대로라 소용이 없다.

---

## CNAME vs Alias — AWS 면접 단골

### CNAME의 제약: 존 apex에는 못 쓴다

- `www.example.com → my-alb-123.ap-northeast-2.elb.amazonaws.com` ✅ CNAME 가능
- `example.com`(루트 도메인, **apex**) `→ ALB 주소` ❌ **CNAME 불가**

이유: DNS 규칙상 **CNAME이 있는 이름에는 다른 레코드를 둘 수 없다.** 그런데 apex에는 **NS와 SOA 레코드가 반드시** 있어야 한다. 그래서 충돌한다.
ALB는 IP가 계속 바뀌어서 A 레코드에 IP를 박을 수도 없다. → 그래서 Alias가 필요하다.

### Route 53 Alias

| | CNAME | Alias |
|---|---|---|
| 표준 여부 | DNS 표준 | **Route 53 전용 기능** |
| apex(`example.com`)에 사용 | ❌ | ✅ |
| 응답 | "저 이름으로 다시 물어봐" (**조회가 한 번 더**) | Route 53이 대상의 **IP를 바로 A 레코드로** 응답 |
| 대상 | 아무 도메인 | **AWS 리소스** (ALB/NLB, CloudFront, S3 웹사이트, API Gateway, 같은 존의 다른 레코드 등) |
| 조회 요금 | 과금 | AWS 리소스 대상이면 **무료** |
| TTL | 직접 설정 | 대상 리소스의 TTL을 따름 |

> 한 줄: **"AWS 리소스를 가리킬 땐 Alias, 특히 루트 도메인은 Alias만 된다."**

---

## 확인 명령어

```bash
dig www.example.com              # 전체 응답 (ANSWER 섹션, TTL, 어느 서버가 답했는지)
dig +short www.example.com       # IP만
dig www.example.com @8.8.8.8     # 특정 리졸버에게 직접 물어보기 (캐시 비교할 때)
dig +trace www.example.com       # 루트부터 차례로 따라가기 (위임이 어디서 끊기는지)
dig example.com NS               # 담당 네임서버 확인
dig -x 93.184.215.14             # 역방향(PTR)
nslookup www.example.com         # 간단 조회 (윈도우에도 있음)
getent hosts www.example.com     # /etc/hosts까지 포함해서 "OS가 실제로" 푸는 결과
```

> **`dig`는 되는데 `curl`은 옛 IP로 간다?** `dig`는 `/etc/hosts`를 안 보고 DNS 서버에 바로 묻는다. 앱은 `/etc/hosts`를 먼저 본다. 이럴 땐 `getent hosts`로 확인하면 차이가 보인다.

---

## 클라우드·컨테이너 환경의 DNS

- **VPC**: VPC CIDR의 **+2 주소**(예: `10.0.0.2`)가 AmazonProvidedDNS다. ([06](06-subnet-routing-nat.md)의 예약 IP 5개 중 하나) VPC 설정에서 `enableDnsSupport`, `enableDnsHostnames`가 켜져 있어야 이름 해석과 퍼블릭 DNS 이름이 동작한다.
- **Private Hosted Zone**: VPC 안에서만 보이는 내부 도메인 (예: `db.internal`)
- **쿠버네티스**: Pod의 `/etc/resolv.conf`는 **CoreDNS**를 가리킨다. `my-svc.my-namespace.svc.cluster.local` 형식으로 Service를 찾는다.
- **Docker**: 사용자 정의 네트워크에서는 **컨테이너 이름으로** 서로 찾을 수 있다 (내장 DNS `127.0.0.11`)

---

## DNS 장애 확인 순서

**"IP로는 되는데 도메인으로는 안 돼요"**
1. `dig +short 도메인` — 이름이 풀리긴 하나? 올바른 IP인가?
2. 안 풀리면 → `/etc/resolv.conf`의 리졸버에 닿는지, 방화벽/SG에서 **UDP·TCP 53**이 막혔는지
3. `dig +trace` — 위임이 어디서 끊기나? (NS 설정을 잘못 바꾼 경우)
4. `dig`는 맞는데 앱은 틀리면 → `/etc/hosts`, `getent hosts`, 앱 자체 DNS 캐시(JVM 등)

**"도메인을 새 서버로 바꿨는데 일부 사용자만 옛 서버로 가요"**
- 각 리졸버에 **옛 레코드가 TTL 동안 캐시**되어 있어서다. 기다리면 해결된다.
- `dig 도메인 @8.8.8.8`, `@1.1.1.1` 등으로 리졸버마다 결과를 비교하고, 남은 TTL을 확인한다.
- 다음부터는 **변경 전에 TTL을 미리 낮춘다.**

---

## 면접에서 이렇게 말한다

**Q. DNS 동작 과정을 설명해주세요.**
> 먼저 브라우저와 OS 캐시, /etc/hosts를 확인하고, 없으면 설정된 리졸버에 재귀 질의를 합니다. 리졸버는 루트 서버에서 TLD 서버, 그다음 그 도메인의 권한 있는 네임서버 순서로 반복 질의해서 최종 IP를 받고, TTL 동안 캐시한 뒤 클라이언트에 돌려줍니다.

**Q. TTL이 뭐고, 서버 이전할 때 어떻게 활용하나요?**
> 리졸버가 응답을 캐시해두는 시간입니다. TTL이 길면 레코드를 바꿔도 캐시가 만료될 때까지 옛 IP로 가기 때문에, 이전 며칠 전에 TTL을 짧게 낮춰두고, 기존 캐시가 다 만료된 뒤 레코드를 바꾸고, 안정되면 다시 올립니다.

**Q. CNAME과 Route 53 Alias의 차이는?**
> CNAME은 이름을 다른 이름으로 연결하는 표준 레코드인데, 다른 레코드와 공존할 수 없어서 NS와 SOA가 있는 루트 도메인에는 쓸 수 없습니다. Alias는 Route 53 기능으로, ALB나 CloudFront 같은 AWS 리소스를 가리키면 Route 53이 대상 IP를 A 레코드로 바로 응답해줍니다. 그래서 루트 도메인에도 쓸 수 있고 AWS 리소스 대상 조회는 무료입니다.

**Q. DNS는 TCP인가요 UDP인가요?**
> 일반 조회는 빠른 UDP 53을 쓰고, 응답이 커서 UDP로 담기 어렵거나 존 전송을 할 때는 TCP 53을 씁니다. 그래서 방화벽에서는 둘 다 허용해야 합니다.

**Q. 도메인을 바꿨는데 일부 사용자만 옛 서버로 접속합니다.**
> 리졸버마다 옛 레코드가 TTL 동안 캐시되어 있어서입니다. dig로 여러 공용 리졸버에 직접 물어서 결과와 남은 TTL을 비교하고, 특정 서버만 문제면 /etc/hosts에 옛 IP가 적혀 있는지도 확인합니다. 근본적으로는 변경 전에 TTL을 미리 낮춰두는 게 해결책입니다.

---

## 더 보기

- [infra-lab/linux-scripts/08-networking](../infra-lab/linux-scripts/08-networking/README.md) — `dig`, `/etc/resolv.conf` 실습
- [infra-lab/docker-labs/04-network](../infra-lab/docker-labs/04-network/README.md) — 컨테이너 이름 기반 DNS
- [06 서브넷 · 라우팅 · NAT](06-subnet-routing-nat.md) — VPC 예약 IP와 DNS 주소
- [README 종합 질문](README.md#종합-질문--전부-엮어서-나오는-단골) — 브라우저에 주소를 치면 첫 단계가 DNS
