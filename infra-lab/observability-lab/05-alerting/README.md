# 05 알림 (Alerting)

알림은 **사람을 깨우는 일**이다. 그래서 "탐지할 수 있는 것"이 아니라 "지금 사람이 행동해야 하는 것"만 알려야 한다.  
기술적으로는 Prometheus가 룰을 평가해 발화하고, Alertmanager가 그걸 묶어서 보낸다 — **두 컴포넌트의 역할 분리**를 이해하는 게 먼저다.

---

## 역할 분리

```
Prometheus                          Alertmanager
┌────────────────────┐             ┌──────────────────────┐
│ 룰 평가             │             │ 그룹핑 (묶기)         │
│ Pending → Firing   │──── HTTP ──▶│ 중복 제거             │
│                    │             │ 억제 (inhibit)        │
│ "조건이 참이다"      │             │ 사일런스              │
└────────────────────┘             │ 라우팅 → Slack/PD     │
                                   └──────────────────────┘
   "무엇이 문제인가"                    "누구에게 어떻게 알릴까"
```

> 알림이 안 오면 **어느 쪽에서 멈췄는지부터 확인**한다. Prometheus에서 firing이 아닌 건지, Alertmanager까지 갔는데 안 보낸 건지에 따라 원인이 완전히 다르다.

```bash
curl -s localhost:9090/api/v1/rules | jq    # Prometheus 에서 firing 인가?
curl -s localhost:9093/api/v2/alerts | jq   # Alertmanager 까지 갔는가?
```

---

## 알림 룰

```yaml
groups:
  - name: platform.workloads
    rules:
      - alert: SampleAppDown
        expr: kube_deployment_status_replicas_available{namespace="sample-app", deployment="sample-app"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "sample-app에 가용 레플리카가 없습니다"
          description: >-
            데모 워크로드가 5분간 완전히 불가용 상태입니다.
            sample-app 네임스페이스의 파드와 ArgoCD Application을 확인하세요.
```

| 필드 | 역할 |
|---|---|
| `alert` | 알림 이름 (CamelCase 관례) |
| `expr` | 참이면 발화하는 PromQL |
| `for` | **이 시간 동안 계속 참이어야** 발화 (일시적 스파이크 무시) |
| `labels` | 라우팅에 쓰인다 (`severity`가 핵심) |
| `annotations` | 사람이 읽는 내용 (`summary`, `description`, `runbook_url`) |

### Inactive → Pending → Firing

```
조건 거짓          조건 참              for 시간 경과
Inactive ────▶ Pending ────────────▶ Firing ──▶ Alertmanager
                  │
                  └─ for 안에 거짓이 되면 Inactive 로 복귀 (알림 안 감)
```

> **`for`가 오탐을 막는 가장 강력한 장치다.** `for` 없이 쓰면 스크랩 한 번 실패에도 알림이 간다.  
> 다만 `for: 30m`처럼 길게 잡으면 그만큼 늦게 안다. 심각도에 따라 조절한다 (critical 2~5분, warning 15~30분).

### 템플릿

```yaml
annotations:
  summary: "{{ $labels.pod }} 파드가 재시작 중입니다"
  description: "지난 1시간 동안 {{ $value | humanize }}회 재시작했습니다."
  runbook_url: "https://wiki.example.com/runbooks/{{ $labels.alertname }}"
```

| 변수 | 의미 |
|---|---|
| `{{ $labels.x }}` | 발화한 시계열의 라벨 |
| `{{ $value }}` | 그때의 값 |
| `{{ $value \| humanize }}` | 읽기 좋게 (1500000 → 1.5M) |

> Helm values에 알림을 넣을 때 `{{ }}`가 Helm 템플릿과 충돌할 것 같지만, kube-prometheus-stack 차트는 `toYaml`로 그대로 통과시키므로 **Prometheus 템플릿이 온전히 전달된다.**

---

## 심각도 설계

| severity | 뜻 | 채널 |
|---|---|---|
| `critical` | **지금 사람이 일어나야 한다** | 전화·PagerDuty |
| `warning` | 업무 시간에 보면 된다 | Slack |
| `info` | 알림 아님, 기록용 | 대시보드 |

```
❓ 새벽 3시에 이걸로 깨워도 되는가?
   YES → critical
   NO  → warning 이하
```

> **critical의 개수가 곧 팀의 삶의 질이다.** 새벽에 울렸는데 아침까지 기다려도 됐던 알림은 즉시 강등한다.

---

## 무엇을 알릴 것인가

