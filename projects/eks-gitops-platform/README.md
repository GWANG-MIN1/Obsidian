# eks-gitops-platform

Terraform으로 EKS를 올리고, 클러스터 안은 ArgoCD(GitOps)로, 그 위에 관측성·DevSecOps를 붙인 플랫폼 학습 프로젝트.

- 저장소: https://github.com/GWANG-MIN1/eks-gitops-platform
- 기간: 2026-06-24 ~ 2026-07-23 (커밋 51개 · 실제 작업일 11일 · 4단계)
- 한 줄 요약: **네 Phase의 코드를 나흘 만에 "완료"로 체크하고, 실제 클러스터에 올린 첫날
  정적 검사로는 안 잡히는 문제들을 만난 프로젝트 — 그리고 마지막 검증일(07-23)이
  이 저장소 기본값인 EKS 1.30의 연장 지원 종료일이었다**

> 저장소 README가 "이 프로젝트가 무엇인가"라면, 이 노트는 **"내가 무엇을 정했고 무엇에 막혔나"** 다.
> 같은 내용을 옮겨 적지 않는다. 다른 프로젝트는 [`projects/`](../README.md) 에 있다.

> **기록의 성격** — 다섯 프로젝트 중 **저장소 자체의 기록이 가장 촘촘하다.** Phase마다 검증 기록
> (`docs/verification/` 4개 · 스크린샷 16장)과 트러블슈팅 문서(`docs/troubleshooting/` 5개)가 이미 있다.
> 그래서 여기엔 명령·출력 원문을 옮기지 않고, **저장소 문서가 안 적은 것** —
> 버린 선택지, 재발 방지가 실제로 반영됐는지, 이 노트를 쓰며(2026-09-14) 확인한 것 — 만 적는다.

