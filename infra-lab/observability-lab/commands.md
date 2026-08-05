# 관측성 명령어 레퍼런스

## PromQL — 기본 조회
```
up                                             # 모든 타겟의 수집 성공 여부 (1=성공)
up{job="node-exporter"}                        # 라벨로 필터
up{job=~"node.*"}                              # 정규식 매칭
up{job!="kubelet"}                             # 부정
node_cpu_seconds_total{mode="idle"}[5m]        # 최근 5분 구간 벡터
node_memory_MemAvailable_bytes offset 1h       # 1시간 전 값
```

## PromQL — 카운터 다루기
```
rate(http_requests_total[5m])                  # 초당 증가율 (카운터에는 항상 rate)
irate(http_requests_total[5m])                 # 마지막 두 샘플 기준 순간 변화율
increase(http_requests_total[1h])              # 1시간 동안의 증가량
resets(process_cpu_seconds_total[1h])          # 카운터 리셋 횟수
```

## PromQL — 집계
```
sum(rate(http_requests_total[5m]))                        # 전체 합
sum by (pod) (rate(http_requests_total[5m]))              # pod 별 합
sum without (instance) (rate(http_requests_total[5m]))    # instance 라벨만 버리고 합
avg / min / max / count / stddev                          # 다른 집계 연산자
topk(5, sum by (pod) (rate(container_cpu_usage_seconds_total[5m])))
bottomk(3, ...)
count(up == 0)                                            # 다운된 타겟 수
```

## PromQL — 자주 쓰는 계산
```
# CPU 사용률 (%)
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 메모리 사용률 (%)
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# 디스크 사용률 (%)
100 * (1 - node_filesystem_avail_bytes{fstype!~"tmpfs"} / node_filesystem_size_bytes)

# 에러율 (%)
100 * sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))

# p95 지연 (히스토그램)
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# 파드 재시작
increase(kube_pod_container_status_restarts_total[1h]) > 0

# 디스크가 4시간 안에 찰지 예측
predict_linear(node_filesystem_avail_bytes[6h], 4*3600) < 0
```

## LogQL — Loki 조회
```
{namespace="sample-app"}                                  # 라벨 셀렉터 (필수)
{namespace="sample-app", container="app"}
{namespace="sample-app"} |= "error"                       # 포함
{namespace="sample-app"} != "healthz"                     # 미포함
{namespace="sample-app"} |~ "(?i)timeout|refused"         # 정규식
{namespace="sample-app"} | json | level="error"           # JSON 파싱 후 필드 필터
{namespace="sample-app"} | logfmt | duration > 1s
{job="app"} | pattern "<_> <status> <_>" | status="500"

# 메트릭으로 변환
rate({namespace="sample-app"} |= "error" [5m])            # 초당 에러 로그 수
sum by (pod) (count_over_time({namespace="sample-app"}[5m]))
```

## kubectl — 관측성 스택 확인
```
kubectl -n observability get pods
kubectl -n observability get servicemonitor,podmonitor
kubectl -n observability get prometheusrule
kubectl -n observability get prometheus,alertmanager
kubectl -n observability logs -l app.kubernetes.io/name=promtail --tail=50

# 포트포워딩
kubectl -n observability port-forward svc/kube-prometheus-stack-prometheus 9090:9090
kubectl -n observability port-forward svc/kube-prometheus-stack-grafana 3000:80
kubectl -n observability port-forward svc/kube-prometheus-stack-alertmanager 9093:9093
kubectl -n observability port-forward svc/loki 3100:3100
```

## Prometheus HTTP API
```
curl -s localhost:9090/-/healthy                          # 헬스 체크
curl -s localhost:9090/-/ready
curl -X POST localhost:9090/-/reload                      # 설정 리로드
curl -s 'localhost:9090/api/v1/targets' | jq '.data.activeTargets[] | {job:.labels.job, health}'
curl -s 'localhost:9090/api/v1/query?query=up' | jq
curl -s 'localhost:9090/api/v1/query_range?query=up&start=...&end=...&step=60'
curl -s 'localhost:9090/api/v1/rules' | jq
curl -s 'localhost:9090/api/v1/label/__name__/values' | jq   # 메트릭 이름 전체
curl -s localhost:9090/api/v1/status/tsdb | jq               # 카디널리티 상위 확인
```

## Alertmanager
```
curl -s localhost:9093/api/v2/alerts | jq                 # 현재 알림
curl -s localhost:9093/api/v2/silences | jq               # 사일런스 목록

amtool alert query                                        # CLI
amtool silence add alertname=SampleAppDown -d 2h -c "배포 중"
amtool silence expire <ID>
amtool config routes test severity=critical               # 라우팅 시뮬레이션
```

## 설정 검증
```
promtool check config prometheus.yml                      # 설정 문법 검사
promtool check rules alerts.yaml                          # 룰 문법 검사
promtool test rules test.yaml                             # 룰 단위 테스트
promtool query instant http://localhost:9090 'up'
amtool check-config alertmanager.yml
```

## exporter 직접 확인
```
curl -s localhost:9100/metrics | head -30                 # node-exporter
curl -s localhost:8080/metrics | grep http_requests       # 앱 메트릭
kubectl -n observability exec -it <pod> -- wget -qO- localhost:9090/metrics
```

## Helm (kube-prometheus-stack)
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm show values prometheus-community/kube-prometheus-stack > default-values.yaml
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  -n observability --create-namespace \
  -f values.yaml -f alerts.yaml --version 87.16.1

helm diff upgrade kps prometheus-community/kube-prometheus-stack -f values.yaml
helm -n observability get values kps
```

## 자주 겪는 상황
```
# 타겟이 DOWN 인데 원인을 모를 때
curl -s localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health!="up") | {job:.labels.job, lastError}'

# ServiceMonitor 를 만들었는데 타겟에 안 잡힐 때 → 라벨/포트 이름 확인
kubectl -n observability get servicemonitor <name> -o yaml
kubectl get svc <target-svc> -o jsonpath='{.spec.ports[*].name}'

# 메모리를 잡아먹는 메트릭 찾기 (카디널리티 폭발)
curl -s localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName'

# 알림이 안 오는지, 안 뜨는지 구분
curl -s localhost:9090/api/v1/rules | jq   # Prometheus 에서 firing 인가?
curl -s localhost:9093/api/v2/alerts | jq  # Alertmanager 까지 갔는가?
```
