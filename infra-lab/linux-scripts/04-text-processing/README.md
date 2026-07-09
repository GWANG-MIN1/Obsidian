# 04 텍스트 처리

리눅스의 힘은 **작은 도구를 파이프로 조합**하는 데서 나온다 (Unix 철학: "한 가지 일을 잘하는 도구"). 로그 분석·설정 추출·데이터 가공의 대부분이 `grep`·`sed`·`awk` + 파이프로 해결된다.

---

## 표준 스트림 & 리다이렉션

모든 프로세스는 3개의 기본 스트림을 갖는다.

| 스트림 | 번호 | 기본 대상 |
|--------|------|-----------|
| stdin | 0 | 키보드 |
| stdout | 1 | 화면 |
| stderr | 2 | 화면 |

```bash
명령 > out.txt          # stdout을 파일로 (덮어쓰기)
명령 >> out.txt         # 이어쓰기
명령 2> err.txt         # stderr만 파일로
명령 > out.txt 2>&1     # stdout+stderr 함께 (순서 중요)
명령 &> all.txt         # (bash) stdout+stderr 함께 (축약형)
명령 < in.txt           # 파일을 stdin으로
명령 2>/dev/null        # 에러 버리기
```

> `2>&1`은 "stderr(2)를 stdout(1)이 가는 곳으로 보내라"는 뜻. `> out 2>&1` 순서를 지켜야 둘 다 파일로 간다.

---

## 파이프 & tee

`|`는 앞 명령의 stdout을 뒤 명령의 stdin으로 연결한다.

```bash
ps aux | grep nginx | wc -l          # nginx 프로세스 개수
cat access.log | grep 404 | tee 404.log | wc -l   # 저장 + 개수
```

`tee`는 스트림을 **화면과 파일로 동시에** 흘려보낸다.

---

## grep — 패턴 검색

```bash
grep "error" app.log              # 매칭 줄 출력
grep -i "error" app.log           # 대소문자 무시
grep -r "TODO" ./src              # 디렉터리 재귀 검색
grep -n "func" main.go            # 줄 번호 표시
grep -v "DEBUG" app.log           # 매칭 제외 (반전)
grep -c "200" access.log          # 매칭 줄 수
grep -E "error|warn|fail" app.log # 확장 정규식 (OR)
grep -A3 -B1 "Exception" app.log  # 매칭 앞1·뒤3줄 컨텍스트
```

자주 쓰는 조합: `grep -rin "패턴" .` (재귀·무시·줄번호).

---

## sed — 스트림 편집

```bash
sed 's/old/new/' file             # 각 줄 첫 매칭만 치환
sed 's/old/new/g' file            # 줄 전체 치환 (global)
sed -i 's/old/new/g' file         # 파일 직접 수정 (in-place)
sed -i.bak 's/old/new/g' file     # 백업(.bak) 남기고 수정
sed -n '10,20p' file              # 10~20번째 줄만 출력
sed '/^#/d' config                # 주석(#) 줄 삭제
sed '/^$/d' file                  # 빈 줄 삭제
```

> `sed -i`는 되돌릴 수 없다. 중요한 파일은 `-i.bak`으로 백업을 남긴다.

---

## awk — 필드 기반 처리

`awk`는 줄을 필드로 쪼개 처리하는 미니 프로그래밍 언어다. 기본 구분자는 공백.

```bash
awk '{print $1}' access.log             # 첫 번째 필드 (IP)
awk '{print $1, $7}' access.log         # 여러 필드
awk -F: '{print $1}' /etc/passwd        # 구분자를 :로
awk '{print NR, $0}' file               # 줄번호(NR) + 전체줄($0)
awk '$3 > 100' data.txt                 # 3번 필드가 100 초과인 줄
awk '/error/ {count++} END {print count}' app.log  # 조건 카운트
awk '{sum+=$1} END {print sum}' nums    # 합계
```

| 변수 | 의미 |
|------|------|
| `$0` | 전체 줄 |
| `$1, $2...` | 각 필드 |
| `NR` | 현재 줄 번호 |
| `NF` | 현재 줄의 필드 개수 |
| `-F` | 필드 구분자 지정 |

---

## sort · uniq · cut · wc

```bash
sort file                    # 사전순 정렬
sort -n file                 # 숫자순
sort -rn file                # 숫자 역순
sort -k2 file                # 2번째 필드 기준
sort file | uniq             # 중복 제거 (정렬 필수)
sort file | uniq -c          # 중복 개수 세기
cut -d: -f1 /etc/passwd      # : 구분, 1번 필드
cut -c1-10 file              # 1~10번째 문자
wc -l file                   # 줄 수
wc -w file                   # 단어 수
tr 'a-z' 'A-Z' < file        # 소문자→대문자
```

---

## 실전 조합 예제

```bash
# access.log에서 요청 많은 IP Top 10
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 404 응답 시각별 집계
grep " 404 " access.log | awk '{print $4}' | cut -c14-15 | sort | uniq -c

# 설정 파일에서 주석·빈 줄 제외한 실제 설정만
grep -vE '^\s*#|^\s*$' /etc/ssh/sshd_config

# 디스크 큰 디렉터리 Top 10
du -sh */ | sort -rh | head -10
```

---

## 배운 점
- **파이프로 작은 도구를 조합**하는 것이 리눅스 텍스트 처리의 핵심
- `grep`(검색) → `sed`(치환) → `awk`(필드 처리)가 3대 도구
- 리다이렉션 `>`·`2>&1`로 출력·에러를 원하는 곳으로 보낸다
- `sort | uniq -c | sort -rn`은 **빈도 집계의 정석** — 로그 분석에 필수
- `sed -i`는 백업(`-i.bak`) 습관을 들이자
