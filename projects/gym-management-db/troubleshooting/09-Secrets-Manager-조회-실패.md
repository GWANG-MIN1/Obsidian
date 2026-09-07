# 09. Secrets Manager에서 DB 접속 정보를 못 읽었다

- **발생**: 2026-05-13, gym-management-db
- **증상 한 줄**: EC2에서 시크릿 조회가 실패해 API가 기동하지 못했다

> 당시 로그를 남기지 않아 **코드와 워크플로우에서 확인되는 것**으로 복원한 노트다.
> 확인된 사실과 확정하지 못한 부분을 나눠 적는다.

---

## 현상

Secrets Manager를 붙인 직후(`0940161`) 배포에서 API가 뜨지 않았다.
`/health`가 응답하지 않아 CD가 헬스체크에서 멈췄다 → [트러블 10](10-배포-후-헬스체크가-안-떴다.md)

## 환경

- EC2 (Amazon Linux) + Docker, ap-northeast-2
- AWS Secrets Manager, EC2 인스턴스 IAM 롤로 접근
- 시크릿 이름: `gym-mgmt-dev/db-credentials`

## 진단

### 1) 접속 정보가 어떻게 컨테이너까지 오는가

```yaml
# .github/workflows/cd.yml (0940161 시점)
docker run -d \
  -e SECRET_ARN="${{ secrets.SECRET_ARN }}" \
  -e AWS_REGION="ap-northeast-2" \
```

```python
# api/database.py (0940161 시점)
secret_arn = os.getenv("SECRET_ARN")
client = boto3.client("secretsmanager", region_name=region)
secret = json.loads(client.get_secret_value(SecretId=secret_arn)["SecretString"])
```

ARN을 **GitHub Secret에 등록해 두고** 배포 때 컨테이너 환경변수로 주입하는 구조다.

### 2) 그 ARN은 어디서 오나

```hcl
# terraform/outputs.tf
output "db_secret_arn" {
  description = "Secrets Manager ARN — SECRET_ARN GitHub Secret에 등록"
}
```

`terraform apply` → 출력된 ARN을 **사람이 복사해서** GitHub Secret에 등록 → CD가 주입.
중간에 수동 단계가 있다.

### 3) GitHub Secret이 없으면 무슨 일이 일어나는가

`${{ secrets.SECRET_ARN }}`은 값이 없으면 오류가 아니라 **빈 문자열로 치환된다.**

```
-e SECRET_ARN=""
  → os.getenv("SECRET_ARN") == ""        (None 이 아니라 빈 문자열)
  → get_secret_value(SecretId="")
  → botocore ParamValidationError: Invalid length for parameter SecretId,
    value: 0, valid min length: 1
```

### 4) 왜 컨테이너가 죽었나

당시 `database.py`는 **모듈 최상단에서** 접속 정보를 만들었다.

```python
DATABASE_URL = _get_database_url()      # import 시점에 실행
```

import 중에 예외가 나면 uvicorn이 뜨기도 전에 프로세스가 끝난다.
`docker run -d`는 이미 성공을 돌려준 뒤라, **컨테이너만 조용히 사라진다.**

> **확정하지 못한 것:** GitHub Secret을 등록하지 않았던 건지, 등록했는데 값이 틀렸던 건지.
> 어느 쪽이든 위 경로는 같다. 확정하려면 당시 `docker logs` 출력이 필요하다.

## 원인

**접속 정보가 예측 불가능한 값(ARN)이라, 사람이 옮겨 적는 단계에 의존했다.**

그리고 그 값이 비어도 **오류가 아니라 빈 문자열로 흘러가** 실패가 늦게, 엉뚱한 곳에서 드러났다.

## 해결

시크릿을 **이름으로 조회**하고, 그 이름을 코드의 기본값으로 둔다. (`a4f560d`)

```diff
-    secret_arn = os.getenv("SECRET_ARN")
+    secret_name = os.getenv("SECRET_NAME", "gym-mgmt-dev/db-credentials")
-        client.get_secret_value(SecretId=secret_arn)["SecretString"]
+        client.get_secret_value(SecretId=secret_name)["SecretString"]
```

```diff
# cd.yml
-              -e SECRET_ARN="${{ secrets.SECRET_ARN }}" \
```

이름은 Terraform이 `"${local.name}/db-credentials"` 규칙으로 만들기 때문에 **미리 알 수 있다.**
ARN은 계정 ID와 무작위 접미사가 붙어 예측할 수 없고, 그래서 사람이 옮겨 적어야 했다.

GitHub Secret 하나와 수동 등록 단계가 함께 사라졌다.

## 재발 방지

- **예측 가능한 식별자를 쓴다.** 이름은 규칙으로 만들 수 있고 ARN은 아니다.
- 사람이 콘솔에서 값을 복사해 붙여 넣는 단계는 재현되지 않는다. 언젠가 빠뜨린다.
- **필수 설정이 비었으면 그 자리에서 실패해야 한다.** 빈 문자열이 그대로 흘러가면
  실패가 몇 단계 뒤에서 다른 얼굴로 나타난다.
- 기동에 실패하면 이유가 보여야 한다 → [트러블 10](10-배포-후-헬스체크가-안-떴다.md)

## 배운 점

**빈 값은 오류보다 나쁘다.** `${{ secrets.X }}`는 없는 시크릿을 빈 문자열로 바꿔 놓고 넘어간다.
그 값은 환경변수를 거쳐 API 호출까지 가서야 터진다. 그때는 원인에서 세 단계쯤 떨어져 있다.

**"한 번만 손으로 해 두면 되는 설정"은 파이프라인의 구멍이다.**
그 한 번을 안 하거나 잘못하면 아무도 모른다. 코드로 표현할 수 있으면 코드로 옮기는 게 맞다.
