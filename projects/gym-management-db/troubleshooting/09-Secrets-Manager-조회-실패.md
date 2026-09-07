# 09. Secrets Manager에서 DB 접속 정보를 못 읽었다

- **발생**: 2026-05-13, gym-management-db
- **증상 한 줄**: EC2에서 시크릿 조회가 실패해 API가 기동하지 못했다

> 이 노트는 **커밋 기록에서 되짚은 것**이다. 당시 로그와 오류 메시지를 남기지 않아
> 확인된 사실(코드 변경)과 되살려야 할 부분(진단 과정)을 나눠 적는다.

---

## 현상

> **되살려 채울 것:** 실제로 본 오류 메시지. `docker logs gym-api`에 무엇이 찍혔나.
> (`ResourceNotFoundException`? `ParamValidationError`? 아니면 `SecretId=None`?)

배포는 끝났는데 API가 뜨지 않았다 → [트러블 10](10-배포-후-헬스체크가-안-떴다.md)

## 환경

- EC2 (Amazon Linux) + Docker, ap-northeast-2
- AWS Secrets Manager, EC2 인스턴스 IAM 롤로 접근
- 시크릿 이름: `gym-mgmt-dev/db-credentials`

## 진단

커밋 `a4f560d`가 바꾼 것은 두 곳이다.

**API 쪽** — 시크릿을 ARN이 아니라 **이름**으로 조회하도록.

```diff
-    secret_arn = os.getenv("SECRET_ARN")
+    secret_name = os.getenv("SECRET_NAME", "gym-mgmt-dev/db-credentials")
-        client.get_secret_value(SecretId=secret_arn)["SecretString"]
+        client.get_secret_value(SecretId=secret_name)["SecretString"]
```

**CD 쪽** — 컨테이너에 ARN을 넣어 주던 환경변수를 제거.

```diff
-              -e SECRET_ARN="${{ secrets.SECRET_ARN }}" \
```

여기서 읽을 수 있는 것: **ARN을 GitHub Secret으로 주입하는 구조였고, 그 값이 컨테이너까지 제대로 전달되지 않았다.**
`terraform/outputs.tf`에도 그 흔적이 남아 있다.

```hcl
output "db_secret_arn" {
  description = "Secrets Manager ARN — SECRET_ARN GitHub Secret에 등록"
}
```

즉 `terraform apply` → 출력된 ARN을 **손으로** GitHub Secret에 등록 → CD가 컨테이너에 주입,
이라는 수동 단계가 중간에 있었다.

> **되살려 채울 것:** GitHub Secret에 등록을 안 했던 건가, 값이 틀렸던 건가, 아니면
> `envs:` 목록에 없어서 SSH 세션까지 전달되지 않았던 건가.

## 원인

> **되살려 채울 것.** 확실한 것은 "ARN을 외부에서 주입하는 방식이 동작하지 않았고,
> 이름으로 직접 조회하도록 바꿔 해결했다"까지다.

## 해결

시크릿을 **이름으로 조회**하고, 그 이름을 코드의 기본값으로 둔다.

```python
secret_name = os.getenv("SECRET_NAME", "gym-mgmt-dev/db-credentials")
```

이름은 Terraform이 `"${local.name}/db-credentials"`로 만들기 때문에 예측 가능하다.
ARN은 계정 ID와 무작위 접미사가 붙어서 미리 알 수 없고, 그래서 사람이 옮겨 적는 단계가 필요했다.

배포마다 사람이 값을 옮기던 단계 하나가 사라졌다.

## 재발 방지

- **예측 가능한 식별자를 쓴다.** 이름은 규칙으로 만들 수 있고 ARN은 아니다.
- 사람이 콘솔에서 값을 복사해 다른 곳에 붙여 넣는 단계는 재현되지 않는다. 언젠가 빠뜨린다.
- 컨테이너가 기동에 실패하면 그 이유가 로그로 보여야 한다 → [트러블 10](10-배포-후-헬스체크가-안-떴다.md)

## 배운 점

> **되살려 채울 것.**
>
> 힌트: 배포 파이프라인에서 "손으로 한 번만 해 두면 되는 설정"이 어떤 문제를 만드는가.
