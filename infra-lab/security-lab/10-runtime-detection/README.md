# 10 런타임 탐지 & 사고 대응

앞의 9장은 전부 **예방**이다. 그런데 예방은 언제나 뚫린다 — 제로데이, 설정 실수, 내부자, 아직 안 켠 Enforce.  
**"뚫렸을 때 알아챌 수 있는가"** 와 **"알아챈 뒤 무엇을 할 것인가"** 가 남은 절반이다.

---

## 예방과 탐지

```
예방 (Prevention)                    탐지 (Detection)
─────────────────                    ────────────────
admission 정책                        Falco 런타임 이벤트
이미지 스캔                           감사 로그 분석
RBAC · NetworkPolicy                 이상 트래픽
"들어오지 못하게"                     "들어온 걸 알아채게"
```

| | 예방 | 탐지 |
|---|---|---|
| 시점 | 배포 전·admission | **실행 중** |
| 대상 | 알려진 위험한 설정 | **알려지지 않은 행동** |
| 실패 방식 | 우회·미적용 | 오탐·놓침 |

> **admission이 막을 수 있는 건 "설정"이지 "행동"이 아니다.** 정상적으로 통과한 컨테이너가 나중에 셸을 띄우고 `/etc/shadow`를 읽는 건 admission의 관심 밖이다.  
> 그래서 런타임 탐지가 필요하다. 다만 **탐지는 예방을 다 한 뒤의 이야기**다 — 기본기를 건너뛰고 Falco부터 깔면 알림만 쌓인다.

---

## Falco

**시스템콜 수준에서 컨테이너 행동을 관찰**하는 CNCF 런타임 보안 도구다.

```
컨테이너의 시스템콜
      │  eBPF (또는 커널 모듈)
      ▼
   Falco 엔진 ──▶ 룰 매칭 ──▶ 이벤트 (stdout / Falcosidekick → Slack·Loki)
```

```yaml
# 룰 예시 — 컨테이너 안에서 셸이 뜨면
- rule: Terminal shell in container
  desc: 컨테이너 안에서 대화형 셸이 생성됨
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
  output: >
    컨테이너에서 셸 실행 (user=%user.name container=%container.name
    image=%container.image.repository cmd=%proc.cmdline)
  priority: WARNING
```

### 기본 룰이 잡는 것들

| 탐지 | 왜 의심스러운가 |
|---|---|
| 컨테이너에서 셸 실행 | 정상 운영에서는 드물다 |
| 민감 파일 읽기 (`/etc/shadow`) | 자격증명 수집 |
| 쓰기 금지 디렉터리에 쓰기 | 바이너리 심기 |
| 컨테이너에서 패키지 설치 | 공격 도구 반입 |
| **IMDS 접근** | 노드 자격증명 탈취 → `09-cluster-hardening/` |
| 예상 밖 네트워크 연결 | C2 통신·유출 |
| 권한 상승 시도 | setuid 실행 |

```bash
kubectl -n falco logs -l app.kubernetes.io/name=falco -f
```

### ⚠️ 오탐 관리가 전부다

```
Falco 를 기본 룰로 깔면
      ↓
CI/CD 잡, 헬스체크 스크립트, 초기화 컨테이너가 전부 걸린다
      ↓
알림이 하루 수백 건 → 아무도 안 본다   💥
```

```yaml
# 예외 정의 — 알려진 정상 동작은 제외한다
- macro: known_ci_containers
  condition: container.image.repository in ("myorg/ci-runner", "myorg/migrate")

- rule: Terminal shell in container
  condition: >
    spawned_process and container and shell_procs and proc.tty != 0
    and not known_ci_containers
  append: true
```

> **이건 관측성의 알림 피로와 정확히 같은 문제다.** → `../observability-lab/05-alerting/`  
> 도입 순서도 같다: **먼저 로그만 수집(알림 없이) → 정상 패턴 파악 → 예외 정의 → 그 다음 알림.**  
> Falco 이벤트를 Loki로 보내면 다른 로그와 함께 조회할 수 있어 초기 튜닝이 훨씬 쉽다. → `../observability-lab/06-logging/`

---

## 감사 로그

**누가 무엇을 했는지에 대한 유일한 기록**이다. Falco가 "컨테이너 안"을 본다면 감사 로그는 "API 서버"를 본다.

```bash
aws eks update-cluster-config --name my-cluster \
  --logging '{"clusterLogging":[{"types":["audit","authenticator"],"enabled":true}]}'
```

### 무엇을 찾을 것인가

