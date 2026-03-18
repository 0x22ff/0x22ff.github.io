---
layout: post
title: "Retry는 어디까지 허용해야 하나: Resilience4j Retry + Timeout 운영기"
date: 2026-03-18 10:20:00 +0900
categories: [backend]
tags: [spring, resilience4j, retry, timeout, sre]
thumbnail: /assets/images/thumb-retry-timeout-resilience4j.svg
---

Circuit Breaker 붙이고 나서도 한동안 장애가 깔끔하게 안 잡힌 적이 있었다.

원인은 이거였다.

- timeout이 길어서 스레드가 오래 묶이고
- retry가 무심코 많이 돌면서 트래픽을 더 키우고
- 결과적으로 복구가 늦어짐

그래서 `Retry + Timeout` 정책을 따로 정리해서 운영 룰로 붙였다.

## 1) 정책 없을 때 어떤 문제가 터지나

![Failure pattern without retry-timeout policy](/assets/images/retry-timeout-failure-pattern.svg)

문제 패턴은 대부분 비슷했다.

1. 다운스트림이 느려짐
2. 업스트림은 기다림 + 재시도
3. 전체 지연/에러가 같이 상승

핵심은 retry 자체가 나쁘다는 게 아니라,  
**경계 없이 retry를 걸면 장애를 증폭시킬 수 있다**는 점이다.

## 2) 내가 잡은 기본 원칙

- retry는 "일시 장애"에만 허용
- timeout은 요청 1회당 상한을 명확히 둠
- retry 대상은 idempotent 요청 위주로 제한
- 4xx/비즈니스 예외는 fail-fast

![Retry decision matrix](/assets/images/retry-decision-matrix.svg)

이 원칙 먼저 정리해두면 설정값 튜닝이 훨씬 쉬워진다.

## 3) Resilience4j 기본 설정 예시

(아래 값은 실무 시작점 예시)

```yaml
resilience4j:
  retry:
    instances:
      orderApi:
        maxAttempts: 3
        waitDuration: 200ms
        retryExceptions:
          - java.net.SocketTimeoutException
          - java.io.IOException
        ignoreExceptions:
          - com.bssj.common.exception.ValidationException
  timelimiter:
    instances:
      orderApi:
        timeoutDuration: 800ms
        cancelRunningFuture: true
```

포인트:

- `maxAttempts`는 작게 시작 (`2~3` 정도)
- `waitDuration`은 짧고 고정적으로 시작한 뒤, 필요하면 백오프 도입
- `timeoutDuration`은 SLO/다운스트림 특성 보고 조정

## 4) 실제 호출 흐름 (Timeout + Retry)

![Retry timeout call flow](/assets/images/retry-timeout-call-flow.svg)

아래는 `CompletionStage` 기반으로 `TimeLimiter` + `Retry`를 함께 쓰는 예시다.

```java
@Service
@RequiredArgsConstructor
public class OrderFacade {

    private final TimeLimiterRegistry timeLimiterRegistry;
    private final RetryRegistry retryRegistry;
    private final ExternalOrderClient externalOrderClient;

    public CompletionStage<OrderResponse> getOrder(String orderId) {
        TimeLimiter timeLimiter = timeLimiterRegistry.timeLimiter("orderApi");
        Retry retry = retryRegistry.retry("orderApi");

        Supplier<CompletionStage<OrderResponse>> originalCall =
                () -> externalOrderClient.getOrderAsync(orderId);

        Supplier<CompletionStage<OrderResponse>> withTimeout =
                TimeLimiter.decorateCompletionStage(timeLimiter, () -> originalCall.get());

        Supplier<CompletionStage<OrderResponse>> withRetry =
                Retry.decorateCompletionStage(retry, withTimeout);

        return withRetry.get()
                .exceptionally(ex -> OrderResponse.fallback(orderId));
    }
}
```

fallback 응답은 "정상 데이터"와 구분 가능하게 만드는 게 중요하다.

## 5) 재시도 대상/비대상은 코드로 고정

retry가 위험해지는 지점은 "모든 예외를 다 재시도"할 때다.

실무에서 내가 고정한 룰:

- 재시도 대상: 네트워크 단절, timeout, 일시적인 5xx
- 재시도 제외: validation/business 예외, 명백한 client 입력 오류(4xx)
- 쓰기 요청은 idempotency key 없는 경우 재시도 신중

## 6) 관측 지표는 이 3개부터

- retry 성공 비율 (`successful_with_retry`)
- timeout 발생 비율
- fallback 비율

![Retry observability dashboard](/assets/images/retry-observability-dashboard.svg)

이 세 개만 제대로 봐도,

- retry가 실제로 복구에 기여하는지
- 오히려 증폭시키는지

판단이 가능해진다.

## 7) 운영 중 동적 제어 (Spring Cloud)

긴급 상황에서 retry 강도를 바로 낮춰야 할 때가 있다.  
그럴 때는 Spring Cloud Config + refresh 흐름이 실전에서 꽤 유용했다.

1. Config 값 변경 (예: `maxAttempts: 3 -> 1`)
2. `/actuator/refresh` 또는 `/actuator/busrefresh` 호출
3. `@RefreshScope` 빈 재바인딩으로 런타임 반영

즉, 재배포 없이 대응 속도를 올릴 수 있다.

## 8) 롤아웃 체크리스트

![Retry rollout checklist](/assets/images/retry-rollout-checklist.svg)

1. idempotency 여부 먼저 확인
2. retry/ignore 예외 분리
3. timeout을 SLO 내에서 설정
4. fallback payload 계약 검증
5. 알람/런북까지 붙이고 배포

---

정리하면 Retry + Timeout은 "성능 옵션"이 아니라,  
**장애 증폭을 막기 위한 운영 정책**에 가깝다.

값 몇 개 넣는 것보다,

- 어디를 retry할지
- 어디서 끊을지
- 어떤 지표로 튜닝할지

이 세 가지를 팀 규칙으로 고정하는 게 더 중요했다.

## References

- Resilience4j Retry Guide: https://resilience4j.readme.io/docs/retry
- Resilience4j TimeLimiter Guide: https://resilience4j.readme.io/docs/timeout
- Resilience4j Spring Boot Getting Started: https://resilience4j.readme.io/docs/getting-started-3
- Spring Cloud Config Reference: https://docs.spring.io/spring-cloud-config/docs/current/reference/html/
- Spring Boot Actuator Endpoints: https://docs.spring.io/spring-boot/reference/actuator/endpoints.html