### 증상에 알린다, 원인이 아니라

```
❌ CPU 80% 초과              ← 그래서 사용자가 불편한가? 모른다
❌ 파드 재시작 1회             ← 자동 복구됐을 수 있다
✅ 에러율 5% 초과 (5분)        ← 사용자가 실패를 겪고 있다
✅ p99 지연 2초 초과 (10분)    ← 사용자가 느림을 겪고 있다
✅ 가용 레플리카 0 (5분)       ← 서비스가 죽었다
```

> **원인 지표는 대시보드에, 증상 지표는 알림에.** CPU가 높아도 사용자가 멀쩡하면 깨울 이유가 없고, CPU가 낮아도 에러가 나면 깨워야 한다.

### 이 스택의 알림 설계

기본 차트가 이미 kubernetes-mixin 룰(크래시루프, 노드 압박, 볼륨 포화 등)을 제공한다. **거기에 없는 것만** 추가한다.

```yaml
additionalPrometheusRulesMap:
  platform-rules:
    groups:
      - name: platform.gitops
        rules:
          # GitOps의 전제가 깨졌다: Git과 클러스터가 다르다
          - alert: ArgoCDAppNotSynced
            expr: argocd_app_info{sync_status!="Synced"} == 1
            for: 15m
            labels: { severity: warning }
            annotations:
              summary: "ArgoCD 앱 {{ $labels.name }}이 15분째 OutOfSync"

          # 동기화는 됐는데 안 돈다 = 드리프트가 아니라 잘못된 배포
          - alert: ArgoCDAppUnhealthy
            expr: argocd_app_info{health_status!="Healthy"} == 1
            for: 15m
            labels: { severity: warning }
```

> **상위 차트가 제공하는 룰을 다시 쓰지 않는다.** 중복 알림은 신호가 아니라 소음이다.  
> 플랫폼 고유의 약속 — "Git과 클러스터가 일치한다", "배포 경로가 실제로 동작한다" — 만 직접 정의한다.

### 발화할 수 없는 룰은 끈다

```yaml
defaultRules:
  rules:
    etcd: false                   # EKS: 컨트롤 플레인 스크랩 불가
    kubeControllerManager: false
    kubeSchedulerAlerting: false
    kubeProxy: false
    windows: false                # Windows 노드 없음
```

> 데이터가 없는 대상에 걸린 룰은 **영원히 발화하거나 영원히 unknown** 상태가 된다. 둘 다 알림 전체의 신뢰를 갉아먹는다. → `08-kube-prometheus-stack/`

---

## Alertmanager

### 그룹핑

```yaml
route:
  group_by: ["alertname", "namespace"]
  group_wait: 30s          # 첫 알림 전 대기 (같은 그룹의 다른 알림을 모은다)
  group_interval: 5m       # 같은 그룹에 새 알림이 추가됐을 때 다음 발송 간격
  repeat_interval: 4h      # 해결 안 된 알림 재통지 주기
  receiver: slack-default
```

```
노드 1대가 죽어 파드 30개가 동시에 실패
  그룹핑 없이 → 알림 30개
  그룹핑 있이 → 알림 1개 ("30 alerts firing")
```

> `group_wait`가 0이면 첫 알림이 온 뒤 나머지가 하나씩 따라온다. **30초만 기다려도 대부분 한 묶음이 된다.**

### 라우팅 트리

```yaml
route:
  receiver: slack-default
  group_by: ["alertname", "namespace"]

  routes:
    - matchers: ['severity="critical"']
      receiver: pagerduty
      continue: false                  # 여기서 멈춘다 (기본값)

    - matchers: ['namespace="sample-app"']
      receiver: slack-app-team

    - matchers: ['severity="info"']
      receiver: "null"                 # 버린다

receivers:
  - name: slack-default
    slack_configs:
      - api_url_file: /etc/alertmanager/secrets/slack-webhook   # 파일로 주입
        channel: "#alerts"
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: "null"
```

> 라우팅은 **위에서부터 첫 매치에서 멈춘다.** 여러 곳으로 보내려면 `continue: true`.  
> **Webhook URL을 values.yaml에 직접 쓰지 않는다.** Secret으로 마운트하고 `api_url_file`로 참조한다.

### 억제 (Inhibition)

```yaml
inhibit_rules:
  # 노드가 죽었으면 그 노드 위 파드 알림은 보내지 않는다
  - source_matchers: ['alertname="NodeDown"']
    target_matchers: ['severity="warning"']
    equal: ["node"]

  # 같은 대상에 critical 이 있으면 warning 은 억제
  - source_matchers: ['severity="critical"']
    target_matchers: ['severity="warning"']
    equal: ["alertname", "namespace"]
```

