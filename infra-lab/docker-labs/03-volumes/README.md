# 03 볼륨

컨테이너의 쓰기 레이어는 **컨테이너와 생명주기를 같이한다.** 컨테이너를 지우면 그 안에 쌓인 DB 데이터도 함께 사라진다.  
볼륨은 이 쓰기 레이어 바깥에 데이터를 두어 **컨테이너보다 오래 살아남게** 하는 장치다.

---

## 왜 볼륨이 필요한가

```
[ 볼륨 없이 ]
컨테이너 ─ 쓰기 레이어 ─ DB 데이터    ← docker rm 하면 같이 소멸

[ 볼륨 사용 ]
컨테이너 ─ 쓰기 레이어
              │ 마운트
              ▼
      볼륨 ─ DB 데이터                ← 컨테이너를 지워도 남는다
```

쓰기 레이어에 데이터를 두면 생기는 문제:

- **소실** — 컨테이너 삭제 시 데이터도 사라진다
- **성능** — UnionFS의 Copy-on-Write를 거치므로 쓰기가 잦은 워크로드(DB)에 불리하다
- **공유 불가** — 다른 컨테이너가 그 데이터를 읽을 수 없다
- **이식성** — 데이터를 백업·이전하기 어렵다

---

## 마운트 방식 3가지

| 방식 | 저장 위치 | 관리 주체 | 주 용도 |
|---|---|---|---|
| **volume** | `/var/lib/docker/volumes/` | 도커 | DB 데이터 등 영속 데이터 (**권장**) |
| **bind mount** | 호스트의 임의 경로 | 사용자 | 소스 코드 실시간 반영, 설정 파일 주입 |
| **tmpfs** | 호스트 메모리 (디스크 X) | 도커 | 비밀 값·임시 파일 (종료 시 소멸) |

```
        컨테이너
   ┌────────────────┐
   │  /var/lib/mysql│──── volume ────▶ /var/lib/docker/volumes/mydata/_data
   │  /app          │──── bind ──────▶ /home/user/project
   │  /tmp/cache    │──── tmpfs ─────▶ (메모리)
   └────────────────┘
```

---

## 도커 볼륨 (volume)

```bash
docker volume create 볼륨명       # 생성
docker volume ls                  # 목록
docker volume inspect 볼륨명      # 상세 (실제 저장 경로 확인)
docker volume rm 볼륨명           # 삭제
docker volume prune               # 사용 중이 아닌 볼륨 일괄 삭제 (주의)
```

### 볼륨 마운트

```bash
docker run -v 볼륨명:/경로 이미지명

# 예: MySQL 데이터를 볼륨에 보관
docker run -d --name db \
  -v mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=password \
  mysql:5.7
```

> 지정한 볼륨이 없으면 **도커가 자동으로 만들어준다.** 오타를 내면 조용히 엉뚱한 새 볼륨이 생기니 주의.

### 명명 볼륨 vs 익명 볼륨

```bash
docker run -v mydata:/data ubuntu      # 명명 볼륨 — 이름으로 재사용 가능
docker run -v /data ubuntu             # 익명 볼륨 — 해시 이름으로 생성됨
```

> 이미지의 `VOLUME` 지시어나 이름 없는 `-v /경로` 때문에 익명 볼륨이 계속 쌓인다.  
> `docker volume ls -f dangling=true`로 확인하고 정리하자.

---

## 호스트 볼륨 공유 (bind mount)

호스트의 특정 디렉터리를 컨테이너 안에 그대로 연결한다.

```bash
docker run -v /호스트경로:/컨테이너경로 이미지명

# 예: 로컬 소스 코드를 컨테이너에 실시간 반영
docker run -d -v /home/user/site:/usr/share/nginx/html -p 8080:80 nginx

# 설정 파일 하나만 읽기 전용으로 주입
docker run -d -v /home/user/nginx.conf:/etc/nginx/nginx.conf:ro nginx
```

> **`-v`에서 호스트 경로는 반드시 절대 경로여야 한다.** 상대 경로(`./site`)를 쓰면 도커가 그것을 **볼륨 이름으로 해석**해서, 원하는 디렉터리 대신 빈 볼륨이 마운트된다. 파일이 안 보이는 흔한 원인이다.

### 마운트 지점의 기존 내용은 가려진다