```
# CloudWatch Logs Insights

# ① Secret 조회 — 가장 중요한 감시 대상
fields @timestamp, user.username, verb, objectRef.namespace, objectRef.name
| filter objectRef.resource = "secrets" and verb in ["get","list"]
| sort @timestamp desc

# ② exec / attach — 운영자가 컨테이너에 들어간 기록
fields @timestamp, user.username, objectRef.namespace, objectRef.name
| filter objectRef.subresource in ["exec","attach","portforward"]

# ③ 권한 변경
fields @timestamp, user.username, verb, objectRef.resource, objectRef.name
| filter objectRef.resource in ["clusterrolebindings","rolebindings","clusterroles","roles"]
  and verb in ["create","update","patch","delete"]

# ④ 인가 실패 — 정찰(reconnaissance) 신호
fields @timestamp, user.username, verb, objectRef.resource
| filter responseStatus.code = 403
| stats count() by user.username
| sort by count() desc

# ⑤ 익명 접근
fields @timestamp, verb, objectRef.resource
| filter user.username = "system:anonymous"
```

> **403이 갑자기 몰리는 계정은 정찰 중일 가능성이 높다.** 공격자가 탈취한 토큰으로 "무엇을 할 수 있는지" 훑어보면 인가 실패가 대량으로 남는다.  
> `pods/exec` 사용 기록은 **정상 운영에서도 나오지만 감사 대상**이다. 누가 언제 운영 파드에 들어갔는지는 남아야 한다.

---

## 침해 지표 (IoC)

```
□ 예상 밖 네임스페이스에 새 파드
□ privileged / hostPath 파드가 갑자기 생성
□ 알 수 없는 이미지 (미승인 레지스트리)
□ ServiceAccount 토큰의 비정상적 사용 위치
□ 이그레스 트래픽 급증 (데이터 유출)
□ 컨테이너에서 셸·패키지 매니저 실행
□ 새 ClusterRoleBinding, 특히 cluster-admin
□ 인가 실패(403) 급증
□ 크립토마이닝 — CPU 사용률 지속 급등
```

```promql
# 관측성 지표로도 잡힌다
sum by (pod) (rate(container_cpu_usage_seconds_total[5m])) > 0.9   # 마이닝 의심
sum by (pod) (rate(container_network_transmit_bytes_total[5m]))    # 유출 의심
```

> **크립토마이닝은 보안 알림보다 CPU 그래프에서 먼저 보이는 경우가 많다.** 관측성과 보안은 분리된 영역이 아니다.  
> 이미 세워둔 대시보드와 알림이 침해 탐지의 1차 센서가 된다. → `../observability-lab/01-observability-basics/`

---

## 사고 대응 절차

장애 대응과 마찬가지로 **미리 정해두고 문서화**한다. 사고 중에는 판단이 흐려진다.

```
1. 탐지 (Detect)      알림·이상 지표 확인, 오탐 여부 1차 판단
2. 격리 (Contain)     확산을 막는다  ← 가장 먼저 할 일
3. 조사 (Investigate) 증거를 보존하고 범위를 파악
4. 제거 (Eradicate)   원인 제거
5. 복구 (Recover)     정상화
6. 사후 (Post-mortem) 재발 방지
```

### ② 격리 — 지우기 전에 격리한다

```bash
# 네트워크 격리 (파드는 살려둔다)
kubectl -n myapp label pod <POD> quarantine=true
# → 이 라벨을 대상으로 하는 default-deny NetworkPolicy 적용

# ServiceAccount 권한 차단
kubectl -n myapp delete rolebinding <BINDING>

# 노드 격리 (드레인하지 않는다 — 증거 보존)
kubectl cordon <NODE>
```

> ⚠️ **파드를 먼저 지우면 증거가 사라진다.** 프로세스 목록, 메모리, 파일시스템, 열린 연결 전부다.  
> **격리 → 증거 수집 → 그 다음 제거** 순서를 지킨다. Deployment가 관리하는 파드는 지우면 즉시 재생성되므로 격리가 더 실용적이기도 하다.

### ③ 조사 — 증거 수집

```bash
kubectl -n myapp get pod <POD> -o yaml > evidence-pod.yaml
kubectl -n myapp logs <POD> --all-containers --timestamps > evidence-logs.txt
kubectl -n myapp logs <POD> --previous > evidence-logs-prev.txt
kubectl -n myapp describe pod <POD> > evidence-describe.txt
kubectl get events -A --sort-by=.lastTimestamp > evidence-events.txt

# 프로세스 확인 (읽기 전용 디버그 컨테이너)
kubectl debug -it <POD> --image=busybox --target=app -n myapp -- ps aux
```

