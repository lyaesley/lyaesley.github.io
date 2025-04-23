---
title: "OpenFeign 적용기"
category: [Troubleshooting]
# tag: [jpa, db, spring boot]
toc: true
toc_sticky: true
published: false
---

## 개요

Ashley 앱 개발에서 외부 솔루션과의 원활한 연동을 위해 중간 프록시 인터페이스 서버가 필요했고, 이를 지원하기 위해 새롭게 구축한 API 서버가 클라이언트 요청을 효율적으로 처리할 수 있도록 설계했습니다. 이번 과정에서 OpenFeign을 도입하여 클라이언트 요청 로직을 한층 더 간결하고 유연하게 구현할 수 있었습니다. OpenFeign의 선언적 인터페이스 덕분에 외부 솔루션과의 통신이 최적화되어 서비스 안정성과 확장성 또한 강화되었습니다.

>📌 **OpenFeign이란?**
>
>OpenFeign은 Java에서 RESTful API 호출을 선언적 인터페이스로 간편하게 구현할 수 있게 해주는 HTTP 클라이언트 라이브러리입니다. `@FeignClient` 어노테이션을 통해 API 경로와 메서드를 정의하며, Spring Boot와 잘 통합됩니다. 로드 밸런싱과 인터셉터 기능을 제공하여 부하 분산과 공통 처리가 가능하고, 테스트 시 외부 API 의존성을 줄일 수 있어 효율적입니다. 직관적이고 유지보수가 쉽습니다.
>

---

## 원클릭과의 통로를 만들자

기존에 `WebClient`를 사용해 OneClick API와 통신하는 서비스 메서드는 코드가 복잡하고 유지보수가 어려운 형태로 작성되어 있었습니다.  이 메서드는 헤더와 파라미터를 직접 추가하고, 응답을 Map으로 받아 처리하는 방식이었는데, 각종 상수와 파라미터를 직접 작성하다 보니 코드가 복잡해지고 반복적인 설정이 많아졌습니다.

```java
getOneclickClient(OneclickDto.dto queryDto) {
        String LOYPGMNO;
        String TGNO;
        ...
        Map result;
        try {
            result = (Map)((WebClient.RequestBodySpec)((WebClient.RequestBodySpec)this.webClient.post().uri(this.ONECLICK_API_DOMAIN + "apiURL", new Object[0])).headers((headers) -> {
                headers.add("Authorization", this.oneclickConfigService.getAuthorization());
                headers.add("Content-Type", "application/x-www-form-urlencoded");
                headers.add("Accept-Encoding", "gzip,deflate");
            })).body(BodyInserters.fromFormData("webId", queryDto.getWebId()).with("startDate", queryDto.getStartDate()).with("endDate", queryDto.getEndDate()).with("page", String.valueOf(queryDto.getPage())).with("pageSize", String.valueOf(queryDto.getSize())).with("trackingNo", UUID.randomUUID().toString()).with("loyPgmNo", LOYPGMNO).with("tgNo", TGNO).with("qryCoCd", QRYCOCD).with("branchCode", BRANCHCODE)).exchangeToMono((clientResponse) -> {
                return clientResponse.bodyToMono(Map.class);
            }).block();
        } catch (Exception var8) {
             error 처리
        }
        if (!"0".equals(result.get("resultCode"))) {
           원클릭 비정상결과 처리
        } else {
	        원클릭 정상 응답 처리
        }
    }
```

원클릭(OneClick) API와의 통신을 위해 OpenFeign을 사용해 API 클라이언트를 개발하면서, 효율적인 설정 관리를 위해 우선적으로 `FeignConfig` 클래스를 구현했습니다. 개발 초기에는 매번 요청에 필요한 헤더와 바디 설정을 개별적으로 적용하던 방식에서 불필요한 반복을 줄이고, 모든 요청에 일관된 처리를 제공할 수 있도록 하는 방향으로 설정 클래스를 만들어가게 되었습니다.

```java
@Slf4j
@Configuration
public class FeignConfig {
    @Value("${authorization}")
    public String ACCESS_TOKEN;

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Bean
    public RequestInterceptor requestInterceptor() {
        return requestTemplate -> {
            setCommonHeaders(requestTemplate);
            setCommonBody(requestTemplate);
        };

    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return new OneclickErrorDecoder();
    }

    private void setCommonBody(RequestTemplate requestTemplate) {
        //공통 body 에 필요한 값 세팅
        String contentType = getContentType(requestTemplate);
        if (Objects.nonNull(contentType)) {
            if(StringUtils.contains(contentType, MediaType.APPLICATION_JSON_VALUE)) {
                appendBodyJson(requestTemplate);
            } else if (StringUtils.contains(contentType, MediaType.APPLICATION_FORM_URLENCODED_VALUE)) {
                appendBodyForm(requestTemplate);
            }
        }
            

    }

    private void setCommonHeaders(RequestTemplate requestTemplate) {
        log.info("ACCESS_TOKEN :{}", ACCESS_TOKEN);
        requestTemplate.header(HttpHeaders.AUTHORIZATION, ACCESS_TOKEN);

        // MediaType.APPLICATION_FORM_URLENCODED_VALUE 타입이 아닌 경우 전부 MediaType.APPLICATION_JSON_VALUE 으로 변경
        String contentType = getContentType(requestTemplate);

        if (Objects.isNull(contentType) || !StringUtils.contains(contentType, MediaType.APPLICATION_FORM_URLENCODED_VALUE)) {
            requestTemplate.header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE);
        }
    }

    private void appendBodyJson(RequestTemplate requestTemplate) {

        Map<String, Object> existingBody = new HashMap<>();
        try {
            //기존
            if (Objects.nonNull(requestTemplate.body())) {
                existingBody = objectMapper.readValue(requestTemplate.body(), new TypeReference<>() {});
            }
            //추가
            existingBody.put("trkNo", UUID.randomUUID().toString());
            //body set
            requestTemplate.body(objectMapper.writeValueAsString(existingBody));

        } catch (IOException e) {
            throw new RuntimeException("appendBodyJson error", e);
        }
    }

    private void appendBodyForm(RequestTemplate requestTemplate) {
        try {
            FormUrlEncodedBodyBuilder bodyBuilder = new FormUrlEncodedBodyBuilder()
                .add("trkNo", UUID.randomUUID().toString())
                .addBodyQueries(new String(requestTemplate.body()))
                .addAll(requestTemplate.queries());

            requestTemplate.body(bodyBuilder.build());

        } catch (Exception e) {
            throw new RuntimeException("appendBodyForm error", e);
        }
    }

    private String getContentType(RequestTemplate requestTemplate) {
        Collection<String> collectionContentType = requestTemplate.headers().get("Content-Type");
        String contentType = StringUtils.EMPTY;
        if(Objects.nonNull(collectionContentType)) {
            contentType = collectionContentType.stream().findFirst().orElse(null);
        }
        return contentType;
    }

}

```