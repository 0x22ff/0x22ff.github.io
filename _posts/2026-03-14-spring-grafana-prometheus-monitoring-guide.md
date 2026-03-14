---
layout: post
title: "Spring + Java21에서 Grafana + Prometheus 운영 모니터링 가이드"
date: 2026-03-14 09:40:00 +0900
categories: [backend]
tags: [spring, java21, prometheus, grafana, observability]
thumbnail: /assets/images/thumb-grafana-prometheus-guide.svg
---

서비스를 운영하다 보면 결국 같은 질문으로 모입니다.

- 지금 시스템은 정상인가?
- 느려졌다면 어디서부터 봐야 하나?
- 장애를 몇 분 안에 감지할 수 있나?

이 글은 Spring Boot(Java 21) 서비스 기준으로, **Prometheus로 수집하고 Grafana로 보는 운영 모니터링 기본 구조**를 한 번에 정리한 가이드입니다.

핵심 범위는 아래 네 가지입니다.

- 메트릭 노출(Actuator + Micrometer)
- 운영 환경에서 `/actuator` 안전하게 열기
- Prometheus scrape 설정
- Grafana 대시보드/알람 구성

## 1) 먼저 큰 그림

![Monitoring architecture overview](/assets/images/grafana-prometheus-overview.svg)

흐름은 단순합니다.

1. Spring Boot 앱이 `/actuator/prometheus`로 메트릭을 노출
2. Prometheus가 주기적으로 scrape
3. Grafana가 Prometheus를 데이터 소스로 조회
4. Grafana Alerting으로 임계치 경보 발행

여기서 중요한 건 "도구를 많이 붙이는 것"보다 **초기 지표를 작게, 명확하게 시작하는 것**입니다.

## 2) Spring Boot(Java 21)에서 메트릭 노출

### 2-1. 의존성 추가

```gradle
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    runtimeOnly("io.micrometer:micrometer-registry-prometheus")
}
```

### 2-2. endpoint 노출 설정

```yaml
management:
  server:
    port: 18081
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

`management.server.port`를 분리해두면 운영 보안 정책을 적용하기가 훨씬 쉽습니다.

### 2-3. 커스텀 메트릭 예시

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

운영에서 자주 놓치는 포인트는 태그(cardinality)입니다.

- `userId`, `requestId`, raw URL 같은 무한 증가 태그는 피하기
- `method`, `status`, `uri`처럼 제한된 집합의 태그 사용

카디널리티가 커지면 Prometheus 메모리 사용량과 쿼리 비용이 빠르게 증가합니다.

## 3) 운영 보안: `/actuator`는 내부망 전용

운영 환경에서 `/actuator`를 퍼블릭으로 열어두는 건 피하는 게 맞습니다.

권장 패턴:

- 앱 포트와 관리 포트 분리
- 보안그룹(Security Group)에서 Prometheus 서버(또는 내부 SG)만 허용
- 외부 인터넷에서 management 포트 차단
- endpoint 최소 공개(`prometheus`, `health`, `info`)

![Actuator private access](/assets/images/actuator-private-access.svg)

애플리케이션 레벨 IP 필터만으로 끝내기보다, **네트워크 레벨에서 먼저 막는 설계**가 더 안전합니다.

## 4) Prometheus 수집 설정

아래는 시작하기 좋은 기본 예시입니다.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "bssj-web-backend"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["host.docker.internal:18081"]
```

![Prometheus scrape pipeline](/assets/images/prometheus-scrape-pipeline.svg)

참고:
- `host.docker.internal`은 로컬/도커 개발 예시입니다.
- EC2/ECS/K8s 운영에서는 내부 DNS나 서비스 디스커버리 경로로 바꿔야 합니다.

운영 팁:

- 처음은 `15s` 또는 `30s`로 시작
- retention/디스크 사용량 같이 모니터링
- 반복적으로 무거운 쿼리는 recording rule 고려

## 5) Grafana 대시보드: 처음부터 크게 만들지 않기

초기 대시보드는 아래 4개 축이면 충분합니다.

- Request Rate (RPS)
- Error Rate (5xx 비율)
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

운영하면서 패널을 늘리는 기준은 간단합니다.  
"문제가 생겼을 때 실제로 원인 좁히기에 도움이 되는가"만 보시면 됩니다.

## 6) 알람 설계: 임계치보다 노이즈 관리가 더 중요

예시 룰(지연 p95 700ms 초과가 10분 지속):

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

알람 품질을 높이려면 아래를 같이 챙기는 게 좋습니다.

- `for` 시간으로 순간 스파이크 노이즈 줄이기
- `severity` 라벨 기준으로 채널 분리
- `runbook` 링크를 annotations에 넣기
- 경보 후 조치 완료까지의 평균 시간을 추적

## 7) 운영 체크리스트

처음 구축 후 바로 점검할 체크리스트입니다.

1. Prometheus가 실제로 scrape 성공하는가
2. Grafana 대시보드 수치가 앱 로그와 방향성이 맞는가
3. 알람 테스트를 수동으로 한 번 발생시켜보았는가
4. `/actuator`가 외부에서 차단되어 있는가
5. 장애 대응 문서(runbook) 링크가 알람 payload에 포함되는가

![Ops loop](/assets/images/monitoring-ops-loop.svg)

## 마무리

Prometheus + Grafana 조합의 장점은 복잡한 기능보다 **운영 기준선을 빠르게 세울 수 있다는 점**입니다.

- 어떤 지표를 계속 볼지
- 어느 지점에서 알람을 울릴지
- 알람 후 무엇을 확인할지

이 세 가지를 팀 기준으로 정해두면, 장애 대응 속도와 품질이 안정적으로 올라갑니다.
