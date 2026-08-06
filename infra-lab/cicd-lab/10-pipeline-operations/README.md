# 10 파이프라인 운영

파이프라인을 만드는 것과 **운영하는 것**은 다르다. 만든 뒤에는 느려지고, 불안정해지고, 아무도 안 보는 알림이 쌓인다.  
이 장은 파이프라인 자체를 측정하고 개선하는 방법, 그리고 실제로 자주 막히는 지점들이다.

---

## DORA 지표 측정하기

`01-cicd-basics/`에서 정의한 4지표를 실제로 어떻게 재는가.

| 지표 | 데이터 출처 | 계산 |
|---|---|---|
| **배포 빈도** | ArgoCD 동기화 이벤트 / GH Actions 성공 실행 | 기간당 배포 횟수 |
| **변경 리드타임** | 커밋 시각 → 배포 완료 시각 | 중앙값 |
| **변경 실패율** | 롤백·핫픽스 배포 수 / 전체 배포 수 | 비율 |
| **복구 시간** | 장애 시작 → 복구 배포 완료 | 중앙값 |

```promql
# 배포 빈도 — ArgoCD 동기화 성공 (일 단위)
sum(increase(argocd_app_sync_total{phase="Succeeded"}[1d]))

# 앱별 배포 빈도
sum by (name) (increase(argocd_app_sync_total{phase="Succeeded"}[7d]))

# 동기화 실패율
sum(rate(argocd_app_sync_total{phase="Failed"}[1d]))
  / sum(rate(argocd_app_sync_total[1d]))

# 현재 문제 있는 앱
argocd_app_info{sync_status!="Synced"} == 1
argocd_app_info{health_status!="Healthy"} == 1
```

> ArgoCD 메트릭을 수집하려면 ServiceMonitor를 붙여야 한다. → `../observability-lab/08-kube-prometheus-stack/`  
> **리드타임과 복구 시간은 자동 수집이 어렵다.** 완벽한 계측보다, 주 단위로 대충이라도 기록하는 편이 낫다 — 절대값이 아니라 **추세**가 중요하다.

### 지표를 볼 때 주의

```
❌ "배포 빈도를 늘려라"           → 의미 없는 배포를 만들어낸다
✅ "리드타임 중앙값이 3일 → 왜?"   → 병목을 찾는다 (리뷰 대기? 파이프라인? 승인?)
```

> **지표는 목표가 아니라 진단 도구다.** 지표를 목표로 삼으면 조작된다(굿하트의 법칙).

---

## 파이프라인 성능

```bash
gh run list --workflow=ci.yml --limit 50 --json databaseId,conclusion,createdAt,updatedAt \
  | jq -r '.[] | [.conclusion, ((.updatedAt|fromdate) - (.createdAt|fromdate))] | @tsv'
```

| 흔한 병목 | 대응 |
|---|---|
| 의존성 재설치 | 캐시 (`actions/cache`, lock 해시 키) |
| 도커 빌드 캐시 미스 | `cache-from/to type=gha`, 레이어 순서 |
| 순차 실행 | 독립 job 병렬화 (`needs` 최소화) |
| E2E 테스트 | PR에서 분리 → 머지 후·야간 |
| 큰 체크아웃 | `fetch-depth: 1` (기본값) |
| 무의미한 실행 | `paths` 필터, `concurrency` 취소 |

```yaml
# 병렬화 — lint/test/scan 은 서로 독립
jobs:
  lint: { runs-on: ubuntu-latest }
  test: { runs-on: ubuntu-latest }
  scan: { runs-on: ubuntu-latest }
  build:
    needs: [lint, test, scan]     # 셋이 끝나야 시작
```

> **`needs`를 습관적으로 붙이면 파이프라인이 직렬화된다.** 정말 의존하는 것만 연결한다.

### 플레이키 테스트

```
가끔 실패하는 테스트가 하나 있으면
      ↓
"재실행하면 되겠지" 가 습관이 된다
      ↓
진짜 실패도 재실행으로 넘긴다   💥
```

> **플레이키 테스트는 버그로 취급한다.** 고칠 수 없으면 격리(quarantine)하고 별도로 추적한다. CI 신뢰도가 무너지는 가장 흔한 경로다.

---

## 배포 알림

