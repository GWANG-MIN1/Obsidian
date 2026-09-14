# 10. 시크릿은 SSM 에, Git 에는 참조만 — 계정 종속 조각은 example 로

> 메모: 값은 SSM Parameter Store, 동기화는 External Secrets Operator, AWS 인증은 IRSA. 정적 AWS 키는 어디에도 없다.
> 2026-07-16 `6bee990` · `08c2505`, 07-23 체인 실측

---

## 상황

Phase 3 에서 Grafana 비밀번호를 Git 에 안 넣고 차트 기본값으로 둔 채 넘어왔다.
공개 저장소라 시크릿은 절대 Git 에 들어갈 수 없고, 파드가 AWS 에서 값을 읽으려면 자격증명이 필요하다.

## 결정

```
Terraform        IAM 역할 eks-gitops-dev-external-secrets
                 ├ 신뢰: OIDC · sub = system:serviceaccount:external-secrets:external-secrets · aud = sts.amazonaws.com
                 └ 권한: ssm:GetParameter* on parameter/eks-gitops/dev/*  +  kms:Decrypt (kms:ViaService = ssm)
사람 (계정당 1회)  aws ssm put-parameter --type SecureString  /eks-gitops/dev/grafana/{admin-user, admin-password}
GitOps           External Secrets Operator 차트 2.7.0 (operator + CRD)
사람 (재생성마다)  kubectl annotate SA(역할 ARN) → rollout restart → ClusterSecretStore · ExternalSecret apply
```

SA 어노테이션 · ClusterSecretStore · ExternalSecret 은 **GitOps 로 동기화하지 않고** `security/external-secrets/examples/` 에 뒀다.

## 왜

- **Git 에는 경로만, 값은 AWS 에.** 파드는 ServiceAccount 토큰으로 역할을 맡으니 정적 키가 어디에도 없다.
- **최소 권한이 두 겹이다.** 신뢰 정책의 `sub` 조건으로 이 SA 만 역할을 맡고, 권한은 경로 접두사로 좁혀서
  dev 의 ESO 가 다른 환경의 파라미터를 못 읽는다. → [security-lab/06](../../../infra-lab/security-lab/06-secrets-management/README.md)
- **역할 ARN 에 계정 ID 가 들어간다.** 공개 저장소에 계정 종속 값을 안 넣으려고 어노테이션을 Git 밖에 뒀다.
- **SSM 파라미터는 destroy 에서 살아남는다.** Terraform 이 만들지 않았으니 매일 재생성해도 값은 그대로다.

## 왜 다른 건 안 썼나

**Sealed Secrets · SOPS**

암호문을 Git 에 넣는 방식이다 → [security-lab/06](../../../infra-lab/security-lab/06-secrets-management/README.md).
Sealed Secrets 는 복호화 키를 클러스터의 컨트롤러가 들고 있어서, 키를 따로 백업해 두지 않으면 **새로 만든 클러스터가 어제 봉인한 암호문을 못 연다.**
매일 destroy 와 상성이 나쁘다. *(저장소가 적은 이유는 아니다 — 이 노트의 판단)*

**EKS Pod Identity**

클러스터에 `eks-pod-identity-agent` 애드온까지 **설치해 놓고 안 썼다.** Pod Identity 는 네임스페이스 · SA · 역할의 연결을
EKS API 객체로 만들어서 Terraform 이 소유할 수 있고, SA 어노테이션이 필요 없다 — 재생성마다 치는 `kubectl annotate` 가 사라지는 선택지였다.
**왜 IRSA 인지는 적혀 있지 않고, ESO 의 Pod Identity 지원 여부는 이 노트에서 확인하지 않았다.**

**어노테이션을 values 에 커밋**

계정 ID 가 공개 저장소에 들어간다.

## 결과

07-23 에 체인 전체를 실측했다.

| 확인 | 결과 |
|---|---|
| ClusterSecretStore `aws-ssm` | `Valid` / READY `True` — IRSA 로 AWS 인증 성공 |
| ExternalSecret `grafana-admin` | `SecretSynced` / READY `True` |
| Secret `grafana-admin` | Opaque · DATA 2 |

어노테이션 뒤의 `rollout restart` 가 필수였다. IRSA 자격증명은 **파드가 뜰 때** 주입되기 때문이다.

## 이 결정의 대가

**① "Git 에는 참조만"이 실제로는 "Git 에는 예제만"이다.** ExternalSecret 매니페스트는 Git 에 있지만 어느 Application 도
`examples/` 를 가리키지 않는다. 그래서 클러스터를 재생성하면 시크릿 동기화는 **자동으로 돌아오지 않는다.**
참조가 **파일로는** Git 에 있고, **desired state 로는** 없다.

**② 계정 종속인 조각은 하나인데 셋을 뺐다.** ESO Application 주석은 SecretStore · ExternalSecret 이 계정 종속이라 example 로 뒀다고 적는다.
매니페스트를 보면 ClusterSecretStore 엔 **리전**만, ExternalSecret 엔 **SSM 경로**만 있다. 계정 ID 가 들어가는 건 SA 어노테이션 하나다.
나머지 둘은 GitOps 로 동기화해 두고, 어노테이션이 붙을 때까지 기다리게 할 수 있었다.

**③ 동기화된 Secret 을 아무도 안 쓴다.** Grafana 는 여전히 차트 기본 계정(observability README 의 `admin / prom-operator`)이고,
`admin.existingSecret` 연결은 후속 과제로 남았다. 체인이 증명한 것은 **"SSM → Kubernetes Secret"** 까지고 **"SSM → Grafana 로그인"** 은 아니다.

**④ 재생성마다 손으로 4단계.** annotate · restart · ClusterSecretStore · ExternalSecret —
[README](../README.md#숫자로-말할-것) 에 센 재생성 수동 명령 8개 중 절반이 이 결정에서 나온다.
[결정 01](01-Terraform은-클러스터-경계에서-멈춘다.md) 의 *"인계점이 안내문으로 남았다"* 가 가장 크게 드러난 곳이다.

## 배운 점

**"값이 Git 에 없다"와 "시크릿 관리가 Git 에 있다"는 다른 문장이다.** 앞의 것은 완벽하게 지켜졌다.
뒤의 것은 인증 연결 한 조각이 계정 종속이라는 이유로, **계정 종속이 아닌 조각들까지 같이 Git 밖으로 나갔다.**

**체인의 끝은 소비자까지 긋는다.** `SecretSynced` 는 중간 지점이다. 이 시크릿이 필요했던 이유 — Grafana 비밀번호 — 는 아직 그대로다.
