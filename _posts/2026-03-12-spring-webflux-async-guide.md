---
layout: post
title: "Spring WebFlux 비동기, 한 번에 이해하는 실전 가이드"
date: 2026-03-12 10:00:00 +0900
categories: [backend]
tags: [spring, webflux, reactor, async, java]
---

요청이 많아질수록 서버가 버거워지는 이유는 보통 비슷합니다.  
CPU가 모자라서라기보다, **스레드가 I/O 대기 시간 동안 묶여버리기 때문**이죠.

Spring WebFlux는 이 문제를 "스레드를 더 늘리는 방식"이 아니라, **대기 시간을 논블로킹으로 처리하는 방식**으로 접근합니다.

이 글은 공식 개념(Reactive Streams, Reactor, Spring WebFlux)을 기준으로, 실무에서 가장 헷갈리는 지점을 쉬운 흐름으로 정리한 내용입니다.

## 먼저 큰 그림

![MVC vs WebFlux](/assets/images/spring-webflux-sync-vs-async.svg)

WebFlux를 한 줄로 정리하면 이렇습니다.

- 스레드는 기다리지 않고
- I/O 완료 시점에 콜백 체인으로 이어서 처리한다

중요한 건 "무조건 빠르다"가 아닙니다.  
**동시성이 높은 I/O 바운드 작업에서 유리하다**가 정확한 표현입니다.

## Mono, Flux부터 이해하면 절반 끝

![Mono and Flux timeline](/assets/images/spring-webflux-mono-flux.svg)

WebFlux에서 리턴 타입은 거의 이 둘입니다.

- `Mono<T>`: 0개 또는 1개
- `Flux<T>`: 0개 이상 N개

컨트롤러 기준으로 보면 감이 쉽습니다.

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public Mono<UserResponse> getUser(@PathVariable String id) {
        return userService.findUser(id);
    }

    @GetMapping
    public Flux<UserResponse> getUsers() {
        return userService.findAllUsers();
    }
}
```

여기서 중요한 포인트는 컨트롤러가 값을 즉시 계산해서 반환하는 게 아니라,  
**비동기 파이프라인(Publisher)을 반환**한다는 점입니다.

## WebClient 병렬 호출: WebFlux가 빛나는 지점

마이크로서비스나 외부 API를 여러 개 조합하는 화면에서 WebFlux 장점이 확실히 드러납니다.

![WebClient fan-out with Mono.zip](/assets/images/spring-webflux-webclient-fanout.svg)

예를 들어 대시보드 API가 아래 3개를 동시에 호출한다고 가정해보겠습니다.

- 프로필 API
- 주문 API
- 포인트 API

```java
@Service
public class DashboardService {

    private final WebClient webClient;

    public DashboardService(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://internal-api").build();
    }

    public Mono<DashboardResponse> getDashboard(String userId) {
        Mono<ProfileDto> profileMono = webClient.get()
                .uri("/profiles/{id}", userId)
                .retrieve()
                .bodyToMono(ProfileDto.class);

        Mono<List<OrderDto>> ordersMono = webClient.get()
                .uri("/orders/{id}", userId)
                .retrieve()
                .bodyToFlux(OrderDto.class)
                .collectList();

        Mono<PointDto> pointMono = webClient.get()
                .uri("/points/{id}", userId)
                .retrieve()
                .bodyToMono(PointDto.class);

        return Mono.zip(profileMono, ordersMono, pointMono)
                .map(tuple -> new DashboardResponse(tuple.getT1(), tuple.getT2(), tuple.getT3()));
    }
}
```

핵심은 `Mono.zip()`입니다.  
순차 호출이 아니라 병렬로 진행해서 전체 대기 시간을 줄일 수 있습니다.

## 가장 많이 하는 실수: 이벤트 루프에서 블로킹 호출

WebFlux에서 성능이 무너지는 대표 원인은 이겁니다.

- JDBC 같은 블로킹 드라이버 호출
- `Thread.sleep(...)`
- `block()` 무분별 사용

![Event loop and blocking pitfall](/assets/images/spring-webflux-event-loop.svg)

### 피해야 하는 코드

```java
public Mono<UserResponse> badExample(String id) {
    UserEntity entity = blockingRepository.findById(id); // blocking call
    return Mono.just(UserResponse.from(entity));
}
```

### 최소한의 방어 코드

```java
public Mono<UserResponse> saferExample(String id) {
    return Mono.fromCallable(() -> blockingRepository.findById(id))
            .subscribeOn(Schedulers.boundedElastic())
            .map(UserResponse::from);
}
```

더 좋은 방향은, 가능하면 **끝까지 논블로킹 스택(WebClient, R2DBC 등)**으로 가는 겁니다.  
중간에 블로킹 구간이 많아지면 WebFlux 이점이 빠르게 줄어듭니다.

## Backpressure: 소비자가 속도를 정한다

Reactive Streams의 핵심은 생산자가 밀어 넣는 게 아니라,  
**소비자가 처리 가능한 만큼 요청(request n)한다**는 점입니다.

![Reactive Streams backpressure](/assets/images/spring-webflux-backpressure.svg)

운영에서 의미 있는 이유는 단순합니다.

- 빠른 producer가 느린 consumer를 압도하지 않게 막는다
- 메모리 폭증이나 지연 급증 가능성을 낮춘다

Reactor에서는 상황에 따라 `limitRate`, `onBackpressureBuffer`, `onBackpressureDrop` 같은 전략을 선택합니다.

## 실무에서 자주 쓰는 스트리밍 예시: SSE

```java
@GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<String>> events() {
    return Flux.interval(Duration.ofSeconds(1))
            .map(seq -> ServerSentEvent.<String>builder()
                    .event("tick")
                    .id(Long.toString(seq))
                    .data("server-time: " + Instant.now())
                    .build());
}
```

이 패턴은 실시간 알림, 진행 상태, 모니터링 이벤트에 잘 맞습니다.

## 그러면 언제 WebFlux를 쓰는 게 맞을까?

아래 조건이 많을수록 적합합니다.

- 외부 API/DB 호출이 많고 동시 요청이 큼
- 스트리밍 응답(SSE 등)이 필요함
- 논블로킹 기반으로 엔드투엔드 설계 가능함

반대로 아래라면 MVC가 더 단순하고 좋은 선택일 수 있습니다.

- 트래픽이 크지 않고 CRUD 중심
- 기존 코드가 블로킹 라이브러리 의존이 큼
- 팀이 Reactor 디버깅/운영 경험이 아직 적음

## 도입 체크리스트 (짧고 중요)

1. "왜 WebFlux인지"를 먼저 정의한다 (동시성? 스트리밍?)
2. 블로킹 라이브러리 사용 지점을 전수 점검한다
3. `block()` 사용 위치를 정책으로 제한한다
4. 타임아웃/재시도/서킷브레이커를 함께 설계한다
5. 테스트에서 지연/오류/취소 시나리오를 반드시 검증한다

---

WebFlux는 마법이 아닙니다.  
대신, **I/O 대기 시간을 어떻게 다룰지**를 정확히 설계하면 큰 차이를 만들 수 있는 도구입니다.

핵심은 단순합니다.

- 비동기 타입(`Mono`, `Flux`)을 자연스럽게 다루고
- 이벤트 루프를 막지 않고
- 블로킹 구간을 명확히 격리하는 것

이 세 가지만 지켜도, WebFlux는 충분히 강력하게 동작합니다.
