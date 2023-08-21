---
title: "프록시 팩토리"
category: [spring]
tag: [spring, 기초, proxy]
toc: true
toc_sticky: true
published: true
---

# 프록시 팩토리

[강의 링크](https://www.inflearn.com/course/lecture?courseSlug=%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8&unitId=94462&tab=curriculum)

## 프록시 팩토리 - 소개

- 문제점
  - 인터페이스가 있는 경우에는 JDK 동적 프록시를 적용하고, 그렇지 않은 경우에는 CGLIB를 적용하려면 어떻게 해야할까?
  - 두 기술을 함께 사용할 때 부가 기능을 제공하기 위해 JDK 동적 프록시가 제공하는 InvocationHandler 와 CGLIB가 제공하는 MethodInterceptor 를 각각 중복으로 만들어서 관리해야 할까?
  - 특정 조건에 맞을 때 프록시 로직을 적용하는 기능도 공통으로 제공되었으면?


```java
import lombok.extern.slf4j.Slf4j;
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;
@Slf4j
public class TimeAdvice implements MethodInterceptor {
  @Override
  public Object invoke(MethodInvocation invocation) throws Throwable {
    log.info("TimeProxy 실행");
    long startTime = System.currentTimeMillis();
    Object result = invocation.proceed();
    long endTime = System.currentTimeMillis();
    long resultTime = endTime - startTime;
    log.info("TimeProxy 종료 resultTime={}ms", resultTime);
    return result;
  }
}
```

- TimeAdvice 는 앞서 설명한 MethodInterceptor 인터페이스를 구현한다. 패키지 이름에 주의하자.
- Object result = invocation.proceed()
  - invocation.proceed() 를 호출하면 target 클래스를 호출하고 그 결과를 받는다.
  - 그런데 기존에 보았던 코드들과 다르게 target 클래스의 정보가 보이지 않는다. target 클래스의 정보는 MethodInvocation invocation 안에 모두 포함되어 있다.
  - 그 이유는 바로 다음에 확인할 수 있는데, 프록시 팩토리로 프록시를 생성하는 단계에서 이미 target 정보를 파라미터로 전달받기 때문이다

```java
@Slf4j
public class ProxyFactoryTest {

  @Test
  @DisplayName("인터페이스가 있으면 JDK 동적 프록시 사용")
  void interfaceProxy() {

    ServiceInterface target = new ServiceImpl();
    ProxyFactory proxyFactory = new ProxyFactory(target);
    proxyFactory.addAdvice(new TimeAdvice());
    ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();
    log.info("targetClass={}", target.getClass());
    log.info("proxyClass={}", proxy.getClass());

    proxy.save();

    assertThat(AopUtils.isAopProxy(proxy)).isTrue();
    assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue();
    assertThat(AopUtils.isCglibProxy(proxy)).isFalse();

  }
}
```

- new ProxyFactory(target) : 프록시 팩토리를 생성할 때, 생성자에 프록시의 호출 대상을 함께
넘겨준다. 프록시 팩토리는 이 인스턴스 정보를 기반으로 프록시를 만들어낸다. 만약 이 인스턴스에
인터페이스가 있다면 JDK 동적 프록시를 기본으로 사용하고 인터페이스가 없고 구체 클래스만 있다면
CGLIB를 통해서 동적 프록시를 생성한다. 여기서는 target 이 new ServiceImpl() 의 인스턴스이기
때문에 ServiceInterface 인터페이스가 있다. 따라서 이 인터페이스를 기반으로 JDK 동적 프록시를
생성한다.
- proxyFactory.addAdvice(new TimeAdvice()) : 프록시 팩토리를 통해서 만든 프록시가 사용할 부가
기능 로직을 설정한다. JDK 동적 프록시가 제공하는 InvocationHandler 와 CGLIB가 제공하는
MethodInterceptor 의 개념과 유사하다. 이렇게 프록시가 제공하는 부가 기능 로직을 어드바이스
( Advice )라 한다. 번역하면 조언을 해준다고 생각하면 된다.
- proxyFactory.getProxy() : 프록시 객체를 생성하고 그 결과를 받는다
