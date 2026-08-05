
# Observability Labs

Prometheus·Grafana·Loki로 시스템을 관측하는 학습 실습 및 명령어 정리 저장소

Terraform으로 인프라를 만들고 Kubernetes로 앱을 띄웠다면, 다음 질문은 **"지금 잘 돌고 있는가"** 다.
장애는 "서버가 죽었다"보다 "왜인지 모르겠는데 느리다" 형태로 온다. 관측성은 그 "왜"에 답할 수 있게 시스템을 계측해두는 일이다.

## 구조
- `commands.md` - PromQL·LogQL·kubectl·promtool 레퍼런스
- `01-observability-basics/` - 모니터링 vs 관측성, 3요소(메트릭·로그·트레이스), SLI/SLO/에러버짓, USE·RED
- `02-prometheus-architecture/` - Pull 모델, 데이터 모델·메트릭 타입, exporter, 서비스 디스커버리, TSDB·리텐션
- `03-promql/` - 셀렉터, rate·increase, 집계 연산자, histogram_quantile, 레코딩 룰
- `04-grafana/` - 데이터소스, 패널·쿼리, 변수·템플릿, 프로비저닝(Dashboard as Code)
- `05-alerting/` - PrometheusRule, for·severity, Alertmanager 라우팅·그룹핑·억제·사일런스, 알림 피로
- `06-logging/` - Loki·promtail(PLG), 라벨 설계와 카디널리티, LogQL, EFK와 비교
- `07-tracing/` - 분산 추적, OpenTelemetry, 컨텍스트 전파, 샘플링, Tempo·Jaeger, exemplar
- `08-kube-prometheus-stack/` - Operator와 CRD(ServiceMonitor·PrometheusRule), Helm values, EKS 실전 적용
