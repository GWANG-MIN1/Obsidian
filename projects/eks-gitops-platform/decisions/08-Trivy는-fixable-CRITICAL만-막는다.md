# 08. Trivy 는 fixable CRITICAL 만 막는다 — HIGH · IaC 는 보이게만

> 메모: 게이트를 넓게 잡으면 `main` 이 늘 빨갛고, 그러면 아무도 안 본다. 오늘 고칠 수 있는 것만 막고 나머지는 로그에 남긴다. 매주 재스캔.
> 2026-07-16 `7743e07` · 첫 실행에서 막힘 → `592ebc6`

---

## 상황

Phase 4 의 shift-left 절반이다. 스캔 대상은 둘 — sample-app 이 쓰는 **업스트림 이미지**, 그리고 Terraform · 매니페스트(**IaC**).
무엇이 CI 를 빨갛게 만들 자격이 있는지 정해야 했다.

## 결정

| 잡 | 대상 | 심각도 | 결과 |
|---|---|---|---|
| image-scan ① | `nginx-unprivileged:1.30.4-alpine` | HIGH · CRITICAL | 표로 출력만 (`exit-code: 0`) |
| image-scan ② | 같은 이미지 | CRITICAL + `ignore-unfixed` | **실패하면 빨간불** (`exit-code: 1`) |
| iac-scan | 저장소 전체 (`trivy config .`) | HIGH · CRITICAL | 출력만 |

트리거는 관련 경로의 push · PR, **매주 월요일 06:00 UTC 재스캔**, 수동 실행.

## 왜

- **초록이 신호여야 한다.** 워크플로우 주석 — *"main 이 소음이 아니라 신호로 초록을 유지하게."* 막는 것은 **오늘 조치할 수 있는 것**뿐이다.
- **`ignore-unfixed`** — 패치가 없는 CVE 로 빌드를 깨면 개발자는 스캔을 끄는 법을 배운다. → [security-lab/07](../../../infra-lab/security-lab/07-image-security/README.md)
- **주간 재스캔** — 태그를 고정하면 코드가 안 바뀌어서 push 트리거가 안 걸리는데, CVE 는 매일 새로 공개된다.
- **IaC 는 report-only** — 주석: *"런타임에서 Kyverno 가 같은 규칙을 강제하므로 여기선 보이게만."*

## 왜 다른 건 안 썼나

**HIGH 까지 게이트** — `main` 이 빨간불로 굳는다.

**IaC 도 게이트** — 먼저 모든 경고를 분류해야 한다. 그 작업 없이 켜면 빨간불로 굳는다.
uptime-monitor [결정 09](../../serverless-uptime-monitor/decisions/09-tfsec을-soft-fail로.md) 와 같은 판단이다.

**재스캔 없음** — 고정 이미지는 영원히 다시 안 보게 된다.

## 결과 — 만든 날 막았다

| 실행 | 시각 (UTC) | 결과 |
|---|---|---|
| security-ci #1 | 07-16 10:47 | ❌ `1.27-alpine` 의 fixable CRITICAL |
| security-ci #2 | 07-16 10:55 | ✅ `1.30.4-alpine` 으로 올린 커밋 |
| #3 ~ #14 | 07-20 ~ 09-07 | ✅ push 4회 · 주간 스케줄 8회 |

게이트를 넣은 push 의 첫 실행이 빨갛게 됐고 **7분 뒤** 초록이 됐다.
uptime-monitor 의 tfsec 은 soft-fail 이라 한 번도 막지 않았는데, 여기선 **게이트가 실제로 막는다는 증거가 첫날 생겼다.**
다만 **어떤 CVE 였는지는 저장소에 없다** — 커밋은 *"a fixable CRITICAL"* 이라고만 적었고, Actions 로그는 이 노트에서 열어 보지 않았다.

## 이 결정의 대가

**① 두 층이 서로를 믿고, 둘 다 안 막는다.** iac-scan 을 report-only 로 둔 근거가 *"Kyverno 가 런타임에서 강제한다"* 인데,
Kyverno 는 **같은 날 같은 push 에서 Audit 으로** 들어갔다. → [결정 07](07-Kyverno를-Audit으로-착지시켰다.md)
그래서 잘못된 설정은 **CI 에서도 클러스터에서도 막히지 않는다** — 둘 다 기록만 한다.

**② IaC 스캔에서 무엇이 몇 건 나왔는지 모른다.** 결과는 실행 로그에만 있고 저장소 어디에도 요약이 없다.
uptime-monitor README 가 tfsec 에 대해 쓴 문장 — *"IaC 보안 스캔이 있다"와 "배포를 막는다"는 다르다* — 가 그대로 걸린다.

**③ 스캔하는 이미지와 배포하는 이미지가 따로 적혀 있다.** `image-ref` 는 워크플로우에 하드코딩이고,
주석은 *"deployment.yaml 과 맞춰 둘 것"* 이다. deployment 만 바꾸고 워크플로우를 안 바꾸면 **CI 는 옛 이미지를 스캔하고 초록**이다.
07-16 의 수정은 두 파일을 같이 고쳤지만, 그걸 보장하는 장치는 사람의 기억뿐이다.

**④ 주간 재스캔이 곧 꺼진다.** GitHub 은 공개 저장소에서 **60일 동안 활동이 없으면 scheduled workflow 를 끈다.**
마지막 push 가 07-23 이니 커밋 기준으로 09-21 전후다. 이 스케줄의 존재 이유가 *"저장소가 안 바뀌어도 새 CVE 를 보려고"* 인데,
**저장소가 안 바뀌면 바로 그 스케줄이 꺼진다.**

**⑤ 클러스터 안에서는 이미지를 안 본다.** CI 가 보는 건 이미지 이름 하나다. 노드에 실제로 떠 있는 이미지를 보는
Trivy Operator 나 Kyverno `verifyImages` 는 없다. → [security-lab/07](../../../infra-lab/security-lab/07-image-security/README.md)

## 배운 점

**게이트의 가치는 "막은 적이 있는가"로 증명된다.** 첫 push 의 빨간불 하나가 이 게이트를 장식이 아니게 만들었다.

**그런데 스케줄은 조용히 멈춘다.** 주간 재스캔은 성공하면 아무 소리도 안 내는 종류라, 꺼져도 겉으로는 차이가 없다 —
**초록이 멈춘 것과 초록인 것이 같아 보인다.** uptime-monitor [트러블 01](../../serverless-uptime-monitor/troubleshooting/01-헬스체크가-조용히-멈췄다.md) 의
*"헬스체크가 멈췄는데 DOWN 알림은 한 건도 없었다"* 와 같은 구조가 CI 에도 있었다.
