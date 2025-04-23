---
title: "API exception response 공통화 처리"
category: [Troubleshooting]
# tag: [jpa, db, spring boot]
toc: true
toc_sticky: true
published: true
---

## 1. 문제 상황
원클릭 API를 개발하면서 비즈니스 오류 처리에 대한 고민이 있었습니다. API 응답에서 HTTP 상태 코드와 비즈니스 로직의 resultCode 간의 불일치로 인해 다음과 같은 문제가 발생했습니다:
- **200 OK** 상태 코드로 응답하면서 가 `"error"`인 경우 `resultCode`
- **400-500대 상태 코드**로 응답하면서도 실제로는 비즈니스 오류인 경우

이러한 불일치는 클라이언트 측에서 오류를 구분하고 처리하는데 어려움을 초래했습니다.
## 2. 해결 방안: 비즈니스 오류의 일관된 처리
### 2.1 핵심 원칙
비즈니스 오류는 항상 다음 두 가지 원칙을 따르도록 설계했습니다:
1. **200 OK** 상태 코드로 응답
2. 를 통한 비즈니스 오류 명시 `resultCode`

### 2.2 예외 처리 구조 설계
비즈니스 오류를 처리하기 위한 구조를 다음과 같이 설계했습니다:
1. `ApiException`: 공통 예외 클래스
2. `OneclickException`: 원클릭 API 전용 예외 처리
3. `OneclickErrorDecoder`: 400-500대 상태 코드 처리

## 3. 구현 세부사항
### 3.1 ApiException 구현
``` java
@Getter
public class ApiException extends RuntimeException {
    private HttpStatus httpStatus = HttpStatus.BAD_REQUEST;
    private String resultCode = "500";
    private String resultMessage = "fail";

    public ApiException(String resultMessage) {
        super(resultMessage);
        this.resultMessage = resultMessage;
    }

    public ApiException(String resultCode, String resultMessage) {
        super(resultMessage);
        this.resultCode = resultCode;
        this.resultMessage = resultMessage;
    }

    public int status() {
        return this.httpStatus == null ? -1 : this.httpStatus.value();
    }
}
```
### 3.2 OneclickException 구현
``` java
public class OneclickException extends ApiException {
    private Map<String, Object> errorMap;

    public OneclickException(int statusCode, Map<String, Object> errorMap) {
        super(statusCode,
              errorMap.getOrDefault("errorCode", "UNKNOWN_ERROR_CODE").toString(),
              errorMap.getOrDefault("errorMsg", "oneclick fail").toString());
        this.errorMap = errorMap;
    }

    public Map<String, Object> getErrorMap() {
        return errorMap;
    }
}
```
### 3.3 OneclickErrorDecoder 구현
``` java
@Slf4j
public class OneclickErrorDecoder implements ErrorDecoder {
    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public Exception decode(String methodKey, Response response) {
        try {
            if (response.status() >= 400 && response.status() < 500 && response.body() != null) {
                String body = Util.toString(response.body().asReader(StandardCharsets.UTF_8));
                Map<String, Object> errorMap = objectMapper.readValue(body, new TypeReference<>() {});
                return new OneclickException(response.status(), errorMap);
            }
        } catch (IOException e) {
            return new Default().decode(methodKey, response);
        }
        return new Default().decode(methodKey, response);
    }
}
```
## 4. 실제 활용 예시
### 4.1 서비스 레이어에서의 예외 처리
``` java
public CpnUpdateResDto updateMyCpnInfo(CpnUpdateReqDto reqDto) {
    try {
        checkWebId(reqDto.getWebId());
        return client.updateMyCpnInfo(reqDto).getBody();
    } catch (OneclickException e) {
        return CpnUpdateResDto.of()
            .errorCode(e.getResultCode())
            .errorMsg(e.getResultMessage())
            .resultCode("1")
            .build();
    }
}
```
## 5. 개선 효과
이러한 구조 개선을 통해 다음과 같은 이점을 얻을 수 있었습니다:
1. **일관성**: 비즈니스 오류 처리 방식 통일
2. **명확성**: 클라이언트 측 오류 처리 로직 단순화
3. **확장성**: 다른 API에도 적용 가능한 공통 구조 확보
4. **유지보수성**: 오류 처리 로직 중앙화

## 6. 마무리
이번 개선을 통해 비즈니스 오류 처리의 일관성을 확보하고, 클라이언트-서버 간 명확한 의사소통이 가능해졌습니다. 특히 HTTP 상태 코드와 비즈니스 오류 코드의 분리를 통해 더 명확한 에러 처리가 가능해졌습니다.
앞으로도 이러한 구조를 기반으로 다른 API 개발에도 일관된 오류 처리 패턴을 적용할 수 있을 것 같습니다. 😊