```
확인할 것
  이미지 다이제스트 — 무엇이 실제로 돌았나
  ServiceAccount   — 어떤 권한을 갖고 있었나
  마운트된 볼륨·Secret
  감사 로그에서 그 SA 의 활동 전부
```

### ④⑤ 제거와 복구

```bash
# 자격증명 로테이션 — 노출 가능성이 있으면 전부
aws ssm put-parameter --overwrite --type SecureString --name /... --value "$(openssl rand -base64 32)"
kubectl -n myapp rollout restart deploy    # 새 값 반영

# 이미지 교체 후 재배포 (GitOps 이므로 커밋)
git revert <취약 이미지 커밋> && git push
```

> **"어디까지 노출됐는지 모르겠다"면 전부 로테이션한다.** 시크릿 로테이션 비용은 침해 지속 비용보다 항상 싸다. → `06-secrets-management/`  
> GitOps 환경에서는 복구도 **Git 커밋**으로 한다. `kubectl`로 고치면 selfHeal이 되돌린다. → `../cicd-lab/10-pipeline-operations/`

### ⑥ 사후 분석

```
□ 어떻게 들어왔는가 (초기 침투 경로)
□ 왜 예방에서 못 막았는가 (정책 부재? Audit 상태? 예외 목록?)
□ 왜 더 일찍 탐지 못 했는가
□ 무엇을 바꿔야 재발하지 않는가  ← 사람이 아니라 시스템을 바꾼다
```

> **"누가 실수했는가"가 아니라 "왜 그 실수가 가능했는가"** 를 묻는다. 비난은 다음 사고를 숨기게 만든다.  
> 트러블슈팅 문서와 같은 형식으로 남긴다: **현상 / 환경 / 원인 / 해결 / 재발 방지.**

---

## 준비해둘 것

```
□ 감사 로그가 켜져 있고 보존 기간이 충분한가
□ 격리용 NetworkPolicy 가 미리 준비돼 있는가
□ 긴급 대응자가 필요한 권한을 갖고 있는가 (평소엔 없어야 한다)
□ 시크릿 로테이션 절차가 문서화·검증돼 있는가
□ 백업에서 복구를 실제로 해봤는가
□ 연락 체계 (누구에게 언제 알리는가)
```

> **"평소엔 권한이 없다가 긴급 시 승격"** 이 최소권한과 대응 속도를 동시에 만족시키는 방법이다(break-glass).  
> 승격 자체가 감사 로그에 남으므로 남용도 추적된다.

---

## 배운 점

- **예방은 언제나 뚫린다** — 탐지와 대응이 남은 절반
- **admission이 막는 건 "설정"이지 "행동"이 아니다** — 통과한 컨테이너의 이후 행동은 관심 밖
- 다만 **탐지는 예방을 다 한 뒤**의 이야기다 — 기본기 없이 Falco부터 깔면 알림만 쌓인다
- Falco는 **시스템콜 수준(eBPF)** 에서 컨테이너 행동을 관찰한다
- ⚠️ **Falco 도입의 전부는 오탐 관리** — 기본 룰로 깔면 CI 잡·헬스체크가 전부 걸린다
- 순서: **로그만 수집 → 정상 패턴 파악 → 예외 정의 → 알림**
- Falco 이벤트를 Loki로 보내면 초기 튜닝이 훨씬 쉽다
- 감사 로그는 **누가 무엇을 했는지에 대한 유일한 기록**
- 감시 우선순위: **Secret 조회 / exec·attach / 권한 변경 / 403 급증 / 익명 접근**
- **403이 몰리는 계정은 정찰 중**일 가능성이 높다
- **크립토마이닝은 보안 알림보다 CPU 그래프에서 먼저 보인다** — 관측성이 1차 센서
- 대응 순서: **탐지 → 격리 → 조사 → 제거 → 복구 → 사후**
- ⚠️ **파드를 먼저 지우면 증거가 사라진다** — 격리 후 증거 수집, 그 다음 제거
- Deployment 파드는 지워도 재생성되므로 **격리가 더 실용적**이다
- "어디까지 노출됐는지 모르겠다"면 **전부 로테이션** — 로테이션 비용이 항상 싸다
- GitOps 환경의 복구도 **Git 커밋**으로 (`kubectl`은 selfHeal이 되돌린다)
- 사후 분석은 **"누가 실수했나"가 아니라 "왜 그 실수가 가능했나"**
- **break-glass**(평소 무권한 → 긴급 시 승격)로 최소권한과 대응 속도를 함께 만족시킨다
