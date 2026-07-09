# 08 네트워킹

서버는 결국 네트워크로 연결되어야 쓸모가 있다. IP·라우팅·포트·방화벽·DNS·SSH는 서버 운영과 트러블슈팅의 핵심이며, Docker 네트워크·K8s Service도 이 위에 얹힌다.

---

## IP 주소 & 인터페이스

`ifconfig`는 구식이고, 현재 표준은 `ip` 명령이다.

```bash
ip addr                       # IP 주소·인터페이스 (약어: ip a)
ip -br addr                   # 간략 요약
ip link                       # 인터페이스 상태 (up/down)
ip link set eth0 up           # 인터페이스 활성화
hostname -I                   # 내 IP만 빠르게
```

---

## 라우팅

```bash
ip route                      # 라우팅 테이블 (약어: ip r)
ip route get 8.8.8.8          # 특정 목적지로 가는 경로
ip route add 10.0.0.0/24 via 192.168.1.1   # 정적 라우트 추가
```

`default via ...`가 **기본 게이트웨이** — 내부에서 못 찾는 목적지는 여기로 나간다.

---

## 포트 & 연결 상태

`netstat`도 구식, 현재는 `ss`(socket statistics)를 쓴다.

```bash
ss -tulpn                     # TCP/UDP 리스닝 포트 + 프로세스
ss -tan                       # 모든 TCP 연결
ss -s                         # 연결 통계 요약
lsof -i :80                   # 80포트 사용 프로세스
```

`ss -tulpn` 옵션: `t`(TCP)·`u`(UDP)·`l`(listening)·`p`(process)·`n`(숫자 포트).

> "포트가 안 열린다" 문제의 1순위 진단: `ss -tulpn | grep :포트`로 **서비스가 실제로 리스닝 중인지** 확인.

---

## 연결 진단

```bash
ping -c 4 8.8.8.8             # 도달성·지연 (4회)
traceroute example.com        # 경로 추적 (어디서 끊기는지)
mtr example.com               # ping+traceroute 실시간 결합
curl -I https://example.com   # HTTP 헤더 응답
curl -v telnet://host:5432    # 특정 포트 열림 확인
nc -zv host 22                # 포트 스캔 (netcat)
telnet host 80                # 수동 연결 테스트
```

### 진단 순서 (계층별로 좁히기)

```
1. ping IP        → L3 도달 가능?  (실패: 라우팅/방화벽)
2. ping 도메인    → DNS 되는가?    (실패: DNS 문제)
3. curl/nc 포트   → 서비스 응답?   (실패: 서비스/방화벽 포트)
```

---

## DNS

```bash
dig example.com               # 상세 DNS 조회 (권장)
dig +short example.com        # IP만
dig example.com MX            # 메일 레코드
nslookup example.com          # 간단 조회
host example.com              # 짧은 조회
cat /etc/resolv.conf          # 사용 중인 DNS 서버
cat /etc/hosts                # 로컬 정적 매핑 (DNS보다 우선)
```

> `/etc/hosts`는 DNS보다 먼저 조회된다. 테스트·긴급 우회에 유용하지만, 남아있는 잘못된 항목이 "왜 이 도메인만 이상하지?"의 원인이 되기도 한다.

---

## 방화벽

| 도구 | 배포판 | 특징 |
|------|--------|------|
| `ufw` | Ubuntu | 가장 단순한 프론트엔드 |
| `firewalld` | RHEL/Fedora | zone 기반, 동적 |
| `iptables` / `nftables` | 공통 | 저수준, 세밀한 제어 |

### ufw (Ubuntu)

```bash
ufw status                    # 상태·규칙 확인
ufw allow 22/tcp              # SSH 허용
ufw allow 80,443/tcp          # 웹 허용
ufw deny 3306                 # 특정 포트 차단
ufw enable                    # 방화벽 활성화
```

> **방화벽을 켜기 전에 SSH(22) 포트를 먼저 허용**하지 않으면 원격 서버에서 스스로를 잠글 수 있다.

### firewalld (RHEL)

```bash
firewall-cmd --list-all
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --add-service=https --permanent
firewall-cmd --reload
```

### iptables (저수준)

```bash
iptables -L -n -v             # 규칙 확인
iptables -A INPUT -p tcp --dport 22 -j ACCEPT   # 22 허용
```

> Docker는 컨테이너 포트 매핑을 위해 `iptables` 규칙을 자동으로 조작한다 — 컨테이너 네트워크가 결국 호스트 방화벽 규칙 위에서 돈다는 증거.

---

## SSH — 원격 접속

```bash
ssh user@host                        # 기본 접속
ssh -i ~/.ssh/key.pem user@host      # 키 지정
ssh -p 2222 user@host                # 포트 지정
ssh-keygen -t ed25519                # 키 쌍 생성 (ed25519 권장)
ssh-copy-id user@host                # 공개키를 서버에 등록
scp file user@host:/path             # 파일 복사
scp -r dir user@host:/path           # 디렉터리 복사
rsync -avz src/ user@host:/dst/      # 증분 동기화 (효율적)
```

### SSH 보안 강화 (`/etc/ssh/sshd_config`)

```
PermitRootLogin no             # root 직접 로그인 차단
PasswordAuthentication no      # 키 인증만 허용
Port 2222                      # 기본 포트 변경 (봇 스캔 감소)
```

```bash
systemctl restart sshd         # 변경 적용
```

---

## 배운 점
- 현대 도구는 `ip`(주소·라우팅)·`ss`(포트) — `ifconfig`·`netstat`은 구식
- 포트 문제는 **`ss -tulpn`으로 리스닝 여부부터** 확인
- 네트워크 진단은 **ping IP → ping 도메인 → 포트** 순으로 계층을 좁힌다
- `/etc/hosts`는 DNS보다 우선 — 우회에도 함정에도 쓰인다
- 방화벽은 **SSH 포트 먼저 허용** 후 활성화, Docker는 iptables를 자동 조작
- SSH는 **키 인증 + root 로그인 차단**이 기본 보안
