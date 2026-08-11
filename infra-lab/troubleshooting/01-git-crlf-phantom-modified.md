# 01. Git CRLF — 내용이 같은데 계속 modified로 잡힌다

- **발생**: 2026-08-09, Obsidian 볼트(`GWANG-MIN1/Obsidian`) 작업 중
- **증상 한 줄**: `git status`는 수정됐다고 하는데 `git diff`는 비어 있다

---

## 현상

커밋할 게 없는데 `git status`에 파일 하나가 계속 남아 있었다.

```bash
$ git status -sb
## master...origin/master
 M .obsidian/workspace.json
 M infra-lab/linux-scripts/04-text-processing/README.md
```

그런데 `git diff`를 보면 그 파일의 변경 내역이 없다.

```bash
$ git diff --stat
warning: in the working copy of 'infra-lab/linux-scripts/04-text-processing/README.md',
LF will be replaced by CRLF the next time Git touches it
 .obsidian/workspace.json | 70 +++++++++++++++++++++++-----------------------
 1 file changed, 36 insertions(+), 34 deletions(-)
```

`04-text-processing/README.md`는 **status에는 있는데 diff에는 없다.**

추가로, `git add`를 할 때마다 매번 같은 경고가 쏟아졌다.

```bash
$ git add infra-lab/terraform-lab/03-state/
warning: in the working copy of 'infra-lab/terraform-lab/03-state/README.md',
LF will be replaced by CRLF the next time Git touches it
```

---

## 환경

```
OS       Windows 11 Home 10.0.26200
Git      Git for Windows (Git Bash)
저장소    GWANG-MIN1/Obsidian (Obsidian 볼트)
편집     Obsidian + 에디터/도구가 혼재해서 파일 생성
```

```bash
$ git config --get core.autocrlf
true                         # ← 로컬 저장소 설정

$ git config --global --get core.autocrlf
                             # 미설정 (exit 1)

$ ls -la .gitattributes
ls: cannot access '.gitattributes': No such file or directory
```

---

## 진단

### 1) 내용이 진짜 같은지부터 확인

파일 내용이 실제로 다른데 diff가 안 보이는 건지 의심했다. blob 해시를 직접 비교했다.

```bash
$ F=infra-lab/linux-scripts/04-text-processing/README.md
$ git hash-object -- "$F"
b1a91fde486075d409bf9e9169021db5180fed80
$ git rev-parse "HEAD:$F"
b1a91fde486075d409bf9e9169021db5180fed80
```

**동일하다.** 내용 차이가 아니다.

### 2) 첫 가설 — 인덱스 stat 캐시 문제 → **반증**

"파일 mtime만 바뀌어서 git이 변경으로 오인한 것"이라 추측하고 재현을 시도했다.

```bash
$ touch "$F"
$ git status --porcelain "$F"
                             # 아무것도 안 나온다
```

**반증됐다.** mtime만으로는 재현되지 않는다. git이 내용을 비교해 동일하다고 판정하고 넘어간다.

### 3) 경고 문구를 다시 읽는다

```
LF will be replaced by CRLF the next time Git touches it
```

**"LF가 CRLF로 바뀔 것"** — 즉 그 시점에 **워킹트리 파일이 LF**였다는 뜻이다. Windows에서 `autocrlf=true`면 체크아웃 시 CRLF가 되어야 하는데.

### 4) 실제 줄바꿈 상태 확인 — `git ls-files --eol`

```bash
$ git ls-files --eol infra-lab/linux-scripts/04-text-processing/README.md \
                     infra-lab/docker-labs/01-container-basics/README.md
i/lf    w/crlf  attr/    infra-lab/linux-scripts/04-text-processing/README.md
i/lf    w/lf    attr/    infra-lab/docker-labs/01-container-basics/README.md
```

```
i/  = index(저장소)의 줄바꿈
w/  = working tree(작업 파일)의 줄바꿈
attr/ = .gitattributes 로 지정된 값 (비어 있음 = 지정 없음)
```

**저장소는 전부 LF인데 워킹트리는 파일마다 다르다.** 전수 조사를 했다.

```bash
$ git ls-files --eol | awk '{print $2}' | sort | uniq -c
     33 w/crlf
    159 w/lf
      4 w/none
```

**워킹트리가 CRLF 33개 / LF 159개로 혼재**돼 있었다.

### 5) 혼재 패턴 확인

```bash
$ git ls-files --eol | grep 'w/crlf' | awk '{print $NF}'
infra-lab/README.md
infra-lab/cicd-lab/02-github-actions-basics/README.md
infra-lab/cost-lab/...                (12개 전부)
infra-lab/k8s-manifests/03~07/...
infra-lab/linux-scripts/...           (13개 전부)
...
```

