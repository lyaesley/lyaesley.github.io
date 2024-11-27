---
title: "동적 서비스 관리를 위한 디자인 패턴 적용"
category: [Troubleshooting]
# tag: [jpa, db, spring boot]
toc: true
toc_sticky: true
# published: true
---

## 개요

이노커머스 플랫폼은 커스텀 쇼핑몰을 구축하고 운영할 수 있도록 설계된 시스템인데요, 처음으로 애슐리의 홈스토랑몰과 프랑제리몰의 입점을 개발하면서 새로운 도전과제를 만나게 되었습니다. 그중 첫 번째 난관이 바로 `최근 본 상품` CRUD 기능이었어요. 각 쇼핑몰마다 고객이 본 상품 목록을 조회하는 API는 동일하게 제공해야 했지만, 동시에 각 몰의 비즈니스 정책에 맞춰 개별 처리가 필요했습니다. 어떻게 하면 이걸 효율적으로 관리할 수 있을까 고민하다가 **동적으로 쇼핑몰 정보에 따라 각기 다른 서비스를 찾아 제공하는 구조**를 만들기로 결정했습니다.

이렇게 해서 생각해 온 것이 `DynamicCommonServiceFactory`입니다. 이 팩토리 패턴을 사용해 쇼핑몰 정보에 따라 각 몰의 정책에 맞춘 서비스를 자동으로 찾아주도록 설계했죠. 이를 통해 이노커머스 플랫폼은 멀티 테넌트 플랫폼으로서의 확장 가능성을 열어두게 되었고,  앞으로도 다양한 몰 입점과 변화하는 요구사항에 유연하게 대응하여 각 쇼핑몰의 고유한 비즈니스를 충족하면서도 하나의 통합된 구조 안에서 효율적으로 관리할 수 있게 되었습니다.

---

## 개발 여정

 Spring의 의존성 주입(Dependency Injection)과 커스텀 어노테이션을 이용한 동적 서비스를 개발한 여정을 이야기 해보려고 합니다.

## `MallManagedService` 어노테이션과 비즈니스 서비스 분리

