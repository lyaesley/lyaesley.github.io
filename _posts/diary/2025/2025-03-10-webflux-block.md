---
title: "Spring WebFlux에서 403 상태 처리와 API Key 갱신"
category: [Troubleshooting]
# tag: [jpa, db, spring boot]
toc: true
toc_sticky: true
published: true
---

## Spring WebFlux에서 403 상태 처리와 API Key 갱신: 비동기 환경에서 안정적인 재시도 전략 구현
### 1. 문제 상황
Spring WebFlux와 WebClient를 사용하여 외부 API 통합 작업 중 다음과 같은 문제가 발생했습니다.
- **HTTP 403 상태 코드** 반환 시, 만료된 API 키를 갱신하고 요청을 재시도해야 하는 요구사항이 있었습니다.
- `renewApiKey()` 메서드의 호출 구조상 동기적인 블로킹 로직을 포함하고 있었으며, 이는 WebFlux 비동기 환경 내에서 문제가 발생하는 원인이 됐습니다.

특정 요청에서 API가 403을 반환했을 때 아래의 예외가 발생했습니다:
``` plaintext
java.lang.IllegalStateException: 
block()/blockFirst()/blockLast() are blocking, which is not supported in thread reactor-http-nio-7
```
위 에러는 WebFlux의 실행 스레드에서 블로킹 호출(`block()`)이 이루어질 때 발생합니다. Spring WebFlux는 비동기-논블로킹 환경을 기반으로 작동하기 때문에, Reactor의 내부 스레드(Event Loop)에서 블로킹 연산을 시도하면 이러한 예외를 즉시 던지게 됩니다.
결론적으로, `renewApiKey()` 메서드가 내부적으로 동기 호출로 설계되어 실행 흐름을 방해했고, Reactor의 비동기 대응 패턴을 준수하지 못했기 때문에 문제가 발생했습니다.
### 2. 해결 접근법 개요
문제를 해결하기 위해 고려한 접근법은 다음과 같습니다:
1. **동기 작업인 `renewApiKey()`를 비동기로 전환**
    - `renewApiKey()` 내부에서 동기 메서드(`block()`)를 호출하지 않도록 리팩토링하여 비동기 반환값(`Mono<Void>`)을 지원하도록 변경.
    - WebFlux의 비동기 흐름을 유지하기 위해 Reactor의 `Mono` 반환을 적용.

2. **비동기로 전환이 불가능한 경우, 별도 Scheduler로 분리**
    - 만일 API 키 갱신 로직을 **완전히 비동기로 변경할 수 없다면**, 동기 호출을 보조 스레드에서 실행할 수 있도록 Reactor의 `Schedulers` 유틸리티를 활용.

3. **재시도 로직 추가**
    - HTTP 403 시 **단일 재시도**를 구현하기 위해 Reactor의 `retryWhen` 연산자를 사용. 이는 403 상태에서 API 키 갱신 후 한 번만 재시도를 수행하도록 설계.

### 3. `renewApiKey` 메서드를 비동기로 리팩토링
`renewApiKey()`를 비동기 방식으로 변경하기 위해 리턴 타입을 `void → Mono<Void>`로 수정했습니다.
기존 동기 로직 외부에 WebClient를 활용한 비동기 데이터 갱신 흐름을 결합한 방식입니다.
#### **리팩토링된 `renewApiKey` 메서드**:
``` java
protected Mono<Void> renewApiKey() {
    return Mono.fromCallable(() -> {
        String newKey = foApiKeyGenerator.getOrGenerateApiKey(foApiUrl); // 여기서 동기 호출
        if (StringUtils.isNotEmpty(newKey)) {
            this.foApiKey = newKey;
            initializeWebClient(); // WebClient 갱신
            log.info("API key renewed successfully.");
        } else {
            log.warn("Failed to renew API key.");
        }
        return newKey;
    }).then(); // 반환값을 Mono<Void>로 전환
}
```
#### **동작 설명**:
- `Mono.fromCallable()`은 블로킹 방식으로 동작하던 기존의 `renewApiKey()` 로직을 비동기 흐름으로 감쌉니다.
- 내부에서 동기적으로 동작하더라도 Reactor가 이를 비동기 스트림으로 실행하며, Reactor 흐름의 논블로킹 환경을 유지합니다.
- `return newKey` 결과값을 사용하지 않기 때문에 `.then()`을 호출하여 `Mono<Void>`로 변경합니다.

