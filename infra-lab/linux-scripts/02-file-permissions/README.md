# 02 파일 권한 & 소유권

리눅스는 다중 사용자 시스템이므로, **누가 무엇을 할 수 있는지**를 파일 단위로 통제한다. 컨테이너 보안의 "비-root 실행"·"읽기 전용 FS"도 결국 이 권한 모델 위에 있다.

---

## 권한 읽는 법

`ls -l` 출력의 첫 열이 권한을 나타낸다.

```
-rwxr-xr--  1  alice  dev  4096  Jul  9 10:00  script.sh
│└┬┘└┬┘└┬┘     │      │
│ │  │  └─ others (그 외 모두): r--
│ │  └──── group (그룹): r-x
│ └─────── user (소유자): rwx
└───────── 파일 타입: - 일반 / d 디렉터리 / l 링크
```

| 권한 | 파일 | 디렉터리 |
|------|------|----------|
| `r` (read, 4) | 내용 읽기 | 목록 보기 (`ls`) |
| `w` (write, 2) | 내용 수정 | 파일 생성·삭제 |
| `x` (execute, 1) | 실행 | 진입(`cd`)·접근 |

> 디렉터리의 `x`가 없으면 그 안으로 `cd`조차 못 한다. `r`만 있으면 이름은 보이지만 접근은 막힌다.

---

## 숫자(8진수) 모드

r=4, w=2, x=1을 더해 각 자리(소유자·그룹·기타)를 한 숫자로 표현한다.

```
rwx = 4+2+1 = 7
rw- = 4+2   = 6
r-x = 4+1   = 5
r-- = 4     = 4
```

| 모드 | 의미 | 용도 |
|------|------|------|
| `755` | rwxr-xr-x | 실행 파일·디렉터리 (기본) |
| `644` | rw-r--r-- | 일반 문서·설정 파일 |
| `600` | rw------- | 개인 키·비밀 파일 (`~/.ssh/id_rsa`) |
| `700` | rwx------ | 개인 전용 디렉터리 |
| `777` | rwxrwxrwx | 누구나 전권 (거의 항상 위험) |

```bash
chmod 755 script.sh          # 숫자 모드
chmod 600 ~/.ssh/id_rsa      # 개인 키는 반드시 600
```

---

## 심볼릭 모드

대상(`u`ser/`g`roup/`o`ther/`a`ll)에 권한을 더하거나(`+`) 빼거나(`-`) 지정(`=`)한다.

```bash
chmod u+x script.sh          # 소유자에게 실행 권한 추가
chmod g-w file               # 그룹의 쓰기 제거
chmod o=r file               # 기타에게 읽기만
chmod a+x script.sh          # 모두에게 실행
chmod -R 755 public/         # 디렉터리 재귀 적용
```

---

## 소유권 변경

```bash
chown alice file             # 소유자 변경
chown alice:dev file         # 소유자:그룹 동시 변경
chown :dev file              # 그룹만 변경
chown -R www-data:www-data /var/www   # 재귀 (웹 루트에 흔함)
chgrp dev file               # 그룹만 변경 (chown :dev와 동일)
```

> 소유권 변경은 대개 root 권한이 필요하다 (`sudo chown ...`).

---

## umask — 기본 권한 마스크

새로 만든 파일·디렉터리의 **기본 권한에서 빼는** 값이다.

```
파일 기본     666
디렉터리 기본  777
umask         022  (빼기)
─────────────────
파일          644
디렉터리       755
```

```bash
umask                # 현재 마스크 확인 (0022)
umask 077            # 개인 전용: 파일 600, 디렉터리 700
```

---

## 특수 권한

일반 rwx 외에 세 가지 특수 비트가 있다.

| 비트 | 표기 | 효과 |
|------|------|------|
| **SUID** | `chmod u+s` | 실행 시 **파일 소유자 권한**으로 동작 (예: `passwd`) |
| **SGID** | `chmod g+s` | 실행 시 그룹 권한 / 디렉터리는 하위 파일이 그룹 상속 |
| **Sticky** | `chmod +t` | 디렉터리 내 파일을 **소유자만 삭제** 가능 (예: `/tmp`) |

```bash
chmod u+s /usr/bin/somebin    # SUID (rws)
chmod g+s shared/             # SGID 디렉터리
chmod +t /shared/upload       # sticky bit (rwt)

ls -l /usr/bin/passwd         # -rwsr-xr-x → s 확인
ls -ld /tmp                   # drwxrwxrwt → t 확인
```

> SUID 바이너리는 권한 상승 공격의 단골 표적이다. 시스템의 SUID 파일을 주기적으로 점검한다:
> ```bash
> find / -perm -4000 -type f 2>/dev/null
> ```

---

## 배운 점
- 권한은 **소유자 / 그룹 / 기타** 3주체 × **rwx** 로 구성된다
- 숫자 모드(755·644·600)를 외워두면 대부분의 상황에 대응된다
- 개인 키는 `600`, 웹 루트는 소유권을 `www-data`로 — 자주 나오는 패턴
- `umask`가 신규 파일의 기본 권한을 결정한다
- SUID/SGID/sticky는 특수 상황용이며, SUID는 보안 점검 대상
