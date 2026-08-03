# 02 네트워킹 & 포트 노출

컨테이너는 기본적으로 격리된 네트워크 네임스페이스를 가지므로, **아무 설정도 하지 않으면 외부에서 접근할 수 없다.**  
이 장은 "바깥 → 컨테이너" 방향, 즉 포트를 퍼블리싱해 서비스를 노출하는 방법을 다룬다.  
컨테이너 ↔ 컨테이너 통신과 네트워크 드라이버는 `04-network/`에서 이어진다.

---

## 포트 매핑 (publish)

```bash
docker run -d -p 호스트포트:컨테이너포트 이미지명

# 예: 호스트 8080 → 컨테이너 80
docker run -d --name web -p 8080:80 nginx
```

```
클라이언트 ──▶ 호스트:8080 ──[ iptables DNAT ]──▶ 172.17.0.2:80 (컨테이너)
```

### 매핑 변형

```bash
docker run -d -p 8080:80 nginx              # 호스트 8080 → 컨테이너 80
docker run -d -p 127.0.0.1:8080:80 nginx    # 루프백에만 바인딩 (외부 노출 안 함)
docker run -d -p 80 nginx                   # 호스트 포트를 랜덤으로 할당
docker run -d -P nginx                      # EXPOSE 된 포트 전부를 랜덤 포트로 노출
docker run -d -p 8080:80 -p 8443:443 nginx  # 여러 포트 매핑
docker run -d -p 5353:53/udp dns-image      # UDP 지정
```

> `-p 127.0.0.1:8080:80`은 실무에서 중요하다. 리버스 프록시 뒤에 둘 백엔드 컨테이너를 그냥 `-p 8080:80`으로 열면 **호스트의 모든 인터페이스에 열려 인터넷에 그대로 노출**된다.

---

## 포트 확인

```bash
docker port 컨테이너명
# 80/tcp -> 0.0.0.0:8080
# 80/tcp -> [::]:8080

docker ps    # PORTS 컬럼에서도 확인 가능
```

> `0.0.0.0`(IPv4)과 `[::]`(IPv6) 두 줄이 나오는 것은 정상이다. 하나의 매핑이 두 프로토콜 스택에 각각 바인딩된 것이다.

---

## `-p` vs `EXPOSE`

자주 헷갈리는 부분이다. **실제로 포트를 여는 건 `-p` 뿐이다.**

| 구분 | `EXPOSE` (Dockerfile) | `-p` / `--publish` (run) |
|---|---|---|
| 역할 | "이 컨테이너는 이 포트를 쓴다"는 **문서화·메타데이터** | 호스트 포트를 컨테이너 포트에 **실제로 연결** |
| 방화벽/NAT 규칙 | 만들지 않음 | iptables DNAT 규칙 생성 |
| 외부 접근 | 불가 | 가능 |
| 연관 | `-P`가 EXPOSE된 포트를 랜덤 매핑할 때 참조 | 단독으로 동작 |

```dockerfile
EXPOSE 80        # 선언만 할 뿐, 이것만으로는 열리지 않는다
```

---

## 내부적으로 어떻게 열리나 (iptables NAT)

도커는 포트 매핑을 위해 호스트의 **iptables `nat` 테이블에 DNAT 규칙**을 자동으로 넣는다.

```bash
# 도커가 만든 NAT 규칙 확인 (Linux 호스트)
sudo iptables -t nat -L DOCKER -n

# 예시 출력
# DNAT  tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80
```

- 호스트 8080으로 들어온 패킷의 목적지를 컨테이너 IP:80으로 바꿔치기(DNAT)한다
- 그래서 `--network host`를 쓰면 NAT를 거치지 않아 성능이 더 좋다 (→ `04-network/`)
- 컨테이너를 지웠는데 포트가 잡혀 있다면 규칙이 남은 경우이므로 도커 데몬 재시작으로 정리한다

