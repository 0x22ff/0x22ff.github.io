---
layout: post
title: "Resilience4j Retry + TimeLimiter, 운영에서 안전하게 붙이는 방법"
date: 2026-04-01 13:20:00 +0900
categories: [backend]
tags: [spring, resilience4j, retry, timelimiter, timeout, sre]
thumbnail: /assets/images/thumb-retry-timeout-resilience4j.svg
---

이번엔 retry를 "한 번 더 해보자" 수준이 아니라, 실제 운영에서 안전하게 붙인 기준으로 정리해본다.

상황은 비슷했다.

- 다운스트림 API가 간헐적으로 timeout
- 한 번 실패했다고 바로 장애로 볼 정도는 아님
- 그런데 retry를 크게 걸면 호출 수만 늘어나서 장애가 더 커짐

그래서 `Retry`만 따로 쓰지 않고,  
`TimeLimiter` + 예외 정책 + 관측을 같이 묶어서 적용했다.

## 1) 왜 Retry만 단독으로 붙이면 위험한가

![Failure pattern before retry timeout policy](/assets/images/retry-timeout-failure-pattern.svg)

문제는 retry 자체가 아니다.  
**느린 호출에 retry를 겹치면, 복구보다 증폭이 먼저 온다**는 점이다.

운영에서 실제로 보인 흐름은 이랬다.

1. 다운스트림 지연/timeout 증가
2. 우리 쪽 대기 시간이 길어짐
3. 같은 요청에 재시도까지 붙으면서 총 호출 수 증가
4. 지연과 에러가 같이 커짐

즉 retry는 보호막이 될 수도 있지만, 설정을 잘못 잡으면 트래픽 증폭기가 되기도 한다.

## 2) 내가 잡은 기본 원칙: 시도마다 timeout 상한을 먼저 둔다

![Retry and timeout call flow](/assets/images/retry-timeout-call-flow.svg)

내 기준은 단순했다.

- 각 시도마다 `timeoutDuration`으로 상한 시간을 먼저 고정
- retry는 정말 재시도할 가치가 있는 실패에만 제한적으로 허용
- 전체 요청 지연이 SLO budget을 넘지 않게 계산

여기서 중요한 건 **attempt 단위 timeout**이다.  
timeout 없이 retry만 있으면, 느린 호출이 길게 붙잡힌 뒤 다시 재시도되면서 전체 응답 시간이 쉽게 무너진다.

예를 들어:

- `maxAttempts: 3`
- `waitDuration: 200ms`
- `timeoutDuration: 800ms`

이 설정이면 최악 지연은 대략 `800 * 3 + 200 * 2 = 2.8s`까지 갈 수 있다.  
숫자만 보면 작아 보여도, API SLO가 1초대면 이미 예산을 넘긴다.

## 3) 설정은 먼저 보수적으로 시작했다

`application.yml` 예시는 아래처럼 시작했다.

```yaml
resilience4j:
  retry:
    instances:
      partnerApi:
        maxAttempts: 3
        waitDuration: 200ms
        retryExceptions:
          - java.net.SocketTimeoutException
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.example.api.ValidationException
          - org.springframework.web.client.HttpClientErrorException$BadRequest

  timelimiter:
    instances:
      partnerApi:
        timeoutDuration: 800ms
        cancelRunningFuture: true
```

포인트는 이 정도였다.

- `maxAttempts`는 보통 `2` 또는 `3`부터 시작
- `waitDuration`은 짧게 두고, 필요하면 backoff로 확장
- `ignoreExceptions`로 4xx/비즈니스 오류는 fail-fast
- `cancelRunningFuture: true`로 timeout 이후 남는 작업을 최대한 줄임

코드 예시는 `TimeLimiter` 때문에 `CompletableFuture` 반환으로 잡았다.

```java
@Service
public class PartnerOrderClient {

    private final Executor ioExecutor;
    private final ExternalPartnerClient externalPartnerClient;

    public PartnerOrderClient(@Qualifier("partnerIoExecutor") Executor ioExecutor,
                              ExternalPartnerClient externalPartnerClient) {
        this.ioExecutor = ioExecutor;
        this.externalPartnerClient = externalPartnerClient;
    }

    @Retry(name = "partnerApi", fallbackMethod = "getOrderFallback")
    @TimeLimiter(name = "partnerApi")
    public CompletableFuture<OrderResponse> getOrder(String orderId) {
        return CompletableFuture.supplyAsync(
                () -> externalPartnerClient.getOrder(orderId),
                ioExecutor
        );
    }

    private CompletableFuture<OrderResponse> getOrderFallback(String orderId, Throwable t) {
        return CompletableFuture.completedFuture(OrderResponse.fallback(orderId));
    }
}
```

여기서 한 가지는 꼭 기억해야 한다.  
Spring annotation 방식을 쓰면 aspect order 영향을 받는다.

기본적으로 `Retry`가 바깥에서 감싸기 때문에,  
**각 retry attempt가 안쪽 `TimeLimiter` 제한을 받는 구조**로 이해하면 된다.

