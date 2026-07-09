# 09 셸 스크립팅 (Bash)

반복 작업을 자동화하고 여러 명령을 하나의 절차로 묶는 것이 셸 스크립트다. 배포·백업·헬스체크·CI 파이프라인의 상당수가 Bash로 짜여 있다.

---

## 첫 스크립트

```bash
#!/bin/bash
# hello.sh
echo "Hello, $USER!"
echo "오늘은 $(date +%F)"
```

```bash
chmod +x hello.sh     # 실행 권한 부여
./hello.sh            # 실행
bash hello.sh         # 셸 지정 실행
```

> 첫 줄 `#!/bin/bash`(셔뱅)은 이 파일을 어떤 인터프리터로 실행할지 지정한다.

---

## 안전 옵션 (거의 필수)

프로덕션 스크립트는 이 세 줄로 시작하는 게 좋다.

```bash
#!/bin/bash
set -euo pipefail
```

| 옵션 | 효과 |
|------|------|
| `set -e` | 명령 하나라도 실패하면 즉시 중단 |
| `set -u` | 정의 안 된 변수 사용 시 에러 |
| `set -o pipefail` | 파이프 중간 명령 실패도 감지 |
| `set -x` | 실행 명령을 출력 (디버깅) |

> `set -e` 없이는 에러가 나도 스크립트가 계속 진행해 위험한 상태를 만들 수 있다. 백업·배포 스크립트에 특히 중요.

---

## 변수

```bash
name="Alice"          # = 좌우에 공백 없음 (중요)
echo "$name"          # 큰따옴표: 변수 확장
echo '$name'          # 작은따옴표: 그대로 출력
echo "${name}_suffix" # 중괄호로 경계 명확히

count=$(ls | wc -l)   # 명령 결과를 변수에 (커맨드 치환)
readonly PI=3.14      # 상수
export PATH="$PATH:/opt/bin"   # 환경변수로 내보내기
```

### 특수 변수

| 변수 | 의미 |
|------|------|
| `$0` | 스크립트 이름 |
| `$1, $2...` | 위치 인자 |
| `$#` | 인자 개수 |
| `$@` | 모든 인자 (개별) |
| `$?` | 직전 명령 종료 코드 (0=성공) |
| `$$` | 현재 PID |

---

## 조건문

```bash
if [ "$count" -gt 10 ]; then
  echo "많음"
elif [ "$count" -eq 0 ]; then
  echo "없음"
else
  echo "보통"
fi
```

### 비교 연산자

| 숫자 | 문자열 | 파일 |
|------|--------|------|
| `-eq` 같음 | `=` 같음 | `-f` 파일 존재 |
| `-ne` 다름 | `!=` 다름 | `-d` 디렉터리 존재 |
| `-gt` 초과 | `-z` 빈 문자열 | `-e` 존재 |
| `-lt` 미만 | `-n` 비지 않음 | `-r`/`-w`/`-x` 읽기/쓰기/실행 |

```bash
# 파일 존재 확인 후 실행
if [ -f /etc/nginx/nginx.conf ]; then
  echo "설정 존재"
fi

# [[ ]]는 Bash 확장 (정규식·&&·|| 가능, 권장)
if [[ "$name" =~ ^A ]]; then echo "A로 시작"; fi
```

---

## 반복문

```bash
# for
for i in 1 2 3; do echo "$i"; done
for f in *.log; do echo "$f"; done
for i in {1..5}; do echo "$i"; done

# C 스타일
for ((i=0; i<5; i++)); do echo "$i"; done

# while
count=0
while [ "$count" -lt 3 ]; do
  echo "$count"
  ((count++))
done

# 파일 한 줄씩 읽기
while IFS= read -r line; do
  echo "처리: $line"
done < input.txt
```

---

## 함수

```bash
log() {
  echo "[$(date +%T)] $1"     # $1 = 첫 인자
}

deploy() {
  local env="$1"              # local: 함수 지역 변수
  log "배포 시작: $env"
  # ...
  return 0                    # 종료 코드
}

log "시작"
deploy "production"
```

---

## 종료 코드 & 에러 처리

```bash
command || { echo "실패!"; exit 1; }   # 실패 시 처리
command && echo "성공"                 # 성공 시만

# trap: 스크립트 종료·중단 시 정리 작업
cleanup() { rm -f /tmp/lock; }
trap cleanup EXIT

if ! systemctl is-active --quiet nginx; then
  echo "nginx 다운!" >&2       # 에러는 stderr로
  exit 1
fi
```

> 종료 코드 `0`=성공, 그 외=실패. CI·systemd·다른 스크립트가 이 값으로 성패를 판단하므로 정확히 반환해야 한다.

---

## cron — 스케줄 자동화

```bash
crontab -e            # 현재 사용자 크론 편집
crontab -l            # 목록 확인
```

크론 표현식 (5개 필드):

```
┌ 분 (0-59)
│ ┌ 시 (0-23)
│ │ ┌ 일 (1-31)
│ │ │ ┌ 월 (1-12)
│ │ │ │ ┌ 요일 (0-7, 0·7=일)
│ │ │ │ │
* * * * *  실행할_명령

0 3 * * *        매일 새벽 3시
*/5 * * * *      5분마다
0 0 * * 0        매주 일요일 자정
0 9 1 * *        매월 1일 오전 9시
```

```bash
# 매일 새벽 2시 백업, 로그 남기기
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

> cron 환경은 최소한의 PATH만 갖는다. 스크립트 안에서 **절대 경로**를 쓰거나 PATH를 명시해야 "수동으론 되는데 cron에선 안 되는" 문제를 피한다.

---

## 실전 예제: 헬스체크 스크립트

```bash
#!/bin/bash
set -euo pipefail

URL="http://localhost:8080/health"
SERVICE="myapp"

if curl -fsS --max-time 5 "$URL" > /dev/null; then
  echo "[$(date +%T)] OK"
else
  echo "[$(date +%T)] DOWN → 재시작" >&2
  systemctl restart "$SERVICE"
fi
```

---

## 배운 점
- 셔뱅 `#!/bin/bash`으로 시작, `set -euo pipefail`로 안전하게
- 변수 대입은 `=` 양쪽 공백 없이, 사용은 항상 `"$var"` (따옴표 습관)
- `$?`(종료 코드)·`$1`(인자) 등 특수 변수와 `[[ ]]` 조건이 핵심
- 함수의 `local`, `trap`으로 정리 작업, 에러는 stderr(`>&2`)로
- cron으로 스케줄 자동화 — **절대 경로**를 써야 함정을 피한다