> **infra-lab과의 관계** — 이 프로젝트의 개념은 이미 [`infra-lab/`](../../infra-lab/README.md) 커리큘럼에 들어가 있다.
> terraform · observability · cicd · security · cost · reliability 트랙 6개가 **이 저장소가 끝난 뒤**
> (2026-08-04 ~ 08-10) 커밋됐고, 이 저장소의 매니페스트를 예제로 쓴다 (`🔧 실제로 겪은 것` 절들).
> 그래서 따로 개념 노트 트랙을 두지 않고 [아래](#개념-노트)에서 연결만 한다.

---

## [타임라인](타임라인.md)

커밋 51개 · 2026-06-24 ~ 07-23 · 4단계
(설계·뼈대 → **코드 완료 4일** → 첫 라이브 검증과 문제 3개 → 재생성하며 Phase 3·4 검증)

---

## 이 프로젝트의 축 — "코드 완료"와 "검증 완료"를 따로 체크했다

로드맵이 두 상태를 구분했다. 07-13 ~ 07-16 에 Phase 1~4 를 하루에 하나씩 **"코드 완료"** 로 체크했는데,
그 커밋들마다 *"박스는 validate 가 초록이라는 뜻이지 클러스터에서 돌았다는 뜻이 아니다"* 라고 적었다.
실제 계정에 올린 07-20 부터 나온 것들은 이렇다.

| 무엇이 | 어디서 드러났나 | 정적 검사로 잡혔을까 |
|---|---|---|
| nginx `1.27-alpine` 의 fixable CRITICAL CVE | **CI 게이트** — 첫 push, 7분 뒤 초록 | ✅ 그게 CI 의 일 → [결정 08](decisions/08-Trivy는-fixable-CRITICAL만-막는다.md) |
| 노드 2대 × max-pods 17 = 34 만석 | 라이브 — hook Job `Pending` | ❌ 파드 수는 떠 봐야 안다 → [트러블 01](troubleshooting/01-노드-두-대가-파드-34개로-만석.md) |
| persistence 를 끄자 `/var/loki` 가 없어짐 | 라이브 — CrashLoop | ❌ 차트 렌더링은 정상 → [트러블 02](troubleshooting/02-persistence를-끄자-Loki가-죽었다.md) |
| k8s 1.30 이 CRD `selectableFields` 를 버림 | 라이브 — 영구 OutOfSync | ❌ API 서버가 있어야 드러남 → [트러블 03](troubleshooting/03-Healthy인데-영원히-OutOfSync.md) ⭐ |
| 재생성마다 바뀌는 API 엔드포인트 | 라이브 — `no such host` | — 워크플로우의 틈 → [트러블 05](troubleshooting/05-재생성하자-kubectl이-no-such-host.md) |
| 알림 3개가 실제로 울리는가 | **한 번도 확인 안 함** | — → [결정 06](decisions/06-컨트롤플레인-수집을-끄고-알림은-3개만.md) |
| 기본값 1.30 의 지원 종료 | **이 노트를 쓰면서** (09-14) | ❌ 시간이 지나야 드러남 → [결정 03](decisions/03-전부-버전을-고정했다.md) ⭐ |

정적 검사(fmt · validate · `kubectl kustomize` · 차트 values 스키마 대조)는 전부 초록이었고,
라이브에서 나온 문제는 **전부 그 바깥**에 있었다. 저장소가 두 상태를 나눠 적은 것이
이 프로젝트에서 가장 잘한 일이고, 그 구분 덕분에 위 표를 복원할 수 있었다.

그런데 마지막 두 줄은 검증 기록이 끝난 **뒤**의 이야기다.
**"검증 완료"는 그날의 사실이지 계속 참인 사실이 아니다.** → [아직 안 한 것](#아직-안-한-것과-이유)

---

## 결정 — 왜 그렇게 만들었나

| | 노트 |
|---|---|
| 01 | [Terraform 은 클러스터 경계에서 멈추고 안쪽은 GitOps 에 넘긴다](decisions/01-Terraform은-클러스터-경계에서-멈춘다.md) |
| 02 | [매일 destroy 를 전제로 설계했다 — SPOT · 단일 NAT · 영구 자원은 state 백엔드뿐](decisions/02-매일-destroy를-전제로-설계했다.md) |
| 03 | [전부 버전을 고정했다 — 그리고 고정한 것이 늙었다](decisions/03-전부-버전을-고정했다.md) ⭐ |
| 04 | [app-of-apps 루트 하나 + 멀티소스 Application](decisions/04-app-of-apps-루트-하나와-멀티소스.md) |
| 05 | [관측성 데이터를 영속화하지 않는다 — Loki SingleBinary · PVC 없음](decisions/05-관측성-데이터를-영속화하지-않는다.md) |
| 06 | [EKS 컨트롤플레인 수집을 끄고, 알림은 3개만](decisions/06-컨트롤플레인-수집을-끄고-알림은-3개만.md) |
| 07 | [Kyverno 는 Audit 으로 착지시키고 인프라 네임스페이스는 뺀다](decisions/07-Kyverno를-Audit으로-착지시켰다.md) |
| 08 | [Trivy 는 fixable CRITICAL 만 막는다 — HIGH · IaC 는 보이게만](decisions/08-Trivy는-fixable-CRITICAL만-막는다.md) |
| 09 | [kube-bench 는 GitOps 밖에서 온디맨드로](decisions/09-kube-bench를-GitOps-밖에-뒀다.md) |
| 10 | [시크릿은 SSM 에, Git 에는 참조만 — 계정 종속 조각은 example 로](decisions/10-시크릿은-SSM에-Git에는-참조만.md) |

## 트러블슈팅 — 무엇에 막혔나

| | 노트 |
|---|---|
| 01 | [노드 두 대가 파드 34개로 만석 — CPU 가 아니라 IP 가 먼저 찼다](troubleshooting/01-노드-두-대가-파드-34개로-만석.md) |
| 02 | [persistence 를 끄자 Loki 가 죽었다](troubleshooting/02-persistence를-끄자-Loki가-죽었다.md) |
| 03 | [Healthy 인데 영원히 OutOfSync — 클러스터가 모르는 필드](troubleshooting/03-Healthy인데-영원히-OutOfSync.md) ⭐ |
| 04 | [root-app 직후 아무것도 안 뜬 것처럼 보였다](troubleshooting/04-root-app-직후-아무것도-안-뜬-것처럼-보였다.md) |
| 05 | [재생성하자 kubectl 이 no such host](troubleshooting/05-재생성하자-kubectl이-no-such-host.md) |

## 개념 노트

재사용 가능한 지식은 이미 infra-lab 커리큘럼에 있다. 이 프로젝트에서 나온 것과의 대응은 아래.

| 이 프로젝트에서 | infra-lab |
|---|---|
| state 부트스트랩 · 모듈 조립 · IRSA · Terraform/GitOps 경계 | [terraform-lab/09](../../infra-lab/terraform-lab/09-aws-vpc-eks/README.md) |
| 멀티소스 · ServerSideApply · 영구 OutOfSync | [cicd-lab/05](../../infra-lab/cicd-lab/05-argocd-advanced/README.md) |
| app-of-apps 부트스트랩 · sync-wave | [cicd-lab/06](../../infra-lab/cicd-lab/06-app-of-apps-applicationset/README.md) |
| Loki SingleBinary · read-only 파일시스템 | [observability-lab/06](../../infra-lab/observability-lab/06-logging/README.md) |
| EKS 컨트롤플레인 · 영속성 · ArgoCD 모니터링 · 용량 | [observability-lab/08](../../infra-lab/observability-lab/08-kube-prometheus-stack/README.md) |
| Audit → Enforce · 대조 실험 | [security-lab/04](../../infra-lab/security-lab/04-kyverno/README.md) |
| ESO + IRSA + SSM 체인 | [security-lab/06](../../infra-lab/security-lab/06-secrets-management/README.md) |
| Trivy 2단 게이팅 · 정기 재스캔 | [security-lab/07](../../infra-lab/security-lab/07-image-security/README.md) |
| kube-bench · 검사 도구가 정책을 위반 | [security-lab/09](../../infra-lab/security-lab/09-cluster-hardening/README.md) |
| max-pods 한계 · prefix delegation | [cost-lab/07](../../infra-lab/cost-lab/07-rightsizing/README.md) |

이 노트를 쓰며 나온 것 중 **아직 어느 트랙에도 없는** 후보 —

- [ ] EKS 버전 수명주기 — 표준 14개월 + 연장 12개월, 연장 구간은 클러스터 시간당 **$0.60** (표준 $0.10),
      종료일부터 그 버전으로 새 클러스터 생성 불가 ([cost-lab/04](../../infra-lab/cost-lab/04-compute-purchasing/README.md) 의 EKS 가격 구조엔 $0.10 만 있다)
- [ ] ArgoCD 는 1.8 부터 `argoproj.io/Application` 의 health 를 안 본다 — app-of-apps 의 sync-wave 는 자식을 기다리지 않는다
- [ ] kube-prometheus-stack 의 기본 Alertmanager receiver 는 `'null'` — Watchdog 이 들어 있지만 받는 곳이 없다
- [ ] GitHub Actions `schedule` 은 공개 저장소에서 60일 동안 활동이 없으면 꺼진다
- [ ] feature gate 단계와 관리형 K8s — EKS 는 알파 기능을 지원하지 않는다 (`selectableFields` 는 1.30 알파 · 1.31 베타)

---

## 숫자로 말할 것

| | |
|---|---:|
| 커밋 | 51개 (2026-06-24 ~ 07-23, 작업일 11일) |
| 병합 커밋 | 0회 — `main` 에 직접 |
| Phase 1~4 코드 완료 | 07-13 ~ 07-16 (하루에 한 Phase) |
| 라이브 검증 | 07-20 (Phase 1·2) · 07-22 (3) · 07-23 (4) — 클러스터 생성 최소 3회 |
| ArgoCD Application | 8개 (root + 자식 7) — **8/8 Synced/Healthy** |
| 코드 | Terraform 651줄 · YAML 1,185줄 · Markdown 1,197줄 |
| 노드 | t3.medium SPOT — **기본값 2대, 검증은 3대** |
| Kyverno | ClusterPolicy 4개 (규칙 5개) — sample-app PASS 5 / FAIL 0 · bad-pod PASS 1 / FAIL 4 |
| kube-bench (`eks-1.5.0`) | 12 PASS / 1 FAIL / 33 WARN |
| 커스텀 알림 | 3개 로드 확인 · **Firing 확인 0회** |
| Loki 조회 | `{namespace="sample-app"}` 398줄 |
| CI 실행 (09-07 까지) | 17회 — terraform-ci 3 (전부 성공) · security-ci 14 (실패 1 = 첫 push) |
| 주간 재스캔 | 8회 연속 초록 (07-20 ~ 09-07) |
| 저장소 문서 | 검증 기록 4 · 트러블슈팅 5 · 스크린샷 16장 |

**주의해서 말할 것**

- **"`terraform apply` 한 번으로"는 기본값으로는 성립하지 않았다.** 기본 `node_desired_size = 2` 로는
  전체 스택이 max-pods 에 걸린다. Phase 3·4 는 `-var="node_desired_size=3"` 을 **명령줄에서** 줘서 만들었고,
  Git 의 기본값은 지금도 2다. → [트러블 01](troubleshooting/01-노드-두-대가-파드-34개로-만석.md)
- **저장소 트러블슈팅 README 의 "전부 Git 수정 → push 로 해결"은 5건 중 2건(02·03)에만 맞다.**
  01 은 AWS CLI 로 노드를 늘렸고, 04 는 기다린 것이고, 05 는 로컬 kubeconfig 갱신이다.
  `kubectl edit` 0회는 사실이다.
- **"수동 조작은 딱 두 번"은 GitOps 가 넘겨받은 뒤의 이야기다.** 빈 계정에서 Phase 4 상태까지 가려면
  재생성마다 손으로 치는 명령이 8개다 — apply(`-var`) · update-kubeconfig · ArgoCD 설치 · root-app ·
  SA 어노테이션 · rollout restart · ClusterSecretStore · ExternalSecret. (검증 기록에서 센 값)
- **알림은 "로드됐다"까지만 확인했고, 받는 곳도 없다.** 차트 기본 receiver 가 `'null'` 인데 values 에서
  안 바꿨다. → [결정 06](decisions/06-컨트롤플레인-수집을-끄고-알림은-3개만.md)
- **비용을 집계하지 않았다.** 설계 전체가 비용 절감인데(매일 destroy · SPOT · 단일 NAT) 청구서로 확인한 값이 없다.
  게다가 1.30 은 검증 당시 **연장 지원 구간**이라 컨트롤플레인이 시간당 $0.60 였다(표준 $0.10).
  저장소 어디에도 "연장 지원"이라는 말이 없다. → [결정 03](decisions/03-전부-버전을-고정했다.md)
- **IaC 스캔(`trivy config`)은 report-only 고 결과 건수를 어디에도 안 적었다.**
  → [결정 08](decisions/08-Trivy는-fixable-CRITICAL만-막는다.md)
- **문서 일부가 검증 이전 상태로 남아 있다.** 모듈 README 두 개가 여전히 *"not yet apply-tested against a live account"* 이고,
  저장소 README 는 *"시작하기 (작성 중)"* 이다. 07-20 에 검증했지만 그 문장들은 안 고쳤다.

---

## 아직 안 한 것과 이유

**기본값 그대로는 이제 클러스터를 만들 수 없다** *(이 노트를 쓰며 찾은 것 — apply 로 확인하진 않음)*

`kubernetes_version` 기본값이 `"1.30"` 인데, EKS 1.30 의 연장 지원은 **2026-07-23 에 끝났다** — Phase 4 검증을 한 그날이다.
AWS 문서는 *"연장 지원 종료일부터는 그 버전으로 새 클러스터를 만들 수 없다"* 고 적는다.
**"매일 부수고 다시 만든다"를 전제로 설계한 저장소가, 코드 한 줄 안 바뀐 채로 다시 만들 수 없는 기본값을 들고 있다.**
트러블 03 의 `ignoreDifferences` 에 적어 둔 제거 조건(*"1.31+ 로 올리면"*)도 이제 선택이 아니라 전제다.
→ [결정 03](decisions/03-전부-버전을-고정했다.md)

**알림이 도착할 곳이 없다** *(이 노트를 쓰며 찾은 것)*

차트 87.16.1 의 기본 Alertmanager 설정은 receiver 가 `'null'` 하나뿐이고, `alerts.yaml` 의 3개도,
차트가 기본으로 넣는 **Watchdog**(항상 울려서 알림 경로가 살아 있음을 증명하는 알림)도 전부 거기로 간다.
[serverless-uptime-monitor](../serverless-uptime-monitor/README.md) 에서 손으로 만든 dead man's switch 가
**차트에 이미 들어 있는데 연결을 안 했다.** 후속 과제인 `SampleAppDown` Firing 실증을 해도
**Prometheus UI 에서 빨개지는 것**까지만 볼 수 있다.
→ [결정 06](decisions/06-컨트롤플레인-수집을-끄고-알림은-3개만.md)

**주간 재스캔이 곧 꺼진다** *(이 노트를 쓰며 찾은 것)*

GitHub 은 공개 저장소에서 **60일 동안 활동이 없으면 scheduled workflow 를 끈다.** 마지막 push 가 07-23 이니
커밋 기준으로 세면 09-21 전후다. security-ci 주석은 *"저장소가 안 바뀌어도 새 CVE 를 잡으려고 매주 재스캔한다"* 인데,
**저장소가 안 바뀌면 바로 그 스케줄이 꺼진다.**
→ [결정 08](decisions/08-Trivy는-fixable-CRITICAL만-막는다.md)

**sync-wave 가 자식을 기다리지 않는다** *(매니페스트와 ArgoCD 문서에서 찾은 것 — 클러스터에서 확인하진 않음)*

`kyverno`(wave 0) → `kyverno-policies`(wave 1) 순서를 걸었는데, ArgoCD 는 1.8 부터 **자식 `Application` 의 health 를
부모가 보지 않는다.** 문서는 app-of-apps 에서 sync wave 로 순서를 조율한다면 health check 를 복원해야 할 수 있다고 적는데,
이 저장소의 ArgoCD 설치는 순정 `install.yaml` 이라 그 설정이 없다. 그리고 `sample-app` 은 wave 가 없어(=0)
정책(wave 1)보다 **먼저** 뜬다. [cicd-lab/06](../../infra-lab/cicd-lab/06-app-of-apps-applicationset/README.md) 이
*"정책 엔진이 앱보다 먼저"* 라고 적은 원칙과 반대다.
→ [결정 04](decisions/04-app-of-apps-루트-하나와-멀티소스.md) · [결정 07](decisions/07-Kyverno를-Audit으로-착지시켰다.md)

**노드 수가 Git 밖에 있다**

실제로 돈 클러스터는 3대, Git 의 기본값은 2대다. 모듈이 `desired_size` 를 `ignore_changes` 로 두기 때문에
생성 후 변경은 AWS CLI 몫이었고, 생성 시점 값은 `-var` 로 줬다. 기본값을 3으로 올리거나
VPC CNI prefix delegation 을 켜는 게 남은 일이다. → [트러블 01](troubleshooting/01-노드-두-대가-파드-34개로-만석.md)

**저장소가 스스로 적은 후속 과제 — 전부 그대로**

Audit → Enforce 전환 · kube-bench FAIL 1건 확인 · Grafana 를 `grafana-admin` Secret 에 연결 · `SampleAppDown` Firing 실증.
07-23 이후 커밋이 없다. `docs/architecture.md` 의 "열어둔 질문" 3개(Karpenter · Kyverno vs OPA · Loki S3)도
**답이 안 채워졌다** — Kyverno 는 Phase 4 에서 이미 골랐는데 문서에선 여전히 열린 질문이다.

---

## 다른 프로젝트와의 대비

| | 검증 | 무너진 지점 |
|---|---|---|
| [gym-management-db](../gym-management-db/README.md) | 리뷰 받고 테스트 56개 | 4개월간 `\|\| true` 로 CI 가 무력 |
| [aws-serverless-agent](../aws-serverless-agent/README.md) | 매일 실배포 + 손으로 호출 | 완성 8일 뒤 재배포에서 전 요청 500 |
| [serverless-uptime-monitor](../serverless-uptime-monitor/README.md) | 테스트 78개 + CI 게이트 | 그래도 프로덕션이 조용히 멈췄다 |
| [riskdetector](../riskdetector/README.md) (팀) | 실배포 + 손으로 호출 | 백엔드가 비용 때문에 교체됐다 |
| **eks-gitops-platform** | **Phase별 라이브 검증 기록 + CI 게이트** | **코드는 그대로인데 기본값이 수명을 다했다** |

**검증을 가장 형식적으로 남긴 프로젝트다.** "코드 완료"와 "검증 완료"를 로드맵에서 나눴고,
Phase 마다 확인 명령 · 결과 표 · 스크린샷을 남겼다. aws-serverless-agent 가 매일 손으로 확인하고
**그 확인을 남기지 않았다면**, 여기는 **확인을 문서로 남겼다.**

**그런데 운영 신호 층이 비어 있다.** 다섯 프로젝트에서 실제로 장애를 잡은 건
[검증 층위 표](../README.md#세-프로젝트가-서로를-비춘다)의 마지막 줄, **운영 신호**였다.
이 저장소는 알림 규칙까지 만들고 받는 곳을 안 붙였다. uptime-monitor 가 요약 알림으로
**정상을 능동적으로 증명**하게 만든 것 — 차트의 Watchdog 이 정확히 그 역할인데 `'null'` 로 간다.

**aws-serverless-agent 의 교훈이 다른 형태로 돌아왔다.** 거기선 *"손으로 하는 검증은 환경이 바뀌면 재현되지 않는다"* 였다.
여기선 전부 IaC 로 자동화했는데도 **플랫폼의 시간**이 흘러 기본값이 생성 불가가 됐고,
주간 재스캔도 저장소가 조용하면 꺼진다. **코드가 안 바뀌어도 코드 밖의 시간은 흐른다.**

**비용 쪽은 riskdetector 와 같은 공백이다.** 다섯 개 중 비용을 설계 조건으로 가장 강하게 건 프로젝트인데
(매일 destroy), 청구서 숫자가 없고, 버전 선택 하나가 컨트롤플레인 요금을 6배로 만든 것을 모른 채 지나갔다.