기본 순서가 애매하거나 조합을 더 세밀하게 제어하고 싶으면, annotation보다 decorator 방식이 덜 헷갈린다.

## 4) 어떤 실패만 retry 대상으로 둘지 먼저 고정해야 한다

![Retry decision matrix](/assets/images/retry-decision-matrix.svg)

운영에서 기준은 아주 보수적으로 잡는 게 낫다.

retry 후보:

- 네트워크 timeout
- 다운스트림 일시적 5xx
- connection reset 같은 일시 장애

fail-fast 후보:

- validation/business 오류(주로 4xx)
- idempotency key 없는 쓰기 요청
- 원인이 명확한 애플리케이션 예외

여기서 제일 많이 실수하는 게 쓰기 요청이다.

- 조회성 GET
- 멱등성이 보장된 PUT
- `idempotency key`가 있는 결제/주문 요청

이 정도가 아니면 retry는 더 조심해야 한다.  
POST를 아무 생각 없이 재시도하면 중복 처리 사고로 바로 이어질 수 있다.

## 5) 설정보다 더 중요한 건 관측이다

![Retry timeout observability dashboard](/assets/images/retry-observability-dashboard.svg)

Retry를 붙인 뒤 최소로 본 건 4가지였다.

- retry 후 성공 비율
- timeout 비율
- fallback 비율
- 전체 응답 지연(p95/p99)

여기서 중요한 건 "retry 성공이 많다 = 좋은 설정"이 아니라는 점이다.  
retry 후 성공이 늘어도 timeout 비율이나 전체 latency가 같이 올라가면,  
실제로는 장애를 길게 끌고 있을 가능성이 크다.

내가 본 기준은 이랬다.

- retry 성공 비율은 오르는데 p95가 같이 튄다 -> wait/attempt 과다 가능성
- timeout 비율이 빠르게 늘어난다 -> 다운스트림 자체 문제 먼저 확인
- fallback 비율이 높다 -> 사용자 영향 범위를 같이 점검
- 배포 직후 retry 호출 수가 급증한다 -> 즉시 canary 범위 재검토

관측 없이 retry를 켜면 "실패가 줄어든 것처럼 보이는 착시"가 자주 생긴다.

## 6) 롤아웃은 체크리스트를 고정해두는 게 낫다

![Retry timeout rollout checklist](/assets/images/retry-rollout-checklist.svg)

나는 순서를 거의 고정했다.

1. idempotency 먼저 확인
2. retry/ignore exception 정책 확정
3. 메트릭과 알람부터 붙임
4. canary로 일부 트래픽만 적용
5. 이상 없을 때만 점진 확대

특히 `maxAttempts`와 `timeoutDuration`은  
SLO budget 안에 들어오는지 먼저 계산해두는 게 좋다.

retry는 실패를 가리는 기능이 아니라,  
**일시 장애를 작은 비용으로 넘기는 기능**이어야 한다.

## 7) 자주 헷갈렸던 포인트

### Q. timeout을 늘리면 retry는 줄여야 하나?

대부분은 그렇다.  
전체 지연 예산은 한정돼 있으니, 한쪽을 늘리면 다른 쪽은 줄이는 게 맞다.

### Q. fallback이 있으면 성공으로 봐도 되나?

아니다. fallback은 대체 응답이지, 원래 호출이 건강했다는 뜻은 아니다.  
운영 지표에서도 fallback 비율은 따로 봐야 한다.

### Q. retry에 exponential backoff를 바로 넣는 게 좋나?

바로 넣기보다, 먼저 fixed wait로 작게 시작하는 쪽이 해석이 쉽다.  
장애 패턴이 확인된 뒤에 backoff/jitter를 추가하는 게 운영에서 덜 꼬였다.

### Q. 쓰기 요청에도 retry를 걸 수 있나?

가능은 하지만, 멱등성 보장 전략이 먼저다.  
`idempotency key`나 중복 방지 설계 없이는 매우 조심해야 한다.

---

정리하면 Retry는 "한 번 더 시도해주는 친절한 옵션"이 아니라,

- timeout 상한을 먼저 두고
- retry할 실패를 좁게 고르고
- 관측과 롤아웃까지 같이 설계해야 하는

운영 장치에 가깝다.

특히 `Retry + TimeLimiter`를 같이 보면  
"실패를 조금 덜 보이게 만드는 설정"이 아니라,  
**전체 지연과 호출 증폭을 통제하는 설정**으로 보게 된다.

## References

- Resilience4j Retry Guide: https://resilience4j.readme.io/docs/retry
- Resilience4j TimeLimiter Guide: https://resilience4j.readme.io/docs/timeout
- Resilience4j Spring Boot Getting Started: https://resilience4j.readme.io/v2.0.0/docs/getting-started-3
- Spring Boot Actuator Metrics: https://docs.spring.io/spring-boot/reference/actuator/metrics.html