컨테이너 안의 `/app`에 원래 파일이 있어도, 그 지점에 마운트하면 마운트된 쪽만 보인다(리눅스 마운트와 동일). 빈 호스트 디렉터리를 마운트해 앱 파일이 통째로 사라진 것처럼 보이는 사고가 자주 난다.

### 권한 문제

bind mount는 호스트의 UID/GID를 그대로 가져온다. 컨테이너 안의 사용자와 호스트 파일 소유자가 다르면 `Permission denied`가 난다.

```bash
docker run --user $(id -u):$(id -g) -v $(pwd):/app 이미지명   # UID 맞추기
sudo chown -R 1000:1000 ./data                               # 또는 호스트 쪽을 맞춘다
```

---

## 볼륨 컨테이너 (--volumes-from)

다른 컨테이너가 가진 볼륨 설정을 그대로 물려받는다.

```bash
docker run --volumes-from 컨테이너명 이미지명

# 예: 데이터 전용 컨테이너를 만들고 여러 컨테이너가 공유
docker create -v /data --name datastore ubuntu:24.04
docker run -it --volumes-from datastore ubuntu:24.04 bash
```

### -v vs --volumes-from

| 옵션 | 설명 |
|---|---|
| `-v` | 내가 직접 볼륨/경로를 지정 |
| `--volumes-from` | 다른 컨테이너의 볼륨 설정을 그대로 따라 씀 |

> 데이터 전용 컨테이너 패턴은 명명 볼륨(`docker volume create`)이 생기기 전의 관례다. 지금은 명명 볼륨을 쓰는 편이 낫다.

---

## `-v` vs `--mount`

같은 일을 하지만 `--mount`가 더 명시적이고, 잘못된 설정에서 조용히 넘어가지 않는다.

```bash
# 축약형
docker run -v mydata:/data ubuntu
docker run -v /home/user/app:/app ubuntu

# 명시형
docker run --mount type=volume,source=mydata,target=/data ubuntu
docker run --mount type=bind,source=/home/user/app,target=/app ubuntu
docker run --mount type=tmpfs,target=/tmp/cache ubuntu
```

| | `-v` | `--mount` |
|---|---|---|
| 문법 | 짧다 | 길지만 명시적 (`type=` 필수) |
| 호스트 경로가 없을 때 | **빈 디렉터리를 자동 생성** | **에러로 알려줌** |
| 권장 | 빠른 실습 | 운영·스크립트 |

---

## 읽기 전용 & tmpfs

```bash
docker run -v mydata:/data:ro ubuntu          # 컨테이너가 수정 못 하게
docker run --tmpfs /tmp/cache:size=64m ubuntu # 메모리에만 존재, 종료 시 소멸
```

---

## 볼륨 백업 & 복원

볼륨은 호스트 경로가 `/var/lib/docker/volumes/` 아래라 직접 만지기보다, 임시 컨테이너로 tar를 뜨는 방식이 표준이다.

```bash
# 백업 — 볼륨을 마운트하고 현재 디렉터리로 tar 추출
docker run --rm \
  -v mysql-data:/data \
  -v $(pwd):/backup \
  ubuntu:24.04 tar czf /backup/mysql-data.tar.gz -C /data .

# 복원
docker run --rm \
  -v mysql-data:/data \
  -v $(pwd):/backup \
  ubuntu:24.04 tar xzf /backup/mysql-data.tar.gz -C /data
```

---

## 배운 점

- `-v`와 `--volume`은 완전히 동일 (단축형)
- **볼륨은 컨테이너를 삭제해도 데이터가 유지된다** — 그게 볼륨을 쓰는 이유
- 태그를 생략하면 자동으로 `:latest`가 적용된다
- 마운트는 **volume / bind mount / tmpfs** 세 가지 — 영속 데이터는 volume, 소스·설정 주입은 bind mount
- 볼륨은 도커가 `/var/lib/docker/volumes/`에서 관리하고, bind mount는 호스트 경로를 그대로 쓴다
- `-v`의 호스트 경로는 **절대 경로**여야 한다 — 상대 경로는 볼륨 이름으로 해석된다
- 마운트 지점의 기존 파일은 가려진다 (앱 파일이 사라진 것처럼 보이는 원인)
- bind mount는 호스트 UID/GID를 따라가므로 권한 충돌이 잦다 → `--user`로 맞춘다
- 운영 스크립트에는 실수를 에러로 잡아주는 `--mount`가 더 안전하다
- 볼륨 백업은 임시 컨테이너에 볼륨과 호스트 디렉터리를 함께 마운트해 tar로 뜬다
