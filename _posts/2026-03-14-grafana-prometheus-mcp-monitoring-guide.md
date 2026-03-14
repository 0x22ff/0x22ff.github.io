---
layout: post
title: "Spring + Java21에서 Grafana + Prometheus 모니터링, 그리고 MCP 자동 분석까지"
date: 2026-03-14 09:40:00 +0900
categories: [backend]
tags: [spring, java21, prometheus, grafana, mcp, observability]
thumbnail: /assets/images/thumb-grafana-prometheus-mcp.svg
---

서비스가 커지면 장애가 안 나는 게 목표가 아니라, **장애를 빨리 감지하고 빨리 원인을 좁히는 것**이 목표가 됩니다.

이번 글은 이 흐름을 한 번에 잡는 구성입니다.

- Spring Boot(Java 21)에서 메트릭 노출
- Prometheus로 수집
- Grafana로 시각화 + 알람
- 알람 이벤트를 받아 MCP 기반 분석 리포트 자동화

중요한 포인트 하나만 먼저 짚고 갈게요.

`Grafana -> MCP`가 기본 내장으로 바로 붙는 구조는 아닙니다.  
보통은 **Webhook 수신 워커**를 하나 두고, 그 워커가 MCP 클라이언트로 분석을 수행하는 방식으로 구현합니다.

## 1) 전체 구조 먼저 보기

![Monitoring architecture overview](/assets/images/grafana-prometheus-overview.svg)

흐름은 단순합니다.

1. 앱이 메트릭을 노출한다.
2. Prometheus가 주기적으로 긁어간다(scrape).
3. Grafana가 시각화하고 임계치 기반으로 알람을 만든다.
4. Webhook으로 전달된 알람을 워커가 받아 MCP 기반 분석을 돌린다.
5. Slack/메일/이슈 트래커로 "요약 + 근거 + 다음 액션"을 보낸다.

## 2) Spring Boot(Java 21)에서 메트릭 내보내기

### 의존성

```gradle
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    runtimeOnly("io.micrometer:micrometer-registry-prometheus")
}
```

### actuator endpoint 노출

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

### 커스텀 메트릭 예시

```java
@Component
public class SermonMetrics {

    private final Counter sermonCreateCounter;
    private final Timer sermonPublishTimer;

    public SermonMetrics(MeterRegistry registry) {
        this.sermonCreateCounter = Counter.builder("cms_sermon_create_total")
                .description("Total created sermons")
                .register(registry);
        this.sermonPublishTimer = Timer.builder("cms_sermon_publish_seconds")
                .description("Sermon publish latency")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);
    }

    public void incrementCreate() {
        sermonCreateCounter.increment();
    }

    public <T> T recordPublish(Supplier<T> supplier) {
        return sermonPublishTimer.record(supplier);
    }
}
```

![Micrometer flow](/assets/images/spring-boot-micrometer-flow.svg)

실무에서 가장 먼저 챙길 건 태그(tag) 폭발 방지입니다.

- `userId`, `requestId`, raw URL 같은 고카디널리티 태그는 피하기
- endpoint, status, method처럼 제한된 값만 태그로 사용하기

## 3) Prometheus 수집 붙이기

`prometheus.yml` 예시:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "bssj-web-backend"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["host.docker.internal:8080"]
```

![Prometheus scrape pipeline](/assets/images/prometheus-scrape-pipeline.svg)

운영 팁:

- scrape 주기는 10~30초 사이에서 시작
- 고비용 쿼리는 recording rule로 미리 계산
- retention 기간/디스크 사용량을 같이 관리

## 4) Grafana 대시보드 구성

처음부터 패널을 많이 만들 필요는 없습니다.  
아래 4개만 있어도 장애 체감이 확 줄어요.

- Request Rate (RPS)
- Error Rate
- Latency p95/p99
- JVM/CPU/메모리 + DB pool

![Grafana dashboard mock](/assets/images/grafana-dashboard-sample.svg)

자주 쓰는 PromQL 예시:

```promql
sum(rate(http_server_requests_seconds_count{uri!="/actuator/prometheus"}[5m]))
```

```promql
histogram_quantile(
  0.95,
  sum(rate(http_server_requests_seconds_bucket[5m])) by (le)
)
```

```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))
```

## 5) 알람(Webhook) 연결

예를 들어 "p95 지연이 10분 이상 700ms 초과" 같은 룰을 둡니다.

```yaml
groups:
  - name: latency-alerts
    rules:
      - alert: ApiLatencyP95High
        expr: histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le)) > 0.7
        for: 10m
        labels:
          severity: warning
          service: bssj-web-backend
        annotations:
          summary: "API latency p95 is above 700ms"
          runbook: "https://internal.wiki/runbooks/api-latency"
```

![Alert webhook flow](/assets/images/grafana-alert-webhook-flow.svg)

그리고 Contact point를 Webhook으로 잡아 워커 엔드포인트에 전달합니다.

- `POST https://ops.example.com/alerts/grafana`

## 6) MCP로 자동 조사 리포트 만들기

여기가 이번 글의 핵심입니다.

알람이 오면 워커가 다음 순서로 움직입니다.

1. 알람 레이블(`service`, `severity`, `startsAt`) 파싱
2. 최근 15~30분 메트릭/로그/배포 이력 수집
3. MCP 툴 체인으로 원인 후보 정리
4. Slack/이슈 트래커에 리포트 전송

![MCP investigation loop](/assets/images/mcp-investigation-loop.svg)

워커의 의사코드는 이런 형태입니다.

```text
onWebhook(alert):
  context = collectMetricsAndLogs(alert.service, alert.startsAt)
  analysis = mcpClient.analyze({ alert, context })
  report = renderReport(analysis)
  sendToSlack(report)
```

리포트 템플릿은 아래 4개를 고정하면 팀 커뮤니케이션이 빨라집니다.

- What happened
- Suspected causes (with evidence)
- Immediate actions (10~30분 내)
- Follow-up actions (재발 방지)

## 7) 운영 루프는 이렇게 굴리면 된다

![Ops loop](/assets/images/monitoring-ops-loop.svg)

알람은 쌓이기 쉬워서, 결국 "루프"를 만들어야 합니다.

- Measure: 지표 정의/정제
- Alert: 노이즈 제거, 임계치 튜닝
- Analyze: MCP 기반 자동 조사
- Improve: 룰/대시보드/코드 개선

이 루프가 돌아가면 장애 대응이 점점 빨라집니다.

## 마무리

정리하면,

- Prometheus/Grafana는 감지와 가시화 담당
- MCP 분석 워커는 원인 좁히기와 보고 자동화 담당

둘을 붙이면 "알람이 울렸다"에서 끝나지 않고,  
**"왜 울렸고 지금 뭘 해야 하는지"까지 자동으로 전달되는 운영 체계**를 만들 수 있습니다.

다음 단계로는 SLO 기반 경보(에러 버짓)까지 붙이면 훨씬 단단해집니다.
