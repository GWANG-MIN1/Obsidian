# 06 패키지 관리

소프트웨어를 소스에서 일일이 컴파일하는 대신, **패키지 매니저**가 설치·의존성·업데이트·삭제를 자동으로 관리한다. 배포판 계열마다 도구가 다르다.

---

## 배포판 계열별 도구

| 계열 | 배포판 | 저수준 | 고수준(권장) |
|------|--------|--------|--------------|
| **Debian** | Ubuntu, Debian | `dpkg` | `apt` |
| **RHEL** | RHEL, CentOS, Fedora, Rocky | `rpm` | `dnf` (구 `yum`) |
| **Arch** | Arch, Manjaro | — | `pacman` |
| **SUSE** | openSUSE | `rpm` | `zypper` |

> 저수준 도구(`dpkg`/`rpm`)는 개별 패키지만 다루고 **의존성을 자동 해결하지 못한다**. 고수준 도구(`apt`/`dnf`)가 저장소에서 의존성까지 끌어온다.

---

## APT (Debian/Ubuntu)

```bash
apt update                    # 패키지 목록 갱신 (설치 전 필수)
apt upgrade                   # 설치된 패키지 업그레이드
apt full-upgrade              # 의존성 변경 포함 업그레이드
apt install -y nginx          # 설치 (-y: 확인 자동 yes)
apt remove nginx              # 제거 (설정 파일 유지)
apt purge nginx               # 설정까지 완전 제거
apt autoremove                # 불필요해진 의존성 정리
apt search keyword            # 검색
apt show nginx                # 패키지 상세 정보
apt list --installed          # 설치 목록
```

### dpkg (저수준)

```bash
dpkg -i package.deb           # .deb 파일 직접 설치
dpkg -l | grep nginx          # 설치 확인
dpkg -L nginx                 # 패키지가 설치한 파일 목록
dpkg -S /usr/sbin/nginx       # 파일이 어느 패키지 것인지
```

---

## DNF / YUM (RHEL/Fedora)

```bash
dnf check-update              # 업데이트 확인
dnf install -y httpd          # 설치
dnf update httpd              # 특정 패키지 업데이트
dnf remove httpd              # 제거
dnf search keyword            # 검색
dnf info httpd                # 상세 정보
dnf list installed            # 설치 목록
dnf grouplist                 # 패키지 그룹
dnf history                   # 트랜잭션 이력 (롤백 가능)
```

### rpm (저수준)

```bash
rpm -ivh package.rpm          # 설치
rpm -qa | grep httpd          # 설치 확인
rpm -ql httpd                 # 설치 파일 목록
rpm -qf /usr/sbin/httpd       # 파일의 소속 패키지
```

---

## 저장소(Repository)

패키지는 원격 저장소에서 받아온다. 저장소 목록과 GPG 키가 신뢰의 기반이다.

```bash
# APT 저장소
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/          # 추가 저장소
add-apt-repository ppa:some/ppa       # PPA 추가 (Ubuntu)

# DNF 저장소
cat /etc/yum.repos.d/*.repo
dnf config-manager --add-repo <URL>
```

> 서드파티 저장소를 추가할 때는 **GPG 키를 함께 등록**해 패키지 위·변조를 검증한다. 키 없는 저장소는 공급망 공격의 통로가 될 수 있다.

---

## 소스에서 빌드

패키지가 없거나 최신 버전이 필요할 때 직접 컴파일한다.

```bash
sudo apt install build-essential      # gcc/make 등 빌드 도구
./configure --prefix=/usr/local       # 빌드 설정
make                                  # 컴파일
sudo make install                     # 설치
```

> 소스 설치는 패키지 매니저가 추적하지 못해 업데이트·삭제가 번거롭다. 가능하면 패키지를 쓰고, 소스 설치는 `/usr/local`에 격리한다.

---

## 배포판 독립적 패키징

| 방식 | 특징 |
|------|------|
| **Snap** | Canonical, 샌드박스·자동 업데이트 (Ubuntu 기본) |
| **Flatpak** | 데스크톱 앱 중심, 배포판 독립 |
| **AppImage** | 단일 실행 파일, 설치 불필요 |
| **컨테이너 이미지** | 의존성까지 통째로 격리 — 서버 배포의 사실상 표준 |

> 서버 애플리케이션 배포에서는 결국 **컨테이너 이미지**가 "궁극의 패키지"가 된다. OS 패키지 매니저로 런타임(예: Docker)을 깔고, 앱은 이미지로 배포하는 조합이 흔하다.

---

## 배운 점
- 계열마다 도구가 다르다: **Debian=apt/dpkg, RHEL=dnf/rpm**
- 고수준 도구(apt·dnf)가 **의존성을 자동 해결**, 저수준(dpkg·rpm)은 개별 패키지만
- `apt update`(목록 갱신) ≠ `apt upgrade`(실제 업그레이드) — 순서 중요
- 서드파티 저장소는 **GPG 키로 신뢰를 검증**
- 앱 배포의 종착점은 컨테이너 이미지 — 다음 단계인 Docker로 이어진다
