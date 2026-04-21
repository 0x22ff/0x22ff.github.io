---
layout: post
title: "Resilience4j Bulkhead, 외부 API 장애를 서비스 전체 장애로 번지지 않게 막는 방법"
date: 2026-04-21 18:20:00 +0900
categories: [backend]
tags: [spring, resilience4j, bulkhead, threadpool, sre]
thumbnail: /assets/images/thumb-resilience4j-bulkhead.svg
---

이번엔 `Bulkhead`를 정리해본다.

앞에서 `Circuit Breaker`, `Retry + TimeLimiter`, `RateLimiter`를 봤다면  
Bulkhead는 그 사이에서 조금 덜 화려하지만 운영에서는 꽤 중요한 역할을 한다.

상황은 이런 식이었다.

- 외부 API 하나가 느려짐
- 그 API를 호출하는 요청이 계속 쌓임
- 애플리케이션의 공용 스레드나 커넥션이 같이 묶임
- 결국 관련 없는 기능까지 느려짐

Circuit Breaker가 "이미 나빠진 의존성을 끊는 장치"라면,  
Bulkhead는 **나쁜 의존성이 잡아먹을 수 있는 자원 자체를 제한하는 장치**에 가깝다.

## 1) 왜 Bulkhead가 필요했는지

![Bulkhead failure isolation](/assets/images/bulkhead-failure-isolation.svg)

문제는 외부 API 장애가 항상 에러로만 오지 않는다는 점이다.

오히려 운영에서 더 무서운 건 "느린 성공"이었다.

1. 외부 API 응답이 200ms에서 3초로 늘어남
2. 우리 API는 계속 기다림
3. 대기 중인 요청이 늘면서 스레드/커넥션 점유 시간이 길어짐
4. 관련 없는 요청까지 영향을 받음

Circuit Breaker가 열리기 전까지는 어느 정도 호출이 흘러간다.  
Retry가 붙어 있으면 호출 수가 더 늘 수도 있다.

그래서 외부 API별로 "이 의존성이 동시에 가져갈 수 있는 최대 자원"을 먼저 정해두는 게 필요했다.

## 2) Bulkhead는 무엇을 막는가

Bulkhead는 선박의 격벽에서 나온 패턴이다.  
한 구역에 물이 들어와도 배 전체가 바로 가라앉지 않도록 구역을 나누는 구조다.

애플리케이션에서는 이렇게 해석했다.

- 결제 API가 느려져도 상품 조회 스레드까지 잡아먹지 않게 한다
- 알림 API가 막혀도 주문 생성 요청이 같이 밀리지 않게 한다
- 특정 파트너 API가 포화돼도 전체 요청 처리량을 보호한다

즉 Bulkhead의 핵심은 "성공률을 높이는 것"이 아니라,  
**장애 영향 반경을 작게 유지하는 것**이다.

## 3) SemaphoreBulkhead와 ThreadPoolBulkhead

![Semaphore bulkhead vs thread pool bulkhead](/assets/images/bulkhead-threadpool-vs-semaphore.svg)

Resilience4j에는 크게 두 가지 방식이 있다.

- `SemaphoreBulkhead`: 현재 실행 흐름은 유지하고 동시 실행 수만 제한
- `ThreadPoolBulkhead`: 별도 thread pool과 bounded queue로 실행 영역을 분리

둘의 차이를 간단히 잡으면 이렇다.

| 구분 | SemaphoreBulkhead | ThreadPoolBulkhead |
| --- | --- | --- |
| 제한 기준 | 동시 실행 permit | thread pool + queue |
| 실행 thread | 호출한 쪽 thread | 별도 bulkhead thread |
| 주요 설정 | `maxConcurrentCalls`, `maxWaitDuration` | `coreThreadPoolSize`, `maxThreadPoolSize`, `queueCapacity` |
| 장점 | 단순하고 오버헤드가 작음 | 느린 I/O를 별도 풀로 격리하기 쉬움 |
| 주의점 | 호출 thread가 그대로 대기할 수 있음 | queue를 크게 잡으면 장애를 늦게 발견함 |

내 기준은 이랬다.

- 이미 비동기/논블로킹 모델이고 동시성만 제한하면 되면 `SemaphoreBulkhead`
- 블로킹 외부 API 호출을 별도 풀로 격리해야 하면 `ThreadPoolBulkhead`

Spring Cloud CircuitBreaker Resilience4j를 같이 쓰는 경우에는 기본 동작과 적용 범위를 꼭 확인해야 한다.  
`resilience4j-bulkhead`가 classpath에 있으면 메서드가 Bulkhead로 감싸질 수 있고, 기본 Bulkhead 타입도 설정에 따라 달라진다.

## 4) 설정은 작게 시작하는 게 낫다

Semaphore 방식은 아래처럼 시작했다.

```yaml
resilience4j:
  bulkhead:
    instances:
      partnerApi:
        maxConcurrentCalls: 20
        maxWaitDuration: 0
```