```yaml
- name: Slack 알림
  if: always()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "${{ job.status == 'success' && ':white_check_mark:' || ':x:' }} *${{ github.repository }}* 배포 ${{ job.status }}",
        "blocks": [{
          "type": "section",
          "text": {
            "type": "mrkdwn",
            "text": "커밋: <${{ github.server_url }}/${{ github.repository }}/commit/${{ github.sha }}|`${{ github.sha }}`>\n실행: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|로그 보기>"
          }
        }]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

| 알릴 것 | 알리지 말 것 |
|---|---|
| **운영 배포 성공/실패** | 모든 PR CI 결과 (GitHub UI에 이미 있다) |
| 자동 롤백 발생 | dev 환경 배포 |
| 배포 승인 요청 | 린트 경고 |
| 보안 스캔 CRITICAL | 매일 도는 스케줄 잡의 성공 |

> **관측성의 알림 원칙이 그대로 적용된다** — 행동할 수 없는 알림은 소음이다. → `../observability-lab/05-alerting/`  
> 알림에는 **커밋 링크와 실행 로그 링크**를 넣는다. "실패했다"만 알려주면 결국 사람이 찾아 들어가야 한다.

### GitOps 쪽 알림

```yaml
# ArgoCD Notifications
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-degraded]
```

```yaml
# 또는 Prometheus 알림으로 (이 프로젝트 방식)
- alert: ArgoCDAppNotSynced
  expr: argocd_app_info{sync_status!="Synced"} == 1
  for: 15m
- alert: ArgoCDAppUnhealthy
  expr: argocd_app_info{health_status!="Healthy"} == 1
  for: 15m
```

> **`for: 15m`이 중요하다.** 배포 중에는 잠시 OutOfSync가 정상이다. 유예 없이 알리면 배포마다 알림이 온다.

---

## 롤백 절차

장애 시 판단이 흐려지므로 **미리 정해두고 문서화**한다.

```
1. 방금 배포가 원인인가?          배포 시각과 지표 악화 시각을 비교
2. 롤백할 것인가, 고칠 것인가?     원인이 5분 안에 안 나오면 → 롤백
3. 롤백 실행
4. 확인
5. 사후: 원인 분석 → 재발 방지
```

| 상황 | 명령 |
|---|---|
| GitOps 배포 | `git revert <커밋> && git push` ← **정석** |
| ArgoCD 리비전 롤백 | `argocd app rollback myapp 3` |
| Rollout 진행 중 | `kubectl argo rollouts abort myapp` |
| Rollout 완료 후 | `kubectl argo rollouts undo myapp` |
| Deployment | `kubectl rollout undo deploy/myapp` |
| Blue-Green | Service selector 되돌리기 (즉시) |

```bash
# 배포 이력 확인
argocd app history myapp
kubectl rollout history deploy/myapp -n myapp

