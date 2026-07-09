# 05 사용자 & 그룹

리눅스는 다중 사용자 시스템이며, 모든 권한은 **사용자(UID)** 와 **그룹(GID)** 에 귀속된다. 서버 운영에서 "최소 권한 원칙"을 지키려면 계정·그룹·sudo를 정확히 다뤄야 한다.

---

## 사용자·그룹의 실체

계정 정보는 몇 개의 텍스트 파일에 저장된다.

| 파일 | 내용 |
|------|------|
| `/etc/passwd` | 사용자 목록 (이름·UID·GID·홈·셸) |
| `/etc/shadow` | 암호화된 비밀번호·만료 정책 (root만 읽기) |
| `/etc/group` | 그룹 목록·구성원 |
| `/etc/gshadow` | 그룹 비밀번호 |

```bash
# /etc/passwd 한 줄의 구조
alice:x:1001:1001:Alice,,,:/home/alice:/bin/bash
 │    │  │    │    │        │            └ 로그인 셸
 │    │  │    │    │        └ 홈 디렉터리
 │    │  │    │    └ 설명(GECOS)
 │    │  │    └ 기본 GID
 │    │  └ UID
 │    └ 비밀번호 자리 (x = shadow 참조)
 └ 사용자명
```

| UID 범위 | 용도 |
|----------|------|
| `0` | root (슈퍼유저) |
| `1~999` | 시스템 계정 (서비스용, 예: `www-data`, `nginx`) |
| `1000+` | 일반 사용자 |

---

## 현재 신원 확인

```bash
whoami           # 현재 사용자명
id               # UID·GID·소속 그룹 전체
id alice         # 특정 사용자 정보
groups           # 소속 그룹 목록
who / w          # 현재 로그인한 사용자
last             # 로그인 이력
```

---

## 사용자 관리

```bash
useradd -m -s /bin/bash alice     # 홈(-m) 생성 + 셸 지정
passwd alice                      # 비밀번호 설정
usermod -s /bin/zsh alice         # 셸 변경
usermod -L alice                  # 계정 잠금 (로그인 차단)
usermod -U alice                  # 잠금 해제
userdel alice                     # 계정 삭제 (홈 유지)
userdel -r alice                  # 홈·메일까지 삭제
```

> `useradd`(저수준)와 `adduser`(대화형 래퍼, Debian 계열)가 있다. 스크립트에는 `useradd`, 수동 작업엔 `adduser`가 편하다.

---

## 그룹 관리

```bash
groupadd dev                      # 그룹 생성
usermod -aG docker alice          # 보조 그룹 추가 (-a 필수!)
gpasswd -d alice docker           # 그룹에서 제거
groupdel dev                      # 그룹 삭제
newgrp dev                        # 현재 세션의 기본 그룹 전환
```

> **`-a`(append)를 빠뜨리면 기존 보조 그룹이 전부 사라진다.** 항상 `usermod -aG`로 추가한다. 그룹 변경은 재로그인해야 반영된다.

---

## 기본 그룹 vs 보조 그룹

- **기본 그룹(Primary)**: `/etc/passwd`의 GID. 새로 만든 파일의 그룹이 됨
- **보조 그룹(Secondary)**: `docker`, `sudo`, `wheel` 등 추가 권한을 부여

```bash
# alice를 docker 그룹에 넣어 sudo 없이 docker 실행 허용
usermod -aG docker alice
```

---

## sudo — 권한 상승

특정 사용자가 root 권한으로 명령을 실행하도록 위임한다. **root 로그인을 직접 열지 않고** 필요한 명령만 허용하는 것이 안전하다.

```bash
sudo apt update           # root 권한으로 단일 명령
sudo -i                   # root 셸로 진입
sudo -u www-data cmd      # 특정 사용자로 실행
sudo -l                   # 내가 허용된 sudo 명령 확인
```

### sudoers 설정

`/etc/sudoers`는 반드시 `visudo`로 편집한다 (문법 오류를 막아 잠김 방지).

```bash
visudo

# 예시 규칙
alice   ALL=(ALL:ALL) ALL              # 모든 명령 허용
%dev    ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx  # 그룹에 특정 명령만, 비번 없이
```

| 배포판 | 관리자 그룹 |
|--------|-------------|
| Ubuntu/Debian | `sudo` |
| RHEL/CentOS/Fedora | `wheel` |

---

## 계정 보안 실무

```bash
chage -l alice            # 비밀번호 만료 정책 확인
chage -M 90 alice         # 90일마다 변경 강제
passwd -l alice           # 비밀번호 잠금

# 로그인 셸 없는 서비스 계정 (SSH 로그인 차단)
useradd -r -s /usr/sbin/nologin appsvc
```

> 서비스용 계정은 `nologin` 셸을 주어 대화형 로그인을 막고, 애플리케이션 실행 전용으로만 쓴다 — 컨테이너의 비-root 실행과 같은 원리.

---

## 배운 점
- 모든 권한은 **UID/GID**에 귀속, 계정 정보는 `/etc/passwd`·`shadow`에 저장
- UID 0=root, 1~999=시스템 계정, 1000+=일반 사용자
- 그룹 추가는 반드시 **`usermod -aG`** (–a 없으면 기존 그룹 소실)
- root 직접 로그인 대신 **sudo로 최소 권한 위임**, `sudoers`는 `visudo`로 편집
- 서비스 계정은 `nologin` 셸로 로그인 차단 — 최소 권한 원칙