### 4. WebClient 요청 로직에서 `renewApiKey` 호출 흐름 수정
위에서 리팩토링한 비동기 `renewApiKey()` 메서드를 WebClient의 비동기 요청 흐름에 자연스럽게 통합합니다.
HTTP 403 상태를 감지했을 때, 갱신 로직(`renewApiKey`)이 실행되고 Mono 연산 체인에서 자연스럽게 재시도로 이어지도록 설계되었습니다.
#### **수정된 `handleRequest`**:
``` java
protected <T> T handleRequest(WebClient.RequestHeadersSpec<?> requestSpec, Class<T> responseType) {
    return requestSpec.cookie(DEFAULT_COOKIE_KEY, CookieUtil.getCookieValue(DEFAULT_COOKIE_KEY))
            .retrieve()
            .onStatus(status -> status == HttpStatus.FORBIDDEN, res -> {
                log.warn("403 Forbidden. Attempting API key renewal...");
                // 비동기로 API 키 갱신 호출
                return renewApiKey()
                        .then(Mono.error(new IllegalStateException("403 Forbidden - Retry")));
            })
            .onStatus(HttpStatus::is4xxClientError, res -> {
                log.warn("Client error: {}", res.statusCode());
                return Mono.error(new RuntimeException("4xx Client Error: " + res.statusCode()));
            })
            .bodyToMono(responseType)
            .retryWhen(reactor.util.retry.Retry.fixedDelay(1, Duration.ofMillis(200))
                .filter(throwable -> throwable instanceof IllegalStateException 
                        && throwable.getMessage().contains("403 Forbidden"))
                .doBeforeRetry(retrySignal -> log.info("Retrying request after API key renewal...")))
            .onErrorResume(ex -> {
                log.error("Error during API call: {}", ex.getMessage());
                return Mono.empty();
            })
            .block(); // 동기 처리
}
```
#### **변경 및 개선 사항**:
1. **`renewApiKey()`를 비동기 호출**
    - 기존 동기 메서드 호출(`block()` 제거).
    - 비동기 API 키 갱신 결과를 상위 로직에서 자연스럽게 처리.

2. **403 상태 재시도 처리 연계**
    - `403 Forbidden` 시 `renewApiKey()`를 실행하고, Mono가 완료된 후 재시도 트리거(`retryWhen`)를 설정.

3. **Retry 전략 개선**
    - **1회 재시도**: 무한 재시도로 인한 성능 문제 예방.
    - **200ms 지연**: 재시도 간격 설정.

### 5. `renewApiKey`를 비동기로 변경할 수 없을 경우 (대안)
만약 `renewApiKey()` 내부적으로 비동기로 전환할 수 없는 상황이라면 Reactor의 `Scheduler`를 이용해 블로킹 작업을 별도의 스레드로 분리할 수 있습니다.
#### **Scheduler를 활용한 처리**:
``` java
protected <T> T handleRequest(WebClient.RequestHeadersSpec<?> requestSpec, Class<T> responseType) {
    return requestSpec.retrieve()
            .onStatus(HttpStatus::is4xxClientError, res -> {
                log.warn("Client error: {}", res.statusCode());
                if (res.statusCode() == HttpStatus.FORBIDDEN) {
                    log.warn("403 Forbidden. Renewing API key...");
                    // 블로킹 작업을 별도 스레드에서 실행
                    return Mono.fromRunnable(() -> renewApiKey().block())
                            .subscribeOn(Schedulers.boundedElastic())
                            .then(Mono.error(new IllegalStateException("403 Forbidden - Retry")));
                }
                return Mono.error(new RuntimeException("Client error: " + res.statusCode()));
            })
            .bodyToMono(responseType)
            .retryWhen(reactor.util.retry.Retry.fixedDelay(1, Duration.ofMillis(200))
                .doBeforeRetry(retrySignal -> log.info("Retrying request after API key renewal...")))
            .block();
}
```
#### **적용 필요성**:
- 완전한 비동기 전환이 어려울 경우 일시적인 대안으로 사용.
- 하지만 장기적으로는 **renewApiKey의 비동기화**가 더 바람직합니다.

### 6. 마무리: 배운 점 및 Best Practice
#### **WebFlux의 비동기 처리 철학**
Spring WebFlux를 사용할 때, 기본적인 설계 원칙은 **완전한 논블로킹 비동기 처리**입니다.
비동기 환경에서는 동기 블로킹 메서드 호출이 Reactor의 Event Loop를 방해할 수 있습니다. 이를 방지하기 위해 다음 사항을 염두에 둬야 합니다:
1. **모든 API 호출과 작업을 비동기로 작성**
   Reactor의 Mono/Flux를 반환하도록 메서드를 작성하고, 필요에 따라 `Schedulers`를 활용해 작업을 분리합니다.
2. **`retryWhen`과 같은 Reactor 유틸리티 적극 활용**
   실패한 요청에 대한 적절한 재시도 전략을 구현합니다. 이는 코드의 가독성과 유지 보수성을 크게 향상시킵니다.
3. **블로킹 로직 최소화 및 제거**
   역사적으로 동기 작업을 사용해왔던 코드라면, 이를 비동기로 리팩토링하여 WebFlux와의 호환성을 유지하는 것이 중요합니다.

#### **결론**
이번 문제는 WebFlux에서 동기적인 블로킹 로직으로 인해 발생한 예외 상황을 다루는 과정에서 Reactor 기반의 비동기 패턴을 깊이 이해할 수 있는 좋은 기회가 됐습니다. 비동기 처리의 원칙을 준수하고 적절히 활용하는 것은 WebFlux의 성능과 안정성 모두를 보장하는 핵심적인 요소입니다.