> **주의:** 도커의 iptables 규칙은 `ufw` 규칙보다 먼저 평가된다. `ufw deny 8080`을 해두어도 `-p 8080:80`으로 띄운 컨테이너는 외부에서 열려 보일 수 있다.

---

## 환경변수 주입

컨테이너 이미지는 그대로 두고 설정만 바꿀 때 쓴다.

```bash
docker run -d -e KEY=value 이미지명
docker run -d -e KEY1=v1 -e KEY2=v2 이미지명
docker run -d --env-file ./app.env 이미지명     # 파일로 한 번에

docker exec 컨테이너명 env                       # 주입된 값 확인
```

> `-e KEY = value`처럼 `=` 앞뒤에 공백을 넣으면 파싱 에러가 난다. **`-e KEY=value` 붙여 쓴다.**  
> 비밀번호 같은 값은 `-e`로 넘기면 `docker inspect`와 프로세스 목록에 그대로 노출된다. 운영에서는 시크릿 관리를 쓴다. → `09-security/`

---

## 실습 — WordPress + MySQL

두 컨테이너를 띄우고, 웹만 외부에 노출하는 구성이다.

```bash
# 1) 컨테이너들이 이름으로 서로를 찾을 수 있도록 사용자 정의 네트워크 생성
docker network create wp-net

# 2) DB — 포트를 퍼블리싱하지 않는다 (외부에 열 이유가 없다)
docker run -d --name wordpressdb \
  --network wp-net \
  -e MYSQL_ROOT_PASSWORD=password \
  -e MYSQL_DATABASE=wordpress \
  mysql:5.7

# 3) 웹 — DB 호스트로 '컨테이너 이름'을 넘긴다
docker run -d --name wordpress \
  --network wp-net \
  -e WORDPRESS_DB_HOST=wordpressdb \
  -e WORDPRESS_DB_USER=root \
  -e WORDPRESS_DB_PASSWORD=password \
  -p 8080:80 \
  wordpress

# 4) 확인
docker ps
docker port wordpress
curl -I http://localhost:8080
```

> `WORDPRESS_DB_HOST`에는 **DB 컨테이너의 이름**을 넣는다. 사용자 정의 네트워크에서는 컨테이너 이름이 곧 DNS 이름이 된다.  
> 기본 `bridge`(docker0)에서는 이름 해석이 안 되므로 이 구성이 동작하지 않는다. → `04-network/`  
> 이렇게 컨테이너 두 개를 손으로 엮는 작업은 `docker compose` 파일 하나로 대체할 수 있다. → `08-compose/`

### 명령이 여러 줄일 때

```bash
docker run -d --name app \
  -e KEY=value \
  -p 8080:80 \
  이미지명
```

> 줄바꿈용 백슬래시(`\`)는 **줄 맨 끝**에 와야 하고, 뒤에 공백이 있으면 안 된다.  
> `\-e`처럼 옵션에 붙여 쓰거나 `\ mysql:5.7`처럼 뒤에 공백을 두면 에러가 난다.

---

## 배운 점

- 컨테이너는 기본적으로 외부에서 접근 불가 — **`-p`로 퍼블리싱해야 열린다**
- `EXPOSE`는 문서화일 뿐, 실제로 포트를 여는 것은 `-p`
- 포트 확인 시 IPv4(`0.0.0.0`), IPv6(`[::]`) 두 줄이 나오는 건 정상
- `-p 127.0.0.1:8080:80`으로 바인딩 주소를 좁히면 불필요한 외부 노출을 막을 수 있다
- 포트 매핑의 실체는 호스트 **iptables DNAT 규칙** — `ufw`보다 먼저 평가되므로 방화벽만 믿으면 안 된다
- `-e` 옵션으로 환경변수 전달, `=` 앞뒤에 띄어쓰기 하면 에러
- 컨테이너 간 통신은 **사용자 정의 네트워크 + 컨테이너 이름(DNS)** 으로 한다
- DB처럼 외부에 열 필요 없는 컨테이너는 포트를 퍼블리싱하지 않는다
