---
layout: post
title: "Spring WebFlux 비동기, 한 번에 이해하는 실전 가이드"
date: 2026-03-12 10:00:00 +0900
categories: [backend]
tags: [spring, webflux, reactor, async, java]
thumbnail: /assets/images/thumb-webflux-async.svg
---

트래픽이 늘어나면 서버가 버거워지는 이유는 생각보다 단순하다.  
CPU가 부족해서가 아니라, **스레드가 I/O 대기 중에 묶여버리는 경우**가 많다.

Spring WebFlux는 이 문제를 "스레드를 더 늘리는 방식"이 아니라,  
**대기 자체를 논블로킹으로 처리하는 방식**으로 푼다.

이번 글은 Reactive Streams/Reactor/WebFlux 기준으로,  
실무에서 제일 헷갈리는 포인트를 흐름 중심으로 정리했다.

## 먼저 큰 그림

![MVC vs WebFlux](/assets/images/spring-webflux-sync-vs-async.svg)

WebFlux를 한 줄로 말하면 이렇다.

- 스레드는 기다리지 않고
- I/O가 끝나는 시점에 다음 체인을 이어서 처리한다

중요한 건 "무조건 빠르다"가 아니다.  
**동시성이 큰 I/O 바운드 작업에서 유리하다**가 정확한 표현이다.

## Mono, Flux부터 잡고 가기

![Mono and Flux timeline](/assets/images/spring-webflux-mono-flux.svg)

WebFlux에서 반환 타입은 거의 둘이다.

- `Mono<T>`: 0개 또는 1개
- `Flux<T>`: 0개 이상 N개

컨트롤러 예시로 보면 바로 감이 온다.

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

핵심은 컨트롤러가 값을 즉시 만들어 반환하는 게 아니라,  
**비동기 파이프라인(Publisher)을 반환한다**는 점이다.

## WebClient 병렬 호출: 체감 차이가 나는 구간

외부 API를 여러 개 조합하는 화면에서 WebFlux 장점이 잘 드러난다.

![WebClient fan-out with Mono.zip](/assets/images/spring-webflux-webclient-fanout.svg)

예시로 대시보드 API가 아래 3개를 동시에 호출한다고 보자.

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

여기서 핵심은 `Mono.zip()`이다.  
순차가 아니라 병렬로 진행되기 때문에 전체 대기 시간을 줄일 수 있다.

## 가장 흔한 실수: 이벤트 루프에서 블로킹 호출

WebFlux에서 성능을 무너뜨리는 대표 원인:

- JDBC 같은 블로킹 드라이버 호출
- `Thread.sleep(...)`
- `block()` 남용

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

가능하면 더 좋은 방향은 분명하다.  
**끝까지 논블로킹 스택(WebClient, R2DBC 등)으로 가는 것**이다.

중간에 블로킹 구간이 늘어나면 WebFlux 장점은 빠르게 줄어든다.

## Backpressure: 처리 가능한 만큼만 받기

Reactive Streams 핵심은 생산자가 밀어 넣는 구조가 아니라,  
**소비자가 처리 가능한 만큼 요청(request n)하는 구조**라는 점이다.

![Reactive Streams backpressure](/assets/images/spring-webflux-backpressure.svg)

운영에서 이게 중요한 이유는 단순하다.

- 빠른 producer가 느린 consumer를 압도하지 않게 막고
- 메모리 급증, 지연 폭증 가능성을 낮춘다

Reactor에서는 상황에 맞춰 `limitRate`, `onBackpressureBuffer`, `onBackpressureDrop` 같은 전략을 쓴다.

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

실시간 알림, 진행 상태, 모니터링 이벤트 같은 시나리오에 잘 맞는다.

## 그러면 언제 WebFlux가 맞을까?

아래 조건이 많으면 WebFlux가 잘 맞는다.

- 외부 API/DB 호출이 많고 동시 요청이 큼
- 스트리밍 응답(SSE 등)이 필요함
- 논블로킹 기반으로 end-to-end 설계 가능함

반대로 아래라면 MVC가 더 단순하고 유지보수에 유리할 수 있다.

- 트래픽이 크지 않고 CRUD 중심
- 기존 코드가 블로킹 라이브러리에 크게 의존
- 팀의 Reactor 디버깅/운영 경험이 적음

## 도입 체크리스트

1. "왜 WebFlux인지"를 먼저 정의
2. 블로킹 라이브러리 사용 지점 전수 점검
3. `block()` 사용 위치를 정책으로 제한
4. 타임아웃/재시도/서킷브레이커 같이 설계
5. 테스트에서 지연/오류/취소 시나리오 검증

---

WebFlux는 마법 도구는 아니다.  
대신 **I/O 대기 시간을 어떻게 다룰지**를 정확히 설계하면 체감 차이를 크게 만든다.

정말 중요한 건 이 세 가지다.

- `Mono`, `Flux`를 자연스럽게 다루고
- 이벤트 루프를 막지 않고
- 블로킹 구간을 명확히 격리하는 것

이 세 가지만 지켜도 WebFlux는 충분히 강력하게 동작한다.
