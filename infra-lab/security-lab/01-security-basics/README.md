# 01 클라우드 네이티브 보안 기초

보안이 **"배포 직전에 한 번 검토하는 절차"** 로 남아 있으면 반드시 뚫린다. 사람이 매번 볼 수 없고, 봐도 놓치기 때문이다.  
DevSecOps의 실질적 의미는 **보안 검사를 파이프라인과 클러스터에 상시 돌아가는 자동 가드레일로 만드는 것**이다.

---

## 4C 모델 — 어디를 지킬 것인가

```
┌───────────────────────────────────────────┐
│  Cloud   (AWS 계정·IAM·VPC·EKS 설정)       │
│ ┌───────────────────────────────────────┐ │
│ │ Cluster (RBAC·admission·NetworkPolicy)│ │
│ │ ┌───────────────────────────────────┐ │ │
│ │ │ Container (이미지·권한·capability)│ │ │
│ │ │ ┌───────────────────────────────┐ │ │ │
│ │ │ │ Code (의존성·시크릿·입력검증) │ │ │ │
│ │ │ └───────────────────────────────┘ │ │ │
│ │ └───────────────────────────────────┘ │ │
│ └───────────────────────────────────────┘ │
└───────────────────────────────────────────┘
```

| 계층 | 위협 | 대응 | 다루는 곳 |
|---|---|---|---|
| **Cloud** | IAM 과다 권한, 퍼블릭 노출 | 최소권한 IAM, IRSA, VPC 격리 | `../terraform-lab/09-aws-vpc-eks/` |
| **Cluster** | RBAC 남용, 무제한 통신 | RBAC, admission 정책, NetworkPolicy | `04-kyverno/` `05-rbac-least-privilege/` `08-network-security/` |
| **Container** | 취약 이미지, root 실행 | 스캔, securityContext, 비-root | `02-pod-container-security/` `07-image-security/` |
| **Code** | 의존성 CVE, 하드코딩 시크릿 | SCA, 시크릿 스캔, 코드 리뷰 | `06-secrets-management/` |

> **바깥 계층이 뚫리면 안쪽 방어는 의미가 없다.** IAM 키가 유출되면 클러스터 정책이 아무리 촘촘해도 소용없다.  
> 그래서 순서는 바깥부터다 — 컨테이너 하드닝보다 IAM 최소권한이 먼저다.

---

## 두 겹으로 세운다 — Shift-left + Runtime

```
[ CI — shift-left ]                      [ 클러스터 — runtime ]
Trivy 이미지 스캔                          Kyverno admission 정책
Trivy IaC 스캔          ────────▶         Pod Security Admission
시크릿 스캔                                NetworkPolicy
     │                                          │
"머지되기 전에 잡는다"                    "그래도 들어오면 막는다"
```

| | CI (shift-left) | 런타임 (admission) |
|---|---|---|
| 시점 | PR·머지 전 | 리소스 생성 시 |
| 우회 가능성 | **있다** (CI를 안 거치면) | 없다 (API 서버를 거쳐야 한다) |
| 피드백 | 빠르다 (개발자에게 즉시) | 늦다 (배포 시점) |
| 범위 | 우리 저장소 | 클러스터 전체 |

> **둘 중 하나만으로는 부족하다.** CI 검사는 `kubectl apply`로 우회되고, admission만 있으면 개발자가 배포 직전에야 실패를 안다.  
> 실제 구성에서도 이렇게 나눠 쓴다 — **Trivy가 CI에서 취약 이미지를 잡고, Kyverno가 클러스터에서 비준수 파드를 막는다.**

---

## 최소권한 원칙

모든 계층에 같은 원칙이 반복된다.

```
"이 주체가 이 일을 하는 데 꼭 필요한 만큼만"

IAM       : s3:GetObject on 특정 버킷      (s3:* on * 아님)
IRSA      : 특정 ServiceAccount 만 assume  (sub 조건)
RBAC      : 특정 네임스페이스의 특정 리소스 (cluster-admin 아님)
컨테이너   : capability 전부 drop 후 필요한 것만 add
네트워크   : default deny 후 필요한 통신만 allow
```

