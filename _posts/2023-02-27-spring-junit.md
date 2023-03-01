---
title: "[Spring] 기본 설정들"
category: [Spring]
tag: [junit, config]
toc: true
toc_sticky: true
published: true
---
작성중...

## JUnit 테스트

```java

@Log4j2
//JUnit5 버전에서 'spring-test'를 사용 (JUnit4버전에서는 @Runwith)
@ExtendWith(SpringExtension.class)
//스프링의 설정 정보를 로딩하기 위해 사용
//xml로 설정했기 때문에 locations 설정을 이용, 자바 설정은 classes 속성 이용
@ContextConfiguration(locations = "file:src/main/webapp/WEB-INF/root-context.xml") 
public class SampleTests {

    @Autowired
    private SampleService sampleService;

    //required = false : 해당 객체를 주입 받지 못하더라도 예외가 발생하지 않는다
    //인텔리제이의 경우 @Service, @Repository..와 같이 Bean으로 등록된 경우가 아니면 경고 발생
    @Autowired(required = false)
    private TimeMapper timeMapper;

    @Test void testService1() {
        log.info(sampleService);
        Assertions.assertNotNull(sampleService);
    }
}

```


