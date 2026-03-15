---
layout: post
title: "Resilience4j Circuit Breaker, 운영에서 이렇게 적용했다"
date: 2026-03-15 01:40:00 +0900
categories: [backend]
tags: [spring, spring-cloud, resilience4j, circuit-breaker, sre]
thumbnail: /assets/images/thumb-resilience4j-circuit-breaker.svg
---

이번엔 Circuit Breaker를 이론 말고, 실제 적용한 방식 기준으로 정리해본다.

상황은 단순했다.

- 외부 API가 간헐적으로 느려지거나 timeout
- 우리 API 스레드가 기다리다가 묶임
- 지연이 누적되면서 5xx가 같이 증가

그래서 `Resilience4j`를 넣었고, 운영에서 급할 때는 Circuit을 강제로 열고/닫을 수 있게  
`Spring Cloud` 기반 제어를 같이 붙였다.

## 1) 왜 Circuit Breaker가 필요했는지

![Failure cascade without circuit breaker](/assets/images/circuit-breaker-failure-cascade.svg)

Circuit 없을 때는 장애가 아래처럼 번졌다.

1. 다운스트림 timeout 증가
2. 업스트림 대기 스레드 누적
3. 전체 응답 지연 + 에러율 상승

핵심은 "느린 외부 의존성 하나"가 전체 서비스 품질을 같이 끌어내린다는 점이었다.

## 2) 적용 스택

- `spring-cloud-starter-circuitbreaker-resilience4j`
- `spring-boot-starter-actuator`
- `spring-cloud-starter-config` (운영 제어용)

이 조합으로,

- 코드 호출은 Spring Cloud CircuitBreaker API로 감싸고
- 실제 상태 관리는 Resilience4j가 담당하고
- 강제 OPEN/CLOSE는 Spring Cloud Config 값으로 제어했다.

## 3) 기본 설정값 (실무에서 많이 쓰는 시작점)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      orderApi:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 50
        minimumNumberOfCalls: 20
        failureRateThreshold: 50
        slowCallRateThreshold: 60
        slowCallDurationThreshold: 2s
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 10
        automaticTransitionFromOpenToHalfOpenEnabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,refresh
```

![Resilience4j config mapping](/assets/images/resilience4j-config-map.svg)

이 값은 정답이 아니라 시작점이다.  
중요한 건 운영 지표 보면서 계속 튜닝하는 거다.

## 4) 실제 호출 코드 예시

```java
@Service
@RequiredArgsConstructor
public class OrderGateway {

    private final CircuitBreakerFactory<?, ?> circuitBreakerFactory;
    private final RestClient orderClient;

    public OrderResponse getOrder(String orderId) {
        CircuitBreaker cb = circuitBreakerFactory.create("orderApi");

        return cb.run(
                () -> orderClient.get()
                        .uri("/external/orders/{id}", orderId)
                        .retrieve()
                        .body(OrderResponse.class),
                throwable -> OrderResponse.fallback(orderId)
        );
    }
}
```

여기서 포인트는 fallback을 "무조건 성공처럼" 만들지 않는 거다.  
프론트/클라이언트가 fallback 응답임을 구분할 수 있게 설계해야 운영에서 덜 꼬인다.

## 5) Circuit 상태 이해 (운영 기준)

![Resilience4j circuit states](/assets/images/resilience4j-circuit-states.svg)

- `CLOSED`: 정상 호출 + 통계 수집
- `OPEN`: fail-fast, 다운스트림 호출 차단
- `HALF_OPEN`: 제한된 probe 호출로 복구 여부 확인

그리고 운영 제어용으로 `FORCED_OPEN`, `FORCED_CLOSED`를 사용했다.

## 6) 강제 OPEN/CLOSE 제어를 Spring Cloud로 붙인 방식

요구사항이 이거였다.

- 장애 대응 중 운영자가 circuit을 강제로 열거나 닫을 수 있어야 함
- 재배포 없이 반영돼야 함

그래서 아래 흐름으로 구성했다.

![Spring cloud forced control flow](/assets/images/spring-cloud-circuit-force-control.svg)

1. Spring Cloud Config 저장소에 모드 값 관리
2. 운영자가 값 변경 (`AUTO`, `FORCED_OPEN`, `FORCED_CLOSED`)
3. `/actuator/refresh` (또는 Bus refresh) 호출
4. 앱이 갱신된 모드를 읽고 Circuit 상태 전환

### 운영 모드 프로퍼티

```yaml
ops:
  circuit:
    order-api:
      mode: AUTO # AUTO | FORCED_OPEN | FORCED_CLOSED
```

### RefreshScope + 상태 동기화 코드

```java
@Getter
@Setter
@RefreshScope
@Component
@ConfigurationProperties(prefix = "ops.circuit.order-api")
public class CircuitControlProperties {
    private Mode mode = Mode.AUTO;

    public enum Mode {
        AUTO,
        FORCED_OPEN,
        FORCED_CLOSED
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class CircuitStateSynchronizer {

    private final CircuitBreakerRegistry registry;
    private final CircuitControlProperties properties;

    @Scheduled(fixedDelayString = "${ops.circuit.sync-delay-ms:3000}")
    public void syncState() {
        io.github.resilience4j.circuitbreaker.CircuitBreaker cb =
                registry.circuitBreaker("orderApi");

        switch (properties.getMode()) {
            case FORCED_OPEN -> cb.transitionToForcedOpenState();
            case FORCED_CLOSED -> cb.transitionToClosedState();
            case AUTO -> cb.reset();
        }
    }
}
```

`FORCED_CLOSED`는 보호막을 우회하는 동작이라 진짜 짧게만 써야 한다.  
장애 대응에선 보통 `FORCED_OPEN`이 더 자주 필요했다.

## 7) 운영에서 실제로 본 지표

- circuit state 전환 빈도
- fallback 비율
- 실패/느린 호출 비율

![Circuit breaker metrics dashboard](/assets/images/resilience4j-metrics-dashboard.svg)

이 지표를 안 보면 "설정 넣었으니 끝"이 되는데,  
실제로는 상태 전환 패턴을 보고 threshold를 계속 조정해야 한다.

## 8) 롤아웃할 때 체크한 순서

![Circuit breaker rollout plan](/assets/images/circuit-breaker-rollout-plan.svg)

1. 관측부터 붙이고(상태/실패/느린 호출)
2. fallback 응답 품질 검증하고
3. 온콜 runbook 정리하고
4. 강제 제어 모드(OPEN/CLOSED)까지 운영 절차에 포함

---

정리하면 Circuit Breaker는 라이브러리 하나 추가하는 작업이 아니라,

- 장애 전파를 끊는 런타임 장치
- 운영자가 제어할 수 있는 대응 장치

이 두 개를 같이 만드는 작업이었다.

특히 `Spring Cloud Config + refresh`로 강제 상태 제어를 붙여두면,  
긴급 대응 속도가 확실히 빨라진다.

## References

- Spring Cloud Circuit Breaker Reference: https://docs.spring.io/spring-cloud-circuitbreaker/reference/
- Resilience4j CircuitBreaker Guide: https://resilience4j.readme.io/docs/circuitbreaker
- Spring Cloud Config Reference: https://docs.spring.io/spring-cloud-config/docs/current/reference/html/
- Spring Cloud Commons Refresh Scope: https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/application-context-services.html#refresh-scope