> **"일단 넓게 주고 나중에 좁힌다"는 영원히 안 좁혀진다.** 반대로 좁게 시작하면 필요할 때 요청이 오므로 자연스럽게 정확해진다.  
> 다만 이건 **처음부터 막으면 아무도 못 쓴다**는 문제와 충돌한다. 그 해법이 다음 절의 Audit-then-Enforce다.

---

## Audit → Enforce — 신뢰를 잃지 않고 정책을 도입하는 법

```
1. Audit 모드로 배포     → 위반을 기록만 한다. 아무것도 안 막힌다.
2. 리포트를 읽는다        → 무엇이 깨질지 미리 본다.
3. 워크로드를 고친다      → 또는 정책을 현실에 맞게 조정한다.
4. Enforce 로 전환        → 리포트가 깨끗해진 정책부터 하나씩.
```

> **살아있는 클러스터에 Enforce를 바로 켜는 것이 플랫폼 팀이 신뢰를 잃는 가장 빠른 길이다.**  
> 어제까지 되던 배포가 오늘 갑자기 거부되면, 개발팀은 정책을 우회할 방법을 찾거나 플랫폼 팀을 거치지 않을 방법을 찾는다.  
> 정책은 **정책별로** 전환한다. 4개를 한꺼번에 Enforce로 바꾸지 않는다. → `04-kyverno/`

### 인프라 네임스페이스는 제외한다

```yaml
exclude:
  any:
    - resources:
        namespaces:
          - kube-system
          - kube-node-lease
          - kube-public
          - kyverno
          - argocd
          - observability
          - external-secrets
          - kube-bench
```

> 업스트림에서 관리하는 컴포넌트는 restricted 기준을 만족하지 않는 경우가 많다. **정책의 대상은 "우리 워크로드"** 다.  
> 대표적인 예가 kube-bench다 — 노드를 검사해야 하므로 `hostPID`와 hostPath 마운트가 필요하다. **보안 검사 도구 자체가 보안 정책을 위반한다.** 그래서 그 네임스페이스를 제외한다.  
> 다만 제외 목록은 **의도적인 예외**여야 한다. "안 되니까 뺐다"가 쌓이면 정책이 껍데기가 된다.

---

## 위협 모델 — 무엇을 막으려는가

```
공격자가 파드 하나를 장악했다고 가정한다. 그 다음 무엇을 할 수 있는가?
```

| 공격 경로 | 막는 방법 | 다루는 곳 |
|---|---|---|
| 컨테이너 탈출 → 노드 장악 | 비-root, capability drop, seccomp | `02-pod-container-security/` |
| 노드의 IMDS로 IAM 자격증명 탈취 | **IMDSv2 + hop limit**, IRSA | `09-cluster-hardening/` |
| ServiceAccount 토큰으로 API 서버 조작 | RBAC 최소권한, 토큰 automount 끄기 | `05-rbac-least-privilege/` |
| 다른 파드로 횡이동 | NetworkPolicy default-deny | `08-network-security/` |
| Secret 조회 | RBAC, 외부 시크릿 저장소 | `06-secrets-management/` |
| 악성 이미지 배포 | 이미지 서명 검증, admission | `07-image-security/` |
| 데이터 외부 유출 | 이그레스 제한, 감사 로그 | `08-network-security/` `10-runtime-detection/` |

> **가장 자주 실현되는 경로가 IMDS 자격증명 탈취다.** 파드에서 노드의 인스턴스 메타데이터에 접근하면 **노드 역할의 IAM 권한을 그대로 얻는다.**  
> IRSA를 쓰는 진짜 이유가 여기 있다 — 노드 역할을 최소화하고 파드마다 필요한 권한만 준다.

---

## 실제 구성 — 레이어별 도구