몰별로 맞춤형 서비스를 제공하기 위해 먼저 `MallManagedService` 어노테이션을 정의했습니다. `isTarget`이라는 기본 파라미터를 포함시켜, 서비스가 특정 몰에만 적용될 수 있도록 향후 확장이 가능하게 설계하였습니다. 아래는 어노테이션 정의입니다.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MallManagedService {
    boolean isTarget() default true;
}
```

이 어노테이션을 `ItemShopService` 인터페이스에 적용하면, 각 몰에 필요한 구체적인 비즈니스 로직을 동적으로 주입할 수 있는 구조를 만듭니다. 이 방식으로 여러 몰에서 동일한 API를 호출하더라도 각 몰에 맞춘 `최근 본 상품` 조회 정책을 구현할 수 있습니다.

```java
@MallManagedService
public interface ItemShopService {
    DisplayItemResponse getRecentSeenItemsForDisplay(DisplayItemImageQueryDto queryDto, FoUserInfo userInfo);
    void deleteOneRecentSeenItem(String itemNo, FoUserInfo userInfo);
    void deleteRecentSeenItems(List<String> itemNoList, FoUserInfo userInfo);
    Boolean resetRecentSeenItems(FoUserInfo userInfo);
}
```

---

## `DynamicCommonServiceFactory`로 몰별 비즈니스 로직 자동 적용하기

몰별로 서비스 인스턴스를 설정한 후에, 직접 하드코딩하거나 팩토리 클래스를 여러 개 만드는 대신, **한 곳에서 모든 몰의 서비스를 동적으로 호출 할 수 있는 구현체**가 필요 했습니다. `DynamicCommonServiceFactory`는 `MallManagedService`라는 커스텀 어노테이션이 붙은 클래스를 필터링해 서비스 맵을 초기화하고, 특정 몰 번호와 서비스 이름을 기반으로 필요한 서비스를 제공합니다.

### 서비스 맵 초기화하기

먼저, `@PostConstruct` 어노테이션을 이용해 `initializeServiceMap` 메서드를 정의했습니다. 이 메서드는 Spring 컨텍스트가 초기화된 직후 실행되며, **`MallManagedService` 어노테이션이 달린 빈(bean)들을 찾아 서비스 맵에 추가**합니다. 이렇게 하면 모든 몰별 서비스가 한 번에 준비되고, 이후 요청이 들어오면 빠르게 서비스를 찾아 사용할 수 있습니다.

```java
@PostConstruct
public void initializeServiceMap() {
    // MallManagedService 어노테이션이 달린 빈을 찾아 서비스 맵에 추가
    serviceMap.putAll(applicationContext.getBeansWithAnnotation(MallManagedService.class));
}
```

이 과정 덕분에 몰마다 다른 로직이 필요한 서비스를 통합적으로 관리할 수 있고, 추가적으로 필요한 서비스가 생기더라도 `MallManagedService` 어노테이션을 붙이기만 하면 자동으로 팩토리에서 처리하게 됩니다.

### 특정 몰의 서비스 가져오기

`DynamicCommonServiceFactory`의 핵심 기능 중 하나는 특정 몰의 고유한 서비스를 찾아 반환하는 `getService` 메서드입니다. 여기서는 `dispMallNo`와 `serviceClazz`를 받아 **몰 번호와 서비스 클래스 이름을 결합한 서비스 이름**을 만들어 `serviceMap`에서 해당 이름으로 검색합니다.

```java
public <T> T getService(String dispMallNo, Class<T> serviceClazz) {
    String serviceName = getServiceName(dispMallNo, serviceClazz.getSimpleName());
    T service;

    try {
        service = (T) serviceMap.get(serviceName);
        if (service == null) {
            throw new BaseException("해당 서비스를 찾을 수 없습니다.[" + serviceName + "]");
        }
    } catch (ClassCastException e) {
        throw new BaseException("요구하는 서비스의 타입과 다릅니다.:[" + serviceClazz.getSimpleName() + "," + serviceName + "]");
    }

    return service;
}
```

이 메서드를 통해 요청이 들어올 때마다 몰 번호에 맞는 서비스를 자동으로 찾아주며, 만약 서비스가 없거나 타입이 맞지 않을 경우 예외를 발생시켜 오류를 바로 파악할 수 있게 설계했습니다. 이 덕분에 서비스 로직을 분리해도 오류 관리가 간편해집니다.

### 서비스 이름 생성

마지막으로, 특정 몰에 맞는 고유 서비스 이름을 생성하는 `getServiceName` 메서드입니다. 여기서는 `dispMallNo`와 `serviceName`을 받아 `"M" + 몰 번호 + "_" + 서비스 이름` 형식으로 고유한 이름을 생성합니다. 이 방식 덕분에 몰 번호와 서비스 이름이 결합된 상태로 각기 다른 서비스를 찾을 수 있어, 빠르고 정확한 서비스 매핑이 가능합니다.

```java
public String getServiceName(String dispMallNo, String serviceName) {
    if (dispMallNo == null || dispMallNo.isEmpty() || serviceName == null || serviceName.isEmpty()) {
        throw new IllegalArgumentException("dispmallNo 또는 serviceName이 null 또는 빈 값입니다.");
    }
    return "M" + dispMallNo + "_" + serviceName;
}
```

### 서비스 호출하기

```java
@GetMapping("/v1/item/recentseen/api")
@Operation(summary = "최근본 상품 목록 조회", description = "최근본 상품 목록을 조회 한다")
public @ResponseBody InnogResponse<DisplayItemResponse> recentSeenItem(DisplayItemImageQueryDto queryDto, FoUserInfo userInfo) {

    ItemShopService service = dynamicCommonServiceFactory.getService(userInfo.getDispMallNo(), ItemShopService.class);
    DisplayItemResponse recentSeenItemsForDisplay = service.getRecentSeenItemsForDisplay(queryDto, userInfo);

    return InnogResponse.<DisplayItemResponse>from()
                        .data(recentSeenItemsForDisplay)
                        .build();
}
```

기존이라면, 컨트롤러나 서비스 클래스를 몰별로 나누고, 로직을 따로 관리해야 했겠지만 이제는 DynamicCommonServiceFactory 사용 하여 해당 몰에 적합한 서비스를 호출 할 수 있게 해줍니다.

## 정리

이 `DynamicCommonServiceFactory` 구조를 통해, 쇼핑몰마다 다른 비즈니스 로직을 유연하게 적용할 수 있게 되었고, 새로운 몰이 추가될 때마다 중복된 코드를 작성할 필요 없이 간단히 어노테이션과 인터페이스 구현으로 확장할 수 있게 되었습니다.

이 구조를 통해서 앞으로 기능이 추가되거나 필요에 따라 확장해볼 수 있는 방향이 무궁무진합니다. 지금 당장 다 적용하지 않더라도, 하나씩 단계적으로 도입해볼 만한 아이디어들이 많습니다.

### 1. **서비스 조건의 세분화**

- 현재 `MallManagedService` 어노테이션에 `isTarget`이라는 기본 플래그를 사용하고 있지만, **몰별로 더 구체적인 조건**을 적용할 수도 있습니다. 예를 들어 `mallType`, `serviceLevel`, `region` 등 다양한 파라미터를 추가하여 조건에 맞는 몰별 정책을 정의할 수 있습니다.
- 이를 통해 더욱 유연하고 세밀하게 몰별 서비스를 구성할 수 있고, 특정 몰에만 특화된 비즈니스 로직을 적용할 수 있습니다.

### 2. **다양한 인터페이스 확장 및 공통 로직 추가**

- 현재 `ItemShopService` 하나의 인터페이스에 `최근 본 상품` 관련 기능을 제공하고 있지만, **기타 비즈니스 기능들을 지원하는 인터페이스를 추가**할 수도 있습니다. 예를 들어 `FavoriteService` (즐겨찾기 관련), `RecommendationService` (추천 관련) 등 다양한 몰별 기능을 갖춘 서비스 인터페이스를 추가하여 `DynamicCommonServiceFactory`를 확장할 수 있습니다.
- 공통된 인터페이스들에 대해 동적 팩토리가 필요에 따라 다르게 서비스 제공이 가능해집니다.

### 3. **어노테이션 기반의 정책 플러그인 시스템 구축**

- 어노테이션을 기반으로 플러그인 방식의 몰별 정책을 추가할 수도 있습니다. 예를 들어 `MallManagedService` 어노테이션과 별개로, **몰별 정책을 지정하는 어노테이션을 추가**하고, 이를 읽어 처리할 수 있는 정책 핸들러를 도입하는 방식입니다.
- 이를 통해 **다양한 몰 정책을 플러그인 방식으로 추가하거나 제거할 수 있는 구조**를 만들 수 있어 유지보수가 훨씬 유연해집니다.

### 4. **다중 서비스 매핑 지원**

- 하나의 몰이 여러 유형의 비즈니스 정책을 사용하거나, 두 개 이상의 서비스가 결합된 상태로 관리되는 경우가 있을 수 있습니다. `DynamicCommonServiceFactory`에 **다중 서비스 매핑 기능**을 추가하여 몰별로 필요한 여러 서비스를 동시에 반환하거나 조합할 수 있는 기능을 고려해볼 수 있습니다.
- 이 방식은 복합적인 비즈니스 로직을 갖춘 몰에서 특히 유용하게 활용될 수 있습니다.

### 5. **서비스 로깅 및 모니터링 기능 추가**

- 몰별로 각기 다른 비즈니스 로직이 적용되므로, 로깅과 모니터링이 중요해집니다. `DynamicCommonServiceFactory`에서 반환하는 서비스에 대해 **특정 비즈니스 로직 실행 내역을 로깅**하거나 **성능 모니터링**을 추가하여, 몰별 서비스 상태를 실시간으로 모니터링하는 체계를 구축할 수 있습니다.
- 예를 들어, 각 서비스 호출 시점에 실행된 비즈니스 로직 내역이나 성능 데이터를 수집하고 분석하는 구조로 개선할 수 있습니다.

### 6. **동적 서비스 캐싱 전략 도입**

- 동일한 요청에 대해 같은 결과가 반복적으로 호출되는 경우가 있다면 **결과를 캐싱**하여 성능을 향상할 수 있습니다. `DynamicCommonServiceFactory`에 간단한 캐싱 로직을 도입하여, 반복되는 호출을 줄이고 응답 시간을 단축하는 방향으로 개선할 수 있습니다. Redis나 EHCache 같은 캐싱 솔루션을 활용하면 더욱 효과적입니다.