패턴이 보였다. **CRLF인 것은 전부 git이 체크아웃한 파일**이다 —
클론 시점에 받은 것, rebase로 다시 적용된 것(`cost-lab/*`), GitHub에서 편집 후 pull한 것(`cicd-lab/02`).
반대로 **LF인 것은 에디터/도구가 직접 쓰고 그대로 커밋된 파일**이다.

### 6) 재현 — 워킹 파일을 LF로 되돌린다

```bash
$ perl -pi -e 's/\r\n/\n/' "$F"          # 내용은 그대로, 줄바꿈만 LF 로
$ git ls-files --eol "$F"
i/lf    w/lf    attr/    infra-lab/linux-scripts/04-text-processing/README.md

$ git status --porcelain "$F"
 M infra-lab/linux-scripts/04-text-processing/README.md      # ← 재현됨

$ git diff --stat "$F"
warning: in the working copy of '...', LF will be replaced by CRLF the next time Git touches it
                                                             # ← diff 비어있음

$ git hash-object -- "$F"    # b1a91fde486075d409bf9e9169021db5180fed80
$ git rev-parse "HEAD:$F"    # b1a91fde486075d409bf9e9169021db5180fed80  (동일)
```

**정확히 같은 증상이 재현됐다.**

---

## 원인

**`core.autocrlf=true`인데 `.gitattributes`가 없어서, 워킹트리의 줄바꿈이 파일마다 달라진 것.**

```
core.autocrlf = true 의 동작
  체크아웃  index(LF) ──▶ 워킹트리(CRLF)      "Windows 니까 CRLF 로 준다"
  커밋      워킹트리(CRLF) ──▶ index(LF)      "저장은 LF 로 한다"
```

```
git 은 "이 파일을 지금 체크아웃하면 어떤 모습이어야 하는가" 와 실제 워킹 파일을 비교한다.

  워킹트리가 LF  →  체크아웃하면 CRLF 여야 하는데 LF 다  →  status: M
  하지만 커밋용 정규화(CRLF→LF)를 적용하면 blob 이 같다   →  diff: 비어있음
```

이 상태가 만들어진 경로:

| 파일이 생긴 방식 | 워킹트리 줄바꿈 | status |
|---|---|---|
| git이 체크아웃 (clone·pull·rebase·checkout) | **CRLF** | 정상 |
| 에디터/도구가 직접 생성 후 커밋 | **LF** | **팬텀 modified** |

`.gitattributes`가 없으니 저장소 차원의 기준이 없고, **각 클라이언트의 `core.autocrlf` 설정과 파일이 만들어진 경로에 따라 제각각**이 된다.

> 부수적으로 확인된 것: **`.gitignore`도 없다.** 그래서 Obsidian UI 상태 파일인
> `.obsidian/workspace.json`이 추적되고 있고, 창을 옮기거나 파일을 열기만 해도 계속 `M`으로 남는다.
> 커밋할 때마다 의미 없는 diff가 섞인다.

---

## 해결

### ① `.gitattributes`로 저장소 기준을 고정한다

```gitattributes
# 저장소에는 LF 로 저장한다. 워킹트리 줄바꿈은 OS 가 알아서.
* text=auto eol=lf

# 바이너리는 건드리지 않는다
*.png  binary
*.jpg  binary
*.pdf  binary
*.zip  binary
```

> `text=auto` = git이 텍스트/바이너리를 판별해 텍스트만 정규화
> `eol=lf` = 워킹트리도 LF로 통일 (팀 전체가 같은 상태가 된다)

이것만으로 **팬텀 modified 는 즉시 해소된다.** `eol=lf`가 되면 git이 워킹트리에 기대하는 값이
LF가 되므로, LF 파일이 더 이상 "달라야 하는데 같다" 상태가 아니게 된다.

```bash
$ git ls-files --eol infra-lab/linux-scripts/04-text-processing/README.md
i/lf    w/crlf  attr/text=auto eol=lf   ...      # 속성이 붙었다

$ git status --porcelain
                                                 # 팬텀 modified 사라짐
```

### ② ⚠️ `git add --renormalize .` 는 이 상황에서 아무것도 안 한다

인터넷 문서 대부분이 다음 단계로 이걸 안내한다. 실제로 실행해봤다.

```bash
$ git add --renormalize .
$ git diff --cached --name-only | wc -l
0                                # ← 아무것도 안 잡힌다
```

**`--renormalize`는 "워킹트리를 clean 필터에 통과시켜 인덱스를 갱신"하는 명령이다.**
그런데 이 저장소는 **인덱스가 이미 전부 LF**(`i/lf`)였다. CRLF 워킹 파일을 clean 필터에
통과시키면 LF가 나오고, 그건 인덱스에 이미 있는 값과 같다. 그래서 변경이 0이다.

