# 09. kube-bench 는 GitOps 밖에서 온디맨드로

> 메모: 한 번 돌고 끝나는 Job 이지 ArgoCD 가 돌볼 서비스가 아니다. 매니페스트 + `make kube-bench`.
> 2026-07-16 `1be5613`, 07-23 1회 실행

---

## 상황

CIS 벤치마크를 넣기로 했다. 나머지는 전부 `gitops/apps/` 로 배포하는데, 이것도 Application 으로 둘지 정해야 했다.

## 결정

`security/kube-bench/job-eks.yaml`(Namespace + Job)을 손으로 apply 하고, Makefile 이 **apply → 완료 대기 → 로그 출력 → 삭제**를 한 번에 한다.

```
kube-bench run --targets node,policies,managedservices,controlplane --benchmark eks-1.5.0
hostPID: true · 읽기 전용 hostPath 3개 (/var/lib/kubelet · /etc/systemd · /etc/kubernetes)
이미지 aquasec/kube-bench:v0.15.6
```

## 왜

- **끝나는 작업이다.** 커밋 본문 — *"한 번 돌고 완료되는 Job 이지, ArgoCD 가 돌볼 서비스가 아니다."*
  ArgoCD 아래 두면 끝난 Job 을 계속 들고 있게 되고, 다시 돌리는 방법을 따로 설계해야 한다.
- **정책을 정면으로 위반한다.** 노드를 보려면 hostPID · hostPath 가 필수라 restricted 정책과 부딪힌다.
  전용 네임스페이스로 분리하고 Kyverno 에서 뺐다. → [결정 07](07-Kyverno를-Audit으로-착지시켰다.md) ·
  [security-lab/09](../../../infra-lab/security-lab/09-cluster-hardening/README.md) *"검사 도구가 정책을 위반한다"*

## 왜 다른 건 안 썼나

**ArgoCD Application**

위의 이유. `Completed` 상태를 어떻게 볼지 계속 시비가 붙는다.

**CronJob**

주기 실행은 오래 사는 클러스터의 답이다. 매일 사라지는 클러스터에선 수명 동안 한 번 돌까 말까라 의미가 약하다.

**다른 점검 도구 (kubescape · Trivy Operator …)**

[security-lab/09](../../../infra-lab/security-lab/09-cluster-hardening/README.md) 의 "다른 점검 도구"에 정리돼 있다. 저장소에서 비교한 흔적은 없다.

## 결과

07-23 에 한 번 돌렸다 — **12 PASS / 1 FAIL / 33 WARN.**
WARN 33 중 12 는 managedservices 섹션으로, 컨트롤플레인이 AWS 관리라 클러스터 안에서 확인할 수 없는 수동 항목이다.
검증 기록은 *"실질적으로 읽어야 할 것은 node · policies 섹션"* 이라고 적었다.

## 이 결정의 대가

**① 결과가 어디에도 안 남는다.** `make kube-bench` 는 로그를 터미널에 찍고 Job 을 지운다.
저장소에 남은 건 스크린샷 한 장인데, 그 스크린샷엔 **섹션 합계만** 있다. **FAIL 1건이 어느 항목인지 기록이 없다.**
후속 과제 *"FAIL 1건 원인 확인"* 은 다시 돌려야 시작할 수 있다.

**② 한 번은 한 번이다.** 매일 재생성하는 클러스터의 노드는 그날의 AMI 로 뜬다. 07-23 의 결과는 **그날 노드**에 대한 사실이다.

**③ 온디맨드는 "생각날 때"와 같다.** 12일 동안 한 번 돌았다. 경계를 긋는 결정이 **실행 주기**를 정하지 않으면 실행이 사람의 기억에 남는다 —
[결정 07](07-Kyverno를-Audit으로-착지시켰다.md) 의 Audit 이 Enforce 로 안 넘어간 것과 같은 모양이다.

## 배운 점

**"GitOps 로 관리할 것"과 "온디맨드로 돌릴 것"의 경계는 잘 그었다.** 끝나는 작업을 선언형 도구에 억지로 넣지 않았다.

**그런데 온디맨드 작업은 출력을 남기는 장치까지 같이 만들어야 한다.** 진단 도구의 결과가 터미널에서 사라지면
*"FAIL 이 무엇이었는지"* 부터 다시 알아내야 한다. 스크린샷이 영속성의 대체물이었던 [결정 05](05-관측성-데이터를-영속화하지-않는다.md) 와 같은 문제다.