| 레이어 | 도구 | 위치 | 하는 일 |
|---|---|---|---|
| 이미지 스캔 | **Trivy** | CI | 고칠 수 있는 CRITICAL CVE에서 빌드 실패, HIGH는 리포트 |
| IaC 스캔 | **Trivy config** | CI | Terraform·매니페스트 미스컨피그 (리포트 전용) |
| Admission 정책 | **Kyverno** | 클러스터 | 비준수 파드 차단·기록 |
| 벤치마크 | **kube-bench** | 온디맨드 | CIS EKS 벤치마크 리포트 |
| 시크릿 | **External Secrets** | 클러스터 | AWS SSM에서 가져옴 — Git엔 참조만 |

```
CI 게이트는 좁게: 고칠 수 있는 CRITICAL 만 실패
                  HIGH·IaC 는 리포트 (로그에는 보이되 main 은 초록)
```

> **게이트를 넓게 잡으면 `main`이 상시 빨개지고, 그러면 아무도 안 본다.** 알림 피로와 같은 실패 양상이다. → `../cicd-lab/01-cicd-basics/`  
> 패치가 없는 CVE(`ignore-unfixed`)로 빌드를 깨뜨리면 개발자는 스캔을 끄는 법을 배운다.

---

## 규정 준수와 실제 보안은 다르다

```
CIS 벤치마크 100% 통과  ≠  안전하다
```

| 체크리스트가 답 못 하는 것 |
|---|
| 우리 IAM 역할에 실제로 필요 없는 권한이 있는가 |
| 이 파드가 저 DB에 접근할 이유가 있는가 |
| 유출됐을 때 피해가 가장 큰 시크릿이 무엇인가 |

> 벤치마크는 **바닥선**이지 목표가 아니다. 통과했다고 안전한 게 아니라, 통과 못 하면 확실히 위험한 것이다.  
> 반대로 EKS처럼 관리형 환경에서는 **조치할 수 없는 항목**이 리포트에 남는다 — 컨트롤 플레인은 AWS 영역이다. 그걸 0으로 만들려 애쓰는 건 시간 낭비다. → `09-cluster-hardening/`

---

## 배운 점

- 보안은 배포 직전 검토가 아니라 **상시 돌아가는 자동 가드레일**이어야 한다
- 4C(**Cloud → Cluster → Container → Code**) — **바깥 계층이 먼저**다
- IAM이 뚫리면 클러스터 정책은 의미가 없다
- **CI(shift-left)와 admission(runtime) 두 겹**으로 세운다
- CI 검사는 `kubectl apply`로 우회되고, admission만으로는 피드백이 늦다
- 최소권한은 IAM·IRSA·RBAC·capability·네트워크 **모든 계층에 같은 원칙**으로 반복된다
- "넓게 주고 나중에 좁힌다"는 영원히 안 좁혀진다
- **살아있는 클러스터에 Enforce를 바로 켜면 플랫폼 팀이 신뢰를 잃는다**
- **Audit → 리포트 확인 → 수정 → 정책별로 Enforce 전환**
- 인프라 네임스페이스는 제외한다 — 정책의 대상은 **우리 워크로드**
- kube-bench는 `hostPID`가 필요해 스스로 정책을 위반한다 (의도적 예외)
- 제외 목록이 "안 되니까 뺐다"로 쌓이면 정책이 껍데기가 된다
- 위협 모델은 **"파드 하나가 뚫렸다면 그 다음은?"** 로 시작한다
- 가장 자주 실현되는 경로는 **IMDS를 통한 노드 IAM 자격증명 탈취** — IRSA를 쓰는 진짜 이유
- CI 게이트를 넓게 잡으면 main이 상시 빨개지고 아무도 안 본다
- **CIS 100% 통과 ≠ 안전** — 벤치마크는 바닥선이지 목표가 아니다
- 관리형 환경에서는 **조치할 수 없는 항목**이 남는 게 정상이다