> 억제는 **근본 원인 알림이 있을 때 파생 알림을 숨긴다.** 노드 장애 시 알림 50개 대신 1개를 받게 하는 장치다.

### 사일런스

```bash
amtool silence add alertname=SampleAppDown -d 2h -c "계획된 배포 작업"
amtool silence query
amtool silence expire <ID>
```

> 계획된 작업 전에 미리 건다. **사일런스는 만료 시간을 반드시 넣는다** — 무기한 사일런스는 알림을 지운 것과 같다.

---

## 알림 피로

가장 흔한 실패는 탐지 실패가 아니라 **너무 많이 알려서 아무도 안 보게 되는 것**이다.

| 증상 | 대응 |
|---|---|
| 알림 채널을 아무도 안 본다 | critical 개수를 세본다. 하루 5건 넘으면 과하다 |
| "그건 원래 그래요" | 발화할 수 없는 룰·상시 위반 룰을 끈다 |
| 같은 장애로 알림 수십 개 | 그룹핑 + 억제 규칙 |
| 받아도 뭘 할지 모른다 | `runbook_url` 추가, 없으면 알림 자체를 재검토 |
| 자동 복구되는데 알림이 온다 | `for`를 늘리거나 알림을 삭제 |

### 점검 질문

```
□ 지난달 이 알림이 몇 번 울렸고, 그중 몇 번 실제로 조치했는가?
    → 조치율이 낮으면 삭제 후보
□ 이 알림 없이 장애를 알 수 있었는가?
    → 있으면 중복
□ 새벽 3시에 울려도 되는가?
    → 아니면 warning 으로 강등
```

> **알림은 추가보다 삭제가 어렵다.** 분기마다 한 번씩 발화 이력을 보고 정리하는 습관이 필요하다.

---

## 테스트

```bash
promtool check rules alerts.yaml     # 문법 검사
promtool test rules test.yaml        # 단위 테스트
```

```yaml
# test.yaml — 룰이 의도대로 발화하는지 검증
rule_files: ["alerts.yaml"]
evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'kube_deployment_status_replicas_available{namespace="sample-app", deployment="sample-app"}'
        values: "0+0x10"          # 10분간 0 유지
    alert_rule_test:
      - eval_time: 6m
        alertname: SampleAppDown
        exp_alerts:
          - exp_labels:
              severity: critical
              namespace: sample-app
              deployment: sample-app
```

```bash
# 라우팅이 의도한 receiver 로 가는지 시뮬레이션
amtool config routes test severity=critical namespace=sample-app
```

> **알림 룰도 코드다.** CI에서 `promtool check rules`를 돌리면 오타로 알림이 조용히 죽는 사고를 막을 수 있다. → `../terraform-lab/10-cicd-policy/`

---

## 배운 점

- **Prometheus는 발화까지, Alertmanager는 발송까지** — 알림이 안 오면 어느 쪽인지부터 확인
- `for`가 오탐을 막는 핵심 장치 — 없으면 스크랩 한 번 실패에도 알림이 간다
- 상태 전이는 **Inactive → Pending(`for` 대기) → Firing**
- `severity`는 라우팅의 기준 — **"새벽 3시에 깨워도 되는가"** 로 critical을 가른다
- **원인이 아니라 증상에 알린다** — CPU 80%가 아니라 에러율·지연·가용성
- 상위 차트(kubernetes-mixin)가 주는 룰을 다시 쓰지 않는다. **없는 것만** 추가
- **발화할 수 없는 룰은 끈다** (EKS의 etcd·scheduler) — 영원히 빨간 알림은 신뢰를 파괴한다
- 그룹핑(`group_by` + `group_wait`)으로 장애 1건 = 알림 1건이 되게 만든다
- 라우팅은 **위에서 첫 매치에서 멈춘다** (`continue: true`로 계속 진행)
- **Webhook URL은 values에 쓰지 않고** Secret 마운트 후 `api_url_file`로 참조
- 억제(inhibit)로 근본 원인 알림이 있을 때 파생 알림을 숨긴다
- 사일런스에는 **반드시 만료 시간**을 넣는다
- 알림은 추가보다 삭제가 어렵다 — 조치율이 낮은 알림은 삭제 후보
- `promtool test rules`로 알림 룰도 단위 테스트할 수 있다
