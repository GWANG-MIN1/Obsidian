# Linux 명령어 레퍼런스

## 탐색 & 파일 조작
```
pwd                                  # 현재 경로
ls -alh                              # 숨김 파일 포함, 상세, 사람이 읽는 크기
cd -                                 # 직전 디렉터리로 이동
tree -L 2                            # 2단계 깊이 트리 출력
cp -r 원본 대상                       # 디렉터리 재귀 복사
mv 원본 대상                          # 이동/이름 변경
rm -rf 디렉터리                       # 재귀 강제 삭제 (주의)
mkdir -p a/b/c                       # 중간 경로까지 한 번에 생성
ln -s /원본 /링크                     # 심볼릭 링크 생성
```

## 파일 찾기 & 내용 보기
```
find / -name "*.conf" 2>/dev/null    # 이름으로 검색 (에러 무시)
find . -type f -mtime -1             # 최근 1일 내 수정된 파일
find . -size +100M                   # 100MB 초과 파일
locate nginx.conf                    # DB 기반 빠른 검색 (updatedb 필요)
cat / less / tail -f 파일             # 전체 / 페이지 / 실시간 확인
head -n 20 파일                       # 앞 20줄
wc -l 파일                            # 줄 수
```

## 권한 & 소유권
```
chmod 755 파일                        # rwxr-xr-x
chmod u+x,g-w 파일                    # 심볼릭 모드
chown user:group 파일                 # 소유자·그룹 변경
chown -R user 디렉터리                 # 재귀 변경
umask 022                            # 기본 권한 마스크 (파일 644, 디렉터리 755)
chmod u+s 파일                        # SUID
chmod +t 디렉터리                      # sticky bit
```

## 프로세스 관리
```
ps aux                               # 전체 프로세스 (BSD 스타일)
ps -ef                               # 전체 프로세스 (표준 스타일)
top / htop                           # 실시간 자원 모니터
kill -TERM PID                       # 정상 종료 요청 (SIGTERM)
kill -9 PID                          # 강제 종료 (SIGKILL)
pkill -f "패턴"                       # 명령줄 패턴으로 종료
jobs / fg %1 / bg %1                 # 잡 목록 / 포그라운드 / 백그라운드
nohup 명령 &                          # 로그아웃 후에도 유지
nice -n 10 명령                       # 낮은 우선순위로 실행
renice -n 5 -p PID                    # 실행 중 우선순위 변경
```

## 텍스트 처리 (파이프 & 필터)
```
grep -rin "패턴" .                    # 재귀·대소문자무시·줄번호 검색
grep -v "제외패턴" 파일               # 매칭 제외
sed 's/old/new/g' 파일                # 치환 (출력)
sed -i 's/old/new/g' 파일             # 파일 직접 치환
awk '{print $1, $3}' 파일             # 1·3번째 필드 출력
awk -F: '{print $1}' /etc/passwd      # 구분자 지정
sort 파일 | uniq -c | sort -rn        # 빈도수 집계 후 내림차순
cut -d: -f1 /etc/passwd               # 필드 추출
tr 'a-z' 'A-Z' < 파일                 # 문자 치환
xargs -I{} 명령 {}                    # 표준입력을 인자로 전달
```

## 리다이렉션 & 파이프
```
명령 > 파일                           # stdout 덮어쓰기
명령 >> 파일                          # stdout 이어쓰기
명령 2> error.log                     # stderr만 저장
명령 > out.log 2>&1                   # stdout+stderr 함께
명령 &> all.log                       # (bash) stdout+stderr 함께
명령1 | 명령2                          # 파이프 연결
명령 | tee 파일                        # 화면 출력 + 파일 저장
```

## 사용자 & 그룹
```
whoami / id                          # 현재 사용자 / UID·GID·그룹
useradd -m -s /bin/bash user          # 홈 생성 + 셸 지정
passwd user                          # 비밀번호 설정
usermod -aG docker user               # 보조 그룹 추가 (-a 필수)
userdel -r user                       # 홈 포함 삭제
groupadd dev                         # 그룹 생성
su - user                            # 사용자 전환
sudo -i                              # root 셸
visudo                               # sudoers 안전 편집
```

## 패키지 관리
```
# Debian/Ubuntu (APT)
apt update && apt upgrade            # 목록 갱신 + 업그레이드
apt install -y 패키지                 # 설치
apt remove / apt purge 패키지         # 제거 / 설정까지 제거
apt search 키워드                     # 검색
dpkg -l | grep 패키지                 # 설치 목록 확인

# RHEL/CentOS/Fedora (DNF/YUM)
dnf install -y 패키지                  # 설치
dnf remove 패키지                      # 제거
dnf search 키워드                      # 검색
rpm -qa | grep 패키지                  # 설치 목록 확인
```

## systemd 서비스
```
systemctl status 서비스               # 상태 확인
systemctl start / stop / restart 서비스
systemctl enable --now 서비스         # 부팅 시 자동 시작 + 즉시 시작
systemctl disable 서비스              # 자동 시작 해제
systemctl daemon-reload              # 유닛 파일 변경 반영
systemctl list-units --type=service  # 서비스 목록
journalctl -u 서비스 -f              # 서비스 로그 실시간
journalctl --since "1 hour ago"      # 시간 필터
systemctl list-timers                # 타이머(스케줄) 목록
```

## 네트워킹
```
ip addr                              # IP 주소 확인 (ifconfig 대체)
ip route                             # 라우팅 테이블
ss -tulpn                            # 리스닝 포트 + 프로세스
ping -c 4 호스트                      # 연결 확인
curl -I https://example.com          # HTTP 헤더만
dig example.com / nslookup example.com  # DNS 조회
ssh -i key.pem user@host             # 키 기반 SSH 접속
scp 파일 user@host:/경로              # 원격 파일 복사
ufw allow 22/tcp                     # 방화벽 포트 허용 (Ubuntu)
firewall-cmd --add-port=80/tcp --permanent  # (RHEL)
```

## 디스크 & 모니터링
```
df -h                                # 파일시스템 사용량
du -sh *                             # 현재 위치 항목별 용량
du -sh * | sort -rh | head           # 용량 큰 순 정렬
free -h                              # 메모리 사용량
vmstat 1                             # CPU·메모리·IO 1초 간격
iostat -x 1                          # 디스크 IO 상세
uptime                               # 부하 평균(load average)
lsof -i :80                          # 특정 포트 사용 프로세스
dmesg -T | tail                      # 커널 메시지 (하드웨어·OOM)
```

## 압축 & 아카이브
```
tar -czvf archive.tar.gz 디렉터리     # gzip 압축
tar -xzvf archive.tar.gz             # gzip 해제
tar -tzvf archive.tar.gz             # 내용 목록만
zip -r archive.zip 디렉터리           # zip 압축
unzip archive.zip                    # zip 해제
gzip / gunzip 파일                    # 단일 파일 압축/해제
```
