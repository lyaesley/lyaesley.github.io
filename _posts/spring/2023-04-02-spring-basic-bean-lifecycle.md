---
title: "스프링 빈 생명주기 콜백"
category: [Spring]
tag: [spring, 기초]
toc: true
toc_sticky: true
published: true
---
# 스프링 빈 생명주기 콜백

## NOTE
출처 : 김영한 선생님의 스프링 강의
[강의 링크](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)

>데이터베이스 커넥션 풀이나, 네트워크 소켓처럼 애플리케이션 시작 시점에 필요한 연결을 미리 해두고,
애플리케이션 종료 시점에 연결을 모두 종료하는 작업을 진행하려면, 객체의 초기화와 종료 작업이
필요하다.

> 스프링 빈의 이벤트 라이프사이클  
> 스프링 컨테이너 생성 -> 스프링 빈 생성 -> 의존관계 주입 -> 초기화 콜백 사용 ->소멸전 콜백 스프링 종료  
> 초기화 콜백: 빈이 생성되고, 빈의 의존관계 주입이 완료된 후 호출   
> 소멸전 콜백: 빈이 소멸되기 직전에 호출

## 요약

> @PostConstruct, @PreDestroy 애노테이션을 사용하자  
> 코드를 고칠 수 없는 외부 라이브러리를 초기화, 종료해야 하면 @Bean 의 initMethod , destroyMethod
를 사용하자.

---

## 어노테이션 @PostConstruct, @PreDestroy

> 이거 쓰세요!

- 최신 스프링에서 권장
- 자바 표준
- 유일한 단점은 외부 라이브러리에는 적용하지 못함, 외부 라이브러리에 필요하다면 @Bean 을 이용하자.

## 빈 등록 초기화, 소멸 메서드

> 설정 정보에 @Bean(initMethod = "init", destroyMethod = "close") 처럼 초기화, 소멸 메서드를
지정할 수 있다.  
> @Bean의 destroyMethod 속성에는 아주 특별한 기능이 있다
> destroyMethod 속성을 지정하지 않으면 디폴트로 `close`, `shutdown` 이름의 메서드르 자동 호출한다.

## 인터페이스 사용

인터페이스를 사용하는 초기화, 종료 방법은 스프링 초창기에 나온 방법들이고, 지금은 다음의 더
나은 방법들이 있어서 거의 사용하지 않는다.

### InitializingBean

```java
@Override public void afterPropertiesSet() throws Exception {
    System.out.println("afterPropertiesSet");
    //초기화 콜백
}
```

### DisposableBean
```java
@Override public void destroy() throws Exception {
    System.out.println("destroy");
   //소멸전 콜백
}
```

