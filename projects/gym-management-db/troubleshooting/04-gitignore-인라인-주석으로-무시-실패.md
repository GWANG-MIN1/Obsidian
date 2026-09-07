# 04. .gitignore에 적었는데 파일이 무시되지 않았다

- **발생**: 2026-09-05, gym-management-db
- **증상 한 줄**: `terraform.tfvars`를 `.gitignore`에 넣었는데 `git check-ignore`가 잡지 못했다

---

## 현상

DB 비밀번호가 들어가는 `terraform/terraform.tfvars`를 무시하도록 적어 뒀다고 생각했다.

```gitignore
terraform/terraform.tfvars        # 패스워드 포함 — 절대 커밋 금지
```

그런데 실제로는 무시되지 않았다.

```bash
$ git check-ignore -v terraform/terraform.tfvars
$ echo $?
1        # 매칭되는 규칙 없음
```

## 환경

- git 2.x (macOS, Command Line Tools)

## 진단

`.gitignore`는 `#`으로 **시작하는 줄**만 주석으로 본다.
줄 중간의 `#`은 주석이 아니라 **패턴의 일부**다.

그래서 위 규칙은 이런 이름의 파일을 찾고 있었다.

```
terraform/terraform.tfvars        # 패스워드 포함 — 절대 커밋 금지
```

그런 파일은 존재하지 않으니 아무것도 무시되지 않는다.
후행 공백까지 패턴에 포함되므로, 주석을 지워도 공백이 남으면 같은 문제가 난다.

## 원인

**패턴 뒤에 같은 줄로 주석을 달았다.**

`.gitignore`에는 인라인 주석 문법이 없다.

## 해결

주석을 별도 줄로 옮겼다.

```gitignore
# ↓ DB 패스워드 포함 — 절대 커밋 금지
# (주석을 같은 줄에 쓰면 패턴의 일부로 인식되어 무시되지 않으므로 반드시 별도 줄에 작성)
terraform/terraform.tfvars
```

확인:

```bash
$ git check-ignore -v terraform/terraform.tfvars
.gitignore:7:terraform/terraform.tfvars	terraform/terraform.tfvars
```

커밋 `5ed3310`.

## 다행이었던 것

**비밀번호 파일이 실제로 커밋돼 있지는 않았다.** `terraform.tfvars`를 만든 적이 없어서
무시 규칙이 동작하지 않는다는 사실이 드러날 일이 없었을 뿐이다.
규칙이 깨진 채로 파일을 만들었다면 그대로 올라갔다.

## 재발 방지

- `.gitignore`를 고치면 **`git check-ignore -v`로 확인한다.** 규칙을 적는 것과 동작하는 것은 다르다.
- 민감한 파일은 규칙만 믿지 말고, 커밋 전에 `git status`에 뜨지 않는지 눈으로 확인.

## 배운 점

**설정 파일마다 주석 문법이 다르다.** `.gitignore`, `.dockerignore` 는 줄 시작 `#`만 주석이다.
"적어 뒀으니 됐다"와 "동작한다"는 다르고, 확인 명령이 있으면 그걸 써야 한다.

보안 설정은 **조용히 실패한다.** 오류도 경고도 없이 그냥 무시되지 않을 뿐이다.