여기서 `maxWaitDuration: 0`은 permit이 없으면 기다리지 않고 바로 거절한다는 의미로 잡았다.  
운영에서는 대기열을 길게 두는 것보다 빠르게 거절하고 fallback/에러 응답으로 전환하는 쪽이 더 해석하기 쉬웠다.

ThreadPool 방식은 별도 설정을 쓴다.

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      partnerApi:
        coreThreadPoolSize: 8
        maxThreadPoolSize: 16
        queueCapacity: 20
        keepAliveDuration: 20ms
```

중요한 건 두 설정이 서로 대체 관계가 아니라는 점이다.

- `resilience4j.bulkhead`는 SemaphoreBulkhead 설정
- `resilience4j.thread-pool-bulkhead`는 ThreadPoolBulkhead 설정

`maxConcurrentCalls`를 ThreadPoolBulkhead에 넣어놓고 왜 안 먹는지 찾는 식의 실수가 생기기 쉽다.

## 5) 코드에서는 BulkheadFullException을 정상적인 운영 분기로 본다

annotation 방식은 읽기 쉽다.

```java
@Service
public class PartnerStockClient {

    private final ExternalPartnerClient externalPartnerClient;

    public PartnerStockClient(ExternalPartnerClient externalPartnerClient) {
        this.externalPartnerClient = externalPartnerClient;
    }

    @Bulkhead(name = "partnerApi", fallbackMethod = "getStockFallback")
    public StockResponse getStock(String sku) {
        return externalPartnerClient.getStock(sku);
    }