```
--renormalize 가 효과 있는 경우:  인덱스에 CRLF 가 섞여 들어간 저장소 (i/crlf)
이 저장소의 경우:                 인덱스는 이미 LF, 문제는 워킹트리 → 효과 없음
```

### ③ 워킹트리를 실제로 통일하려면 재체크아웃이 필요하다

`.gitattributes`는 **체크아웃 시점에** 적용된다. 이미 워킹트리에 있는 파일은
다시 체크아웃되기 전까지 CRLF로 남는다.

```bash
# 반드시 커밋되지 않은 변경이 없는 상태에서 (git status 가 비어야 한다)
git status --porcelain           # ← 먼저 확인
git rm --cached -r -q .          # 인덱스에서 내렸다가
git reset --hard                 # 새 속성으로 전부 다시 체크아웃
```

> ⚠️ 트리가 깨끗할 때만 한다. 커밋 안 된 변경이 있으면 `reset --hard`가 그걸 날린다.
> `.gitignore`된 파일(추적 안 되는 파일)은 영향받지 않는다.

### ④ 검증

```bash
$ git ls-files --eol | awk '{print $2}' | sort | uniq -c
    195 w/lf                     # CRLF 가 사라졌다
      4 w/none                   # 바이너리

$ git status --porcelain
                                 # 팬텀 modified 없음
```

### ④ `.gitignore` 추가

```gitignore
# Obsidian UI 상태 — 창 배치·열린 탭. 내용이 아니다.
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# 플러그인 캐시
.obsidian/plugins/*/data.json

# OS
.DS_Store
Thumbs.db
```

```bash
# 이미 추적 중이면 추적만 해제한다 (파일은 로컬에 남는다)
git rm --cached .obsidian/workspace.json
```

> ⚠️ `.obsidian/` 전체를 무시하면 **플러그인·테마·핫키 설정까지 동기화가 끊긴다.**
> 볼트를 여러 기기에서 쓴다면 `workspace.json`만 골라서 제외한다.

---

## 재발 방지

| 조치 | 효과 |
|---|---|
| **`.gitattributes`를 저장소 생성 시 함께 커밋** | 클라이언트 설정과 무관하게 기준이 하나가 된다 |
| `.gitignore`를 저장소 생성 시 함께 커밋 | 상태 파일이 애초에 추적되지 않는다 |
| 새 저장소 체크리스트에 둘 다 포함 | 매번 기억하지 않아도 된다 |
| 팀 저장소면 `core.autocrlf`에 의존하지 않는다 | 사람마다 설정이 다르다 |

```bash
# 새 저장소를 만들 때 첫 커밋에 함께
printf '* text=auto eol=lf\n' > .gitattributes
git add .gitattributes .gitignore
git commit -m "chore: 줄바꿈·무시 규칙 설정"
```

---

## 배운 점

- **`git status`와 `git diff`가 어긋나면 내용이 아니라 줄바꿈·필터를 의심한다**
- ⭐ **`git ls-files --eol`이 결정적 진단 명령** — `i/`(저장소) `w/`(워킹트리) `attr/`(속성)를 한 번에 보여준다
- 경고 문구 **"LF will be replaced by CRLF"** 는 **워킹 파일이 지금 LF**라는 뜻이다 (읽는 방향을 헷갈리기 쉽다)
- `core.autocrlf`는 **클라이언트 설정**이라 저장소 차원의 기준이 되지 못한다 → **`.gitattributes`가 답**
- `.gitattributes`에 `eol=lf`를 넣는 것만으로 **팬텀 modified 는 즉시 해소**된다
- ⭐ **`git add --renormalize .` 가 항상 답은 아니다** — 인덱스가 이미 LF면 아무것도 안 잡는다
  (이 명령은 워킹트리가 아니라 **인덱스**를 갱신하는 명령이다)
- **`.gitattributes`는 체크아웃 시점에 적용**된다 → 워킹트리를 통일하려면 `git rm --cached -r . && git reset --hard`
- 워킹트리 줄바꿈이 혼재하는 이유: **git이 체크아웃한 파일(CRLF) vs 도구가 직접 쓴 파일(LF)**
- **첫 가설(stat 캐시)을 `touch`로 반증**한 게 방향을 바꿨다 — 추측을 검증하지 않았으면 엉뚱한 걸 고쳤을 것
- 재현에 성공하니(`perl -pi -e 's/\r\n/\n/'`) 원인이 확정됐다 — **재현 못 하면 고쳤는지도 모른다**
- 부수 발견: **`.gitignore`가 없어 Obsidian UI 상태 파일이 추적되고 있었다**
- 볼트에서 `.obsidian/` 전체를 무시하면 플러그인·테마 동기화가 끊긴다 — **`workspace.json`만** 제외한다
