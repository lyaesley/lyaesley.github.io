---
title: "Spring WebFlux에서 403 상태 처리와 API Key 갱신: WebClient Filter를 활용한 개선"
category: [Troubleshooting]
# tag: [jpa, db, spring boot]
toc: true
toc_sticky: true
published: true
---

## 1. 개선 전/후 비교

### Before (기존 코드)
```java
@Component
public class FoApiClient {
    private WebClient webClient;
    
    protected <T> T handleRequest(WebClient.RequestHeadersSpec<?> requestSpec, Class<T> responseType) {
        return requestSpec
            .retrieve()
            .onStatus(status -> status == HttpStatus.FORBIDDEN, response -> {
                // 403 발생 시 여기서 처리
                renewApiKey();  // API 키 갱신
                return response.createException(); // 예외 발생으로 재시도 유도
            })
            .bodyToMono(responseType)
            .block();
    }

    private void renewApiKey() {
        // API 키 갱신 로직
        this.apiKey = generateNewApiKey();
        // WebClient 재생성
        this.webClient = createNewWebClient();
    }
}
```


### After (개선된 코드)
```java
public abstract class AbstractFoApiClient {
    private volatile WebClient webClient;
    private boolean initialized = false;

    private synchronized void initializeWebClient() {
        this.webClient = WebClient.builder()
            .baseUrl(foApiUrl)
            .defaultHeaders(headers -> initializeDefaultHeaders().forEach(headers::add))
            .filter((request, next) -> {
                synchronized (this) {
                    if (!isInitialized()) {
                        initializeClient();
                    }
                }
                return next.exchange(request);
            })
            .build();
    }
}
```


## 2. 주요 변경 사항 비교

| 구분 | 기존 방식 | 개선된 방식 |
|------|-----------|------------|
| 403 처리 위치 | 각 요청 메서드 내부 | WebClient Filter |
| 초기화 검증 | 요청 시점 | 필터 단계에서 선제적 검증 |
| 동시성 제어 | 없음 | synchronized 블록으로 보장 |
| WebClient 관리 | 단순 참조 | volatile로 가시성 보장 |
| 에러 처리 범위 | 403 에러만 | 전체 4xx 에러 대응 |

## 3. 개선 효과

### 1) 아키텍처 측면
- **중앙화된 처리**: 모든 인증 관련 로직이 필터에서 일괄 처리
- **관심사 분리**: 비즈니스 로직과 인증 처리 분리
- **재사용성**: 추상 클래스를 통한 공통 기능 제공

### 2) 성능 측면
- **선제적 검증**: 요청 전 인증 상태 확인으로 불필요한 403 감소
- **효율적인 초기화**: 필요한 시점에 한 번만 초기화
- **스레드 안전성**: 동시성 제어로 안정적인 동작

### 3) 운영 측면
- **모니터링 용이성**: 로그를 통한 상태 추적 가능
- **유지보수성**: 인증 관련 로직 수정 시 필터만 수정
- **확장성**: 새로운 인증 요구사항 추가가 용이

## 4. 실제 동작 흐름 비교

### 기존 방식
1. 요청 발생
2. API 호출
3. 403 발생
4. API 키 갱신
5. WebClient 재생성
6. 재시도

### 개선된 방식
1. 요청 발생
2. Filter에서 초기화 상태 확인
3. 필요시 선제적 초기화
4. API 호출
5. 응답 처리

## 5. 결론
WebClient Filter를 활용한 개선은 다음과 같은 이점을 제공합니다:
1. 선제적인 인증 상태 확인으로 불필요한 403 에러 감소
2. 스레드 안전한 구현으로 동시성 이슈 해결
3. 중앙화된 처리로 코드 유지보수성 향상
4. 명확한 책임 분리로 코드 품질 개선

이러한 개선은 특히 높은 트래픽 환경에서 시스템의 안정성과 성능을 향상시키는데 큰 도움이 됩니다.