    private StockResponse getStockFallback(String sku, Throwable throwable) {
        if (throwable instanceof BulkheadFullException) {
            return StockResponse.temporarilyUnavailable(sku);
        }

        return StockResponse.fallback(sku);
    }
}
```

여기서 핵심은 `BulkheadFullException`을 "예상 못한 예외"처럼 다루지 않는 것이다.

Bulkhead가 꽉 찼다는 건 설정이 의도대로 동작해서 더 큰 장애를 막았다는 뜻이기도 하다.  
그래서 로그 레벨, 알람 기준, 사용자 응답을 별도로 잡아야 한다.

예를 들어:

- 사용자에게는 "일시적으로 조회할 수 없음"을 명확히 반환
- 운영 로그에는 bulkhead 이름과 요청 키를 남김
- 알람은 단건 예외가 아니라 거절률 기준으로 발송

이렇게 해야 Bulkhead가 장애를 숨기는 장치가 아니라, 장애 범위를 제한하는 장치로 남는다.

## 6) 조합 순서가 중요하다

![Bulkhead composition order](/assets/images/bulkhead-composition-order.svg)

Bulkhead를 단독으로 쓰는 경우보다 다른 Resilience4j 모듈과 같이 쓰는 경우가 많다.

내가 운영 관점에서 잡은 기본 순서는 이랬다.

1. `Bulkhead`로 동시에 들어갈 수 있는 호출 수를 먼저 제한
2. `TimeLimiter`로 각 호출의 최대 대기 시간을 제한
3. `Retry`는 제한된 실패에만 아주 작게 허용
4. `Circuit Breaker`로 실패율/느린 호출 비율이 높아지면 차단

정답 순서라기보다 "자원 보호를 먼저 생각하는 순서"다.

특히 Retry를 Bulkhead 바깥에서 크게 걸면, Bulkhead가 거절한 호출을 계속 재시도하면서 다시 포화를 만들 수 있다.  
그래서 retry 대상 예외에서 `BulkheadFullException`을 제외하거나, 아주 제한적으로만 다뤄야 한다.

## 7) 관측은 permit, queue, rejection을 같이 본다

![Bulkhead operations dashboard](/assets/images/bulkhead-ops-dashboard.svg)

SemaphoreBulkhead에서는 최소한 아래 지표를 본다.

- `resilience4j.bulkhead.available.concurrent.calls`
- `resilience4j.bulkhead.max.allowed.concurrent.calls`
- bulkhead 거절 횟수/비율
- 호출 latency p95/p99

ThreadPoolBulkhead에서는 여기에 queue와 thread pool 지표가 추가된다.

- `resilience4j.bulkhead.queue.depth`
- `resilience4j.bulkhead.queue.capacity`
- `resilience4j.bulkhead.thread.pool.size`
- `resilience4j.bulkhead.core.thread.pool.size`
- `resilience4j.bulkhead.max.thread.pool.size`

여기서 내가 가장 먼저 보는 건 queue depth였다.

queue가 계속 차오르는데 에러율이 낮으면 겉으로는 안정적으로 보일 수 있다.  
하지만 실제로는 장애를 queue에 숨기고 있는 상태일 수 있다.

그래서 ThreadPoolBulkhead를 쓸 때는 queue를 크게 잡지 않았다.  
queue는 순간 피크를 흡수하는 용도이지, 다운스트림 장애를 오래 버티는 버퍼가 아니라고 봤다.

## 8) 값은 어떻게 잡았나

처음부터 복잡하게 계산하지 않았다.  
대신 아래 기준으로 작게 시작했다.

### SemaphoreBulkhead

- 외부 API별 정상 p95 latency 확인
- 해당 API가 동시에 가져가도 되는 최대 요청 수 산정
- `maxWaitDuration`은 0 또는 아주 짧게 시작
- 거절률과 사용자 영향도를 보고 점진 조정

### ThreadPoolBulkhead

- 블로킹 I/O 전용 pool을 작게 분리
- `queueCapacity`는 작게 시작
- pool size보다 queue 증가가 먼저 보이면 다운스트림 지연으로 판단
- pool을 늘리기 전에 timeout과 circuit 상태를 먼저 확인

여기서 주의할 점은 thread pool을 키우는 게 항상 해결책은 아니라는 것이다.

외부 API가 느린데 pool만 키우면 더 많은 요청이 동시에 외부 API로 몰린다.  
그 결과 다운스트림을 더 압박하고, 우리 쪽 메모리/스레드 사용량도 같이 올라간다.

## 9) 롤아웃 체크리스트

Bulkhead는 설정값 하나로 끝내기보다 아래 순서로 붙이는 게 안전했다.

1. 보호할 외부 의존성 목록을 먼저 정한다
2. 의존성별 timeout, retry, circuit breaker 설정을 같이 확인한다
3. Semaphore/ThreadPool 중 어떤 격리가 필요한지 결정한다
4. rejection/fallback 응답을 사용자 관점에서 정의한다
5. permit, queue, rejection, latency 지표를 먼저 대시보드에 올린다
6. canary로 일부 트래픽에만 적용한다
7. 거절률이 정상적인 보호 신호인지, 과도한 사용자 영향인지 구분한다

특히 Bulkhead를 붙인 직후에는 에러가 "늘어난 것처럼" 보일 수 있다.

하지만 그 에러가 기존에는 긴 대기와 전체 지연으로 숨어 있었을 가능성이 있다.  
그래서 단순 에러 수보다 p95/p99 latency, thread pool 사용량, 다른 기능 영향도를 같이 봐야 한다.

## 10) 자주 헷갈렸던 포인트

### Q. Circuit Breaker가 있으면 Bulkhead는 없어도 되나?

아니다. Circuit Breaker는 실패율이나 느린 호출 비율을 보고 상태를 바꾼다.  
그 판단이 일어나기 전까지 이미 많은 호출이 자원을 점유할 수 있다.

Bulkhead는 그 전에 "이 의존성이 가져갈 수 있는 자원"을 제한한다.

### Q. queueCapacity를 크게 잡으면 더 안전한가?

대부분은 아니다.  
queue가 크면 순간 피크는 흡수하지만, 다운스트림 장애를 늦게 드러내고 tail latency를 키울 수 있다.

### Q. BulkheadFullException은 장애인가?

사용자 관점에서는 실패지만, 시스템 관점에서는 보호 동작이다.  
그래서 단건 예외 알림보다 비율 기반 알람이 더 낫다.

### Q. WebFlux에서도 ThreadPoolBulkhead가 필요한가?

항상 필요한 건 아니다.  
논블로킹 호출이면 SemaphoreBulkhead로 동시성만 제한하는 쪽이 단순하다.  
다만 어쩔 수 없이 블로킹 라이브러리를 호출한다면 별도 scheduler나 thread pool 격리를 같이 검토해야 한다.

---

정리하면 Bulkhead는 장애를 없애는 기능이 아니라,  
**장애가 가져갈 수 있는 자원을 제한하는 기능**이다.

Circuit Breaker가 장애 전파를 끊고,  
Retry + TimeLimiter가 일시 장애와 지연을 제어한다면,  
Bulkhead는 그 모든 호출이 사용할 수 있는 실행 공간을 먼저 나눈다.

운영에서는 이 차이가 꽤 컸다.

외부 API 하나가 느려졌을 때 전체 서비스가 같이 느려지는지,  
아니면 해당 기능만 제한적으로 실패하고 나머지는 살아남는지의 차이를 만들기 때문이다.

## References

- Resilience4j Bulkhead Guide: https://resilience4j.readme.io/docs/bulkhead
- Resilience4j Micrometer Guide: https://resilience4j.readme.io/docs/micrometer
- Spring Cloud CircuitBreaker Bulkhead Pattern: https://docs.spring.io/spring-cloud-circuitbreaker/reference/spring-cloud-circuitbreaker-resilience4j/bulkhead-pattern-supporting.html
- Spring Cloud CircuitBreaker Bulkhead Properties: https://docs.spring.io/spring-cloud-circuitbreaker/reference/spring-cloud-circuitbreaker-resilience4j/bulkhead-properties-configuration.html
