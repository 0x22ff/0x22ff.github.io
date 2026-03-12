---
layout: post
title: "WebFlux에서 BlockHound로 블로킹 잡기: 실무 적용 가이드"
date: 2026-03-12 11:30:00 +0900
categories: [backend]
tags: [spring, webflux, blockhound, reactor, async]
thumbnail: /assets/images/thumb-webflux-blockhound.svg
---

WebFlux를 쓰는데도 응답이 갑자기 느려지는 순간이 있습니다.  
대부분 원인은 비슷합니다. **논블로킹 경로 안에 블로킹 코드가 섞여 들어간 경우**입니다.

BlockHound는 이 지점을 아주 직설적으로 잡아줍니다.

- "지금 이 스레드에서 이 호출은 블로킹이다"
- "그래서 여기서 실패시킨다"

이 글은 BlockHound를 **왜 쓰는지**, **어떻게 붙이는지**, **실패가 났을 때 어떻게 고치는지**를 실전 흐름으로 정리한 가이드입니다.

## 1) 왜 BlockHound가 필요한가

![Why BlockHound](/assets/images/blockhound-why.svg)

WebFlux의 강점은 이벤트 루프 스레드를 오래 붙잡지 않는 데 있습니다.  
그런데 `Thread.sleep`, 블로킹 JDBC, 파일 I/O 같은 코드가 섞이면 그 장점이 바로 사라집니다.

겉으로는 CPU가 여유 있어 보이는데 p95/p99가 튀는 경우가 여기서 자주 나옵니다.

## 2) BlockHound가 실제로 하는 일

![BlockHound detection flow](/assets/images/blockhound-detection-flow.svg)

BlockHound는 런타임에서 블로킹 메서드 호출을 감지합니다.  
그리고 그 호출이 "논블로킹으로 간주되는 스레드"에서 일어나면 `BlockingOperationError`를 발생시킵니다.

핵심은 "성능 최적화 도구"라기보다 **위반 탐지 도구**라는 점입니다.

## 3) 적용: 테스트에서 먼저 켜는 게 가장 안전하다

### Gradle (테스트 의존성)

```gradle
dependencies {
    testImplementation("io.projectreactor.tools:blockhound")
}
```

### JUnit에서 설치

```java
import reactor.blockhound.BlockHound;

class ReactiveTestSupport {
    static {
        BlockHound.install();
    }
}
```

실무에서는 보통 테스트 공통 베이스에서 한 번만 설치합니다.

## 4) 실패를 재현해보면 감이 바로 온다

![BlockHound test story](/assets/images/blockhound-test-story.svg)

아래 코드는 의도적으로 블로킹을 넣은 예시입니다.

```java
import org.junit.jupiter.api.Test;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;
import reactor.test.StepVerifier;

class BlockHoundDemoTest extends ReactiveTestSupport {

    @Test
    void detectsBlockingCallOnNonBlockingThread() {
        Mono<String> pipeline = Mono.fromRunnable(() -> {
                    try {
                        Thread.sleep(30); // blocking
                    } catch (InterruptedException e) {
                        throw new RuntimeException(e);
                    }
                })
                .publishOn(Schedulers.parallel())
                .thenReturn("ok");

        StepVerifier.create(pipeline)
                .expectErrorMatches(t -> t.getClass().getName().contains("BlockingOperationError"))
                .verify();
    }
}
```

이 실패가 반갑다는 게 포인트입니다.  
운영에서 지연이 터지기 전에 CI에서 막아주니까요.

## 5) 실패가 나면 어떻게 고칠까

![Fix patterns](/assets/images/blockhound-fix-patterns.svg)

실무에서 보통 아래 3가지 중 하나입니다.

### A. 논블로킹 대체재로 교체 (가장 좋음)

- `RestTemplate` -> `WebClient`
- JDBC -> R2DBC (가능한 경우)

### B. 블로킹 구간을 격리

```java
Mono<User> loadUser(String id) {
    return Mono.fromCallable(() -> blockingRepository.findById(id))
            .subscribeOn(Schedulers.boundedElastic());
}
```

마이그레이션 단계에서 현실적으로 가장 많이 쓰는 방식입니다.

### C. 정말 불가피한 레거시 예외만 allow-list

```java
BlockHound.install(builder ->
    builder.allowBlockingCallsInside("com.example.LegacyBridge", "readOnce")
);
```

중요: 이건 "문제 해결"이 아니라 "예외 관리"입니다.  
이유/담당자/제거 시점을 같이 기록해두는 게 좋습니다.

## 6) 롤아웃은 이렇게 가면 안전하다

![Rollout strategy](/assets/images/blockhound-rollout.svg)

추천 순서:

1. 로컬 테스트에 먼저 적용
2. CI 테스트에서 강제
3. allow-list 최소화 + 정리 일정 관리
4. 지표(p95/p99)로 실제 효과 확인

이 순서로 가면 팀 충격을 줄이면서 품질 가드를 만들 수 있습니다.

## 7) 도입할 때 자주 묻는 질문

### Q1. BlockHound를 프로덕션에 항상 켜야 하나요?

필수는 아닙니다. 많은 팀이 테스트/스테이징에서 강하게 사용합니다.  
목표는 "위반을 조기 발견"하는 것이기 때문입니다.

### Q2. 모든 블로킹을 100% 잡아주나요?

아닙니다. 하지만 실무에서 문제를 만드는 주요 패턴을 빠르게 드러내는 데 매우 유용합니다.

### Q3. WebFlux면 무조건 BlockHound를 써야 하나요?

필수는 아니지만, 팀이 커지거나 코드가 늘어날수록 효과가 큽니다.  
특히 리뷰만으로 놓치기 쉬운 블로킹 호출을 자동으로 잡아줍니다.

---

정리하면 BlockHound는 "성능 튜닝 도구"보다 **반응형 규율을 지키는 안전장치**에 가깝습니다.

- 이벤트 루프를 막는 코드를 조기에 발견하고
- 실패를 테스트 단계로 당기고
- 운영 지연 사고 가능성을 낮춘다

WebFlux를 이미 쓰고 있다면, BlockHound는 가장 ROI가 높은 품질 가드 중 하나입니다.
