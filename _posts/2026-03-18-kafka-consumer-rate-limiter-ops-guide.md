---
layout: post
title: "Kafka Consumer를 초당 1건으로 제어한 방법: Resilience4j RateLimiter 적용기"
date: 2026-03-18 12:10:00 +0900
categories: [backend]
tags: [kafka, spring-kafka, resilience4j, rate-limiter, ops]
thumbnail: /assets/images/thumb-kafka-rate-limiter.svg
---

이번엔 딱 내가 실제로 썼던 케이스만 정리해본다.

요구사항은 단순했다.

- Kafka 메시지를 consumer에서 읽어서 비즈니스 처리
- 다운스트림 제약 때문에 처리 속도는 `초당 1건`으로 고정
- 과속 처리로 장애 전파가 나지 않게 제어

핵심은 producer를 막는 게 아니라, **consumer 처리 속도를 의도적으로 제한**하는 쪽이었다.

## 1) 전체 흐름부터 보면 쉽다

![Kafka consumer flow with rate limiter](/assets/images/kafka-rate-limiter-flow.svg)

흐름은 아래처럼 잡았다.

1. poll로 레코드 1건 수신
2. `RateLimiter` permit 획득 시도
3. permit이 있으면 비즈니스 처리
4. 정상 처리 후 ack(오프셋 커밋)

여기서 중요한 건, permit을 못 받았는데 ack를 해버리면 메시지를 잃을 수 있다는 점이다.

## 2) 설정은 먼저 보수적으로 시작했다

![Configuration map for 1 msg/sec consumer](/assets/images/kafka-rate-limiter-config.svg)

`application.yml` 예시는 아래처럼 시작했다.

```yaml
spring:
  kafka:
    listener:
      ack-mode: manual
      concurrency: 1
    consumer:
      max-poll-records: 1

resilience4j:
  ratelimiter:
    instances:
      kafkaWorker:
        limitForPeriod: 1
        limitRefreshPeriod: 1s
        timeoutDuration: 0
```

설정 포인트:

- `limitForPeriod: 1` + `limitRefreshPeriod: 1s` -> 초당 1 permit
- `timeoutDuration: 0` -> permit 없으면 즉시 실패 처리
- `concurrency: 1`, `max-poll-records: 1` -> burst를 먼저 차단

인스턴스가 여러 개인 경우엔 전체 처리량이 합산되니, 인스턴스 수까지 포함해서 총량 계산을 먼저 해야 한다.

## 3) Consumer 코드 예시

아래 코드는 Spring Kafka listener에서 permit을 먼저 확인한 다음 처리/ack를 분기하는 패턴이다.

```java
@Service
public class BillingConsumer {

    private final RateLimiter rateLimiter;
    private final BillingService billingService;

    public BillingConsumer(RateLimiterRegistry registry, BillingService billingService) {
        this.rateLimiter = registry.rateLimiter("kafkaWorker");
        this.billingService = billingService;
    }

    @KafkaListener(topics = "billing-events", groupId = "billing-worker")
    public void consume(ConsumerRecord<String, BillingEvent> record,
                        Acknowledgment ack) {

        boolean permitted = rateLimiter.acquirePermission();
        if (!permitted) {
            // permit 미획득: ack하지 않고 예외를 던져 재시도/백오프 경로로 보냄
            throw new IllegalStateException("rate limit exceeded");
        }

        billingService.process(record.value());
        ack.acknowledge();
    }
}
```

여기서 재시도 전략은 listener container의 error handler(`DefaultErrorHandler`, backoff, DLT 등)와 같이 설계해야 한다.

## 4) 운영에서 진짜 봐야 하는 지표

![Rate limiter metrics dashboard](/assets/images/kafka-rate-limiter-metrics.svg)

내가 최소로 본 건 3개였다.

- 실제 처리 TPS(정말 1 근처로 유지되는지)
- consumer lag(지속 증가하는지)
- permit 미획득 비율(거부가 과도한지)

TPS만 보면 놓치는 게 많다. lag가 천천히 쌓이면 나중에 한 번에 터지기 때문에, lag 추세를 같이 봐야 한다.

## 5) 롤아웃은 한 번에 하지 않았다

![Kafka rate limiter rollout checklist](/assets/images/kafka-rate-limiter-rollout.svg)

안전하게 적용하려고 순서를 고정했다.

1. 현재 TPS/lag baseline 먼저 확보
2. 일부 consumer(canary)만 적용
3. 24시간 이상 지표 관찰
4. 이상 없으면 점진 확대
5. 알람 임계치 + 장애 runbook 확정

메시지 처리 계열은 작은 설정 하나가 누적 지연으로 바로 연결되기 때문에, canary 없이 한 번에 여는 건 피하는 게 낫다.

## 6) 자주 헷갈렸던 포인트

### Q. `@KafkaListener` concurrency가 2 이상이면?

전체 처리량이 늘어난다. `초당 1건` 요구가 전체 기준인지, 인스턴스/스레드 기준인지 먼저 정의해야 한다.

### Q. permit 미획득 시 sleep으로 기다리면 안 되나?

가능은 하지만, listener 스레드를 오래 묶는 방식은 운영에서 병목 원인이 되기 쉽다. 그래서 나는 `timeoutDuration: 0`으로 즉시 실패시키고, 재시도/백오프 경로에서 제어하는 쪽을 택했다.

### Q. lag가 계속 오르면 limit을 올려야 하나?

먼저 처리 로직 지연(외부 API, DB, 락 경합)부터 확인했다. 무작정 limit만 올리면 다운스트림 장애를 다시 키울 수 있다.

---

정리하면 이 케이스의 핵심은 이것 하나였다.

**"consumer 속도를 제어해서 시스템 전체를 안정적으로 운영한다."**

RateLimiter는 단순한 성능 옵션이 아니라, 메시지 처리 안정성을 지키는 운영 장치에 가깝다.

## References

- Resilience4j RateLimiter Guide: https://resilience4j.readme.io/docs/ratelimiter
- Resilience4j Spring Boot 3 Guide: https://resilience4j.readme.io/docs/getting-started-3
- Spring for Apache Kafka Reference: https://docs.spring.io/spring-kafka/reference/
- Spring Kafka Message Listener Containers: https://docs.spring.io/spring-kafka/reference/kafka/receiving-messages/message-listener-container.html
