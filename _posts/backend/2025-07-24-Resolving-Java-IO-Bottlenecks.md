---
title: "# Java IO 병목 해결하기: 블로킹 IO에서 가상 스레드까지"
category: [server]
tag: [server, backend, IO, blocking]
toc: true
toc_sticky: true
published: true
---

## 📋 개요
백엔드 서비스에서 가장 흔하게 마주치는 성능 문제 중 하나가 바로 **IO 병목**입니다. 최범균 저자의 "주니어 백엔드 개발자가 반드시 알아야 할 실무 지식" 7장을 바탕으로, IO 병목의 원인과 해결 방법에 대해 정리해보겠습니다.
## 🔍 IO 병목이 발생하는 이유
### 전통적인 블로킹 IO의 문제점
**트래픽이 증가하면서 발생하는 문제들:**[[6]](https://ms727.tistory.com/62)
- **IO 대기**: 네트워크 응답을 기다리는 동안 스레드가 블로킹됨
- **컨텍스트 스위칭**: 스레드 간 전환으로 인한 CPU 낭비
- **메모리 사용량 증가**: 요청마다 스레드를 할당하여 메모리 부족
``` java
// 문제가 되는 블로킹 IO 예시
@RestController
public class UserController {
    
    @GetMapping("/user/{id}")
    public User getUser(@PathVariable String id) {
        // DB 조회 - 블로킹 IO
        User user = userRepository.findById(id);
        
        // 외부 API 호출 - 블로킹 IO  
        Profile profile = externalApiClient.getProfile(id);
        
        // 총 응답 시간 = DB 조회 시간 + API 호출 시간
        return user.withProfile(profile);
    }
}
```
## 💡 해결 방법 1: 가상 스레드 (Virtual Thread)
### 가상 스레드란?
Java 21에서 도입된 **가상 스레드**는 기존 플랫폼 스레드의 한계를 극복하는 혁신적인 해결책입니다.[[7]](https://velog.io/@shlee327/Java-21-Virtual-Thread-%EC%84%B1%EB%8A%A5-%EC%8B%A4%ED%97%98-%EB%B0%8F-%EB%B6%84%EC%84%9D)
**주요 특징:**
- **경량성**: 수백만 개를 생성해도 메모리 사용량이 크게 증가하지 않음
- **자동 일시 중단**: I/O 작업 시 자바 런타임이 논블로킹 OS 호출을 수행하고 가상 스레드를 자동으로 일시 중단[[6]](https://mangkyu.tistory.com/309)
- **높은 동시성**: 컨텍스트 스위칭과 메모리 사용량을 최소화[[9]](https://0soo.tistory.com/259)

### 가상 스레드 사용 예시
``` java
// 기존 플랫폼 스레드
public class TraditionalThreadExample {
    public void processRequests() {
        ExecutorService executor = Executors.newFixedThreadPool(100);
        
        for (int i = 0; i < 10000; i++) {
            executor.submit(() -> {
                // IO 작업 시 플랫폼 스레드가 블로킹됨
                performNetworkCall();
            });
        }
    }
}

// 가상 스레드 활용
public class VirtualThreadExample {
    public void processRequests() {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            for (int i = 0; i < 10000; i++) {
                executor.submit(() -> {
                    // IO 작업 시 가상 스레드가 자동으로 일시 중단
                    performNetworkCall();
                });
            }
        }
    }
}
```
### Spring Boot에서 가상 스레드 활용
``` java
// application.yml
spring:
  threads:
    virtual:
      enabled: true

// 또는 Java 코드로 설정
@Configuration
public class VirtualThreadConfig {
    
    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }
}
```
## 🚀 해결 방법 2: 논블로킹 IO
### 논블로킹 IO의 핵심 개념
**"기다리지 말고, 알림을 받자"**
논블로킹 IO는 IO 작업을 기다리지 않고 다른 작업을 계속 처리하는 방식입니다.
``` java
// 블로킹 방식
@Service
public class BlockingUserService {
    
    public UserInfo getUserInfo(String userId) {
        // 각 호출이 순차적으로 실행됨 (총 3초 소요)
        User user = userRepository.findById(userId); // 1초
        Profile profile = profileService.getProfile(userId); // 1초  
        Settings settings = settingsService.getSettings(userId); // 1초
        
        return new UserInfo(user, profile, settings);
    }
}

// 논블로킹 방식 (CompletableFuture 활용)
@Service
public class NonBlockingUserService {
    
    public CompletableFuture<UserInfo> getUserInfo(String userId) {
        // 모든 호출을 병렬로 실행 (최대 1초 소요)
        CompletableFuture<User> userFuture = 
            CompletableFuture.supplyAsync(() -> userRepository.findById(userId));
            
        CompletableFuture<Profile> profileFuture = 
            CompletableFuture.supplyAsync(() -> profileService.getProfile(userId));
            
        CompletableFuture<Settings> settingsFuture = 
            CompletableFuture.supplyAsync(() -> settingsService.getSettings(userId));
        
        return CompletableFuture.allOf(userFuture, profileFuture, settingsFuture)
            .thenApply(v -> new UserInfo(
                userFuture.join(),
                profileFuture.join(), 
                settingsFuture.join()
            ));
    }
}
```
### WebFlux를 활용한 리액티브 프로그래밍
``` java
@RestController
public class ReactiveUserController {
    
    @GetMapping("/user/{id}")
    public Mono<UserInfo> getUser(@PathVariable String id) {
        return Mono.zip(
            userRepository.findById(id),
            profileService.getProfile(id),
            settingsService.getSettings(id)
        ).map(tuple -> new UserInfo(
            tuple.getT1(), // User
            tuple.getT2(), // Profile  
            tuple.getT3()  // Settings
        ));
    }
}
```
## 📊 성능 비교

| 방식 | 동시 처리 수 | 메모리 사용량 | 복잡도 | 디버깅 난이도 |
| --- | --- | --- | --- | --- |
| 전통적 스레드 | 제한적 | 높음 | 낮음 | 쉬움 |
| 가상 스레드 | 매우 높음 | 낮음 | 낮음 | 보통 |
| 논블로킹 IO | 높음 | 보통 | 높음 | 어려움 |
## 🎯 실무 적용 가이드
### 언제 가상 스레드를 사용할까?
**적합한 경우:**
- IO 집약적인 작업이 많은 애플리케이션
- 기존 블로킹 코드를 큰 변경 없이 성능 개선하고 싶을 때
- 동시 연결 수가 많은 웹 서비스
``` java
// 적용 예시: 배치 처리
@Service
public class BatchProcessingService {
    
    public void processLargeDataset(List<String> dataIds) {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            
            List<CompletableFuture<Void>> futures = dataIds.stream()
                .map(id -> CompletableFuture.runAsync(() -> {
                    // IO 집약적인 작업
                    processData(id);
                }, executor))
                .toList();
                
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                .join();
        }
    }
}
```
### 언제 논블로킹 IO를 사용할까?
**적합한 경우:**
- 매우 높은 처리량이 필요한 시스템
- 스트리밍 데이터 처리
- 마이크로서비스 간 비동기 통신

## ⚠️ 주의사항
### 가상 스레드 사용 시 주의점
``` java
// ❌ 피해야 할 패턴
public void badExample() {
    // CPU 집약적 작업에는 적합하지 않음
    Thread.ofVirtual().start(() -> {
        performCpuIntensiveTask(); // CPU 바운드 작업
    });
    
    // synchronized 블록은 가상 스레드를 고정시킬 수 있음
    synchronized(this) {
        performIOTask();
    }
}

// ✅ 권장 패턴  
public void goodExample() {
    // IO 집약적 작업에 적합
    Thread.ofVirtual().start(() -> {
        performNetworkCall(); // IO 바운드 작업
    });
    
    // ReentrantLock 사용 권장
    ReentrantLock lock = new ReentrantLock();
    lock.lock();
    try {
        performIOTask();
    } finally {
        lock.unlock();
    }
}
```
## 🏁 결론
IO 병목 해결의 핵심은 **상황에 맞는 적절한 도구 선택**입니다:
1. **가상 스레드**: 기존 코드 변경을 최소화하면서 성능 개선
2. **논블로킹 IO**: 최고 성능이 필요하지만 복잡도 증가 감수
3. **단계적 적용**: IO 집약적인 부분부터 점진적으로 도입

현재 Java 21의 가상 스레드가 가장 실용적인 해결책으로 평가받고 있으며, 복잡한 비동기 프로그래밍 없이도 높은 성능을 달성할 수 있어 많은 개발팀에서 주목받고 있습니다.
**참고 자료:**
- [[6]](https://ms727.tistory.com/62) 주니어 백엔드 개발자가 반드시 알아야 할 실무 지식 서평
- [[7]](https://velog.io/@shlee327/Java-21-Virtual-Thread-%EC%84%B1%EB%8A%A5-%EC%8B%A4%ED%97%98-%EB%B0%8F-%EB%B6%84%EC%84%9D) Java 21 Virtual Thread 성능 실험 및 분석
- [[9]](https://0soo.tistory.com/259) Java 가상 스레드(Virtual Thread)의 이해
