---
title: "리플렉션"
category: [Java]
tag: [reflection]
toc: true
toc_sticky: true
published: true
---

# 리플렉션

## NOTE

출처 : 김영한 선생님의 스프링 강의
[강의 링크](https://www.inflearn.com/course/lecture?courseSlug=%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B3%A0%EA%B8%89%ED%8E%B8&unitId=94462&tab=curriculum)

- 클래스나 메서드의 메타정보를 동적으로 획득하고, 동적으로 호출할 수 있다.

```java
    @Test
    void reflection2() throws Exception{
        //클래스 정보
        Class<?> classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");
        Hello target = new Hello();

        //callA 메서드 정보
        Method methodCallA = classHello.getMethod("callA"); //해당 클래스의 callA 메서드 메타정보 획득
        dynamicCall(methodCallA, target);

        //callB 메서드 정보
        Method methodCallB = classHello.getMethod("callB");
        dynamicCall(methodCallB, target);

    }

    private void dynamicCall(Method method, Object target) throws Exception{
        log.info("start");
        Object result = method.invoke(target);  //획득한 메서드 메타정보로 실제 인스턴스의 메서드를 호출
        log.info("result={}", result);
    }

    @Slf4j
    static class Hello {
        public String callA() {
            log.info("callA");
            return "A";
        }
        public String callB() {
            log.info("callB");
            return "B";
        }
    }
```

- 리플렉션을 사용해서 Method라는 메타정보로 추상화가 가능
- 덕분에 공통 로직을 만들 수 있다

## 주의

- 컴파일 시점에 오류를 잡을 수 없다.
- "callA" 의 문자를 실수로 "callZZ"로 작성해도 컴파일 오류가 발생하지 않고 런타임 오류가 발생
- 프로그래밍 언어가 타입정보를 기반으로 컴파일 시점에 오류를 잡아준 덕분에 편하게 개발 가능했는데 리플렉션은 그것에 역행하는 방식
- 리플렉션은 프레임워크 개발이나 또는 매우 일반적인 공통 처리가 필요할 때 부분적으로 주의해서 사용