# 롤백 후 확인
kubectl rollout status deploy/myapp -n myapp --timeout=300s
kubectl -n argocd get app myapp -o jsonpath='{.status.sync.status} {.status.health.status}'
```

> **`argocd app rollback`이나 `kubectl rollout undo`는 클러스터만 되돌린다.** Git은 여전히 새 버전을 가리키므로, `selfHeal`이 켜져 있으면 몇 초 뒤 다시 새 버전으로 되돌아간다.  
> **GitOps에서 진짜 롤백은 `git revert`다.** 응급 시 클러스터 명령으로 시간을 벌고, 곧바로 Git을 맞춘다.

```
응급: kubectl argo rollouts abort   (초 단위 정지)
정식: git revert && git push        (Git = 클러스터 일치 회복)
```

### 롤백할 수 없는 것

| | 이유 |
|---|---|
| DB 마이그레이션 | 컬럼을 지웠으면 되돌릴 수 없다 → `08-deployment-strategies/` |
| 외부로 나간 이벤트 | 발송된 메일·웹훅 |
| 삭제된 데이터 | 백업에서 복구해야 한다 |

> **"롤백 가능한 배포"를 설계하는 게 롤백 절차보다 먼저다.** Expand-Contract, 피처 플래그, 멱등한 마이그레이션이 그 수단이다.

---

## 자주 겪는 문제

### GitHub Actions

| 증상 | 원인·해결 |
|---|---|
| OIDC 인증 실패 | `permissions: id-token: write` 누락, IAM 신뢰 정책 `sub` 불일치 |
| 캐시가 안 먹음 | 키에 lock 해시가 없거나, 브랜치별로 캐시가 격리됨 |
| 시크릿이 비어 있음 | 포크 PR에는 시크릿이 전달되지 않는다 (정상 동작) |
| 워크플로가 안 돌음 | `paths` 필터, 브랜치 조건, 워크플로 파일 위치(`.github/workflows/`) |
| 조건이 항상 false | `${{ }}` 안 문자열 비교 — `github.ref`는 `refs/heads/main` 전체 |
| 러너 대기 | 동시 실행 한도 — `concurrency`로 불필요한 실행 취소 |

### ArgoCD

| 증상 | 원인·해결 |
|---|---|
| 푸시했는데 반영 안 됨 | 폴링 기본 3분 → 웹훅 설정, `refresh=hard` |
| 영구 OutOfSync | 클러스터가 모르는 필드 → 실제 diff 확인 → `05-argocd-advanced/` |
| `annotations: Too long` | `ServerSideApply=true` |
| 삭제 후 리소스 잔존 | `finalizers` 누락 |
| Helm 렌더링 실패 | `argocd app manifests`, repo-server 로그 |
| replicas 원복 반복 | HPA 충돌 → `ignoreDifferences` |

### 배포

| 증상 | 원인·해결 |
|---|---|
| 배포 중 5xx | readinessProbe 부정확, `preStop`·우아한 종료 부재 |
| 롤아웃이 멈춤 | probe 실패, 리소스 부족(Pending), 이미지 pull 실패 |
| 카나리 분석 Inconclusive | 트래픽 부족 → `count`·`interval` 조정 |
| 롤백했는데 다시 새 버전 | `selfHeal` — Git을 되돌려야 한다 |

```bash
# 배포가 멈췄을 때 진단 순서
kubectl -n myapp get pods                      # Pending? CrashLoop? ImagePullBackOff?
kubectl -n myapp describe pod <POD>            # Events 를 본다
kubectl -n myapp logs <POD> --previous         # 이전 컨테이너 로그
kubectl -n myapp get events --sort-by=.lastTimestamp | tail -20
```

---

## 파이프라인 점검 체크리스트

```
□ PR 파이프라인이 10분 안에 끝나는가
□ 플레이키 테스트가 없는가 (재실행이 습관이 아닌가)
□ 자격증명이 장기 키가 아니라 OIDC인가
□ 서드파티 액션이 SHA로 고정돼 있는가
□ 이미지 태그가 불변인가 (latest 미사용)
□ 운영 배포에 승인 게이트가 있는가
□ 롤백 절차가 문서화돼 있고 실제로 해봤는가
□ 배포 알림이 행동 가능한 것만 오는가
□ 보안 스캔이 주기적으로 재실행되는가
□ 파이프라인 자체가 코드로 리뷰되는가
```

> **"롤백을 실제로 해봤는가"** 가 가장 중요하다. 장애 상황에서 처음 해보는 절차는 대부분 실패한다. 평소에 한 번 돌려본다.

---

## 배운 점

- DORA 지표는 **목표가 아니라 진단 도구** — 목표로 삼으면 조작된다
- 리드타임·복구시간은 자동 수집이 어렵다 — **절대값보다 추세**
- `needs`를 습관적으로 붙이면 파이프라인이 직렬화된다
- **플레이키 테스트는 버그로 취급**한다 — 재실행이 습관이 되면 CI 신뢰가 무너진다
- 알림은 **운영 배포·자동 롤백·승인 요청**만, PR CI 결과는 GitHub UI에 이미 있다
- 알림에는 **커밋 링크와 실행 로그 링크**를 넣는다
- ArgoCD 알림에 **`for: 15m` 유예**가 없으면 배포마다 알림이 온다
- **`argocd app rollback`·`kubectl rollout undo`는 클러스터만 되돌린다** — selfHeal이 원복시킨다
- **GitOps의 진짜 롤백은 `git revert`** — 클러스터 명령은 시간을 버는 응급 조치
- **롤백할 수 없는 것들이 있다** — DB 마이그레이션, 발송된 이벤트, 삭제된 데이터
- 롤백 절차보다 **"롤백 가능한 배포"를 설계하는 것**이 먼저
- 포크 PR에 시크릿이 안 가는 건 정상 동작이다
- `github.ref`는 `refs/heads/main` 전체 문자열 — 비교 시 자주 틀린다
- 배포 진단은 **pods → describe(Events) → logs --previous → events** 순서
- **롤백을 평소에 한 번 해본다** — 장애 때 처음 하는 절차는 실패한다
