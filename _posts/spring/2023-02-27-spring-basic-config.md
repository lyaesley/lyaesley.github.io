---
title: "[Spring] 기본 설정들"
category: [Spring]
tag: [junit, config]
toc: true
toc_sticky: true
published: true
---

## 스프링 MVC

### root-context.xml 파일

- 스프링의 빈 들을 어떻게 관리할 것인지를 설정하는 파일

```xml
    <!-- 해당 패키지를 스캔해서 스프링의 어노테이션들을 인식 -->
    <context:component-scan base-package="com.lyae.workbook.springex.sample"/>

    <!-- mybatis 설정에 XML(메퍼XML) 파일들을 인식하도록 설정 -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <property name="mapperLocations" value="classpath:/mappers/**/*.xml"></property>
    </bean>

    <!-- mybatis 메퍼 인터페이스를 설정했는지 작성 -->
    <mybatis:scan base-package="com.lyae.workbook.springex.mapper"></mybatis:scan>
```

### servlet-context.xml 파일

- 스프링 MVC 에 관련된 설정을 작성

```xml
    <!-- 스프링 MVC 설정을 어노테이션 기반으로 처리한다는 의미 -->
    <!-- 스프링 MVC 의 여러 객체들을 자동으로 스프링의 빈(Bean)으로 등록하게 하는 기능 -->
    <mvc:annotation-driven></mvc:annotation-driven>
    
    <!-- 이미지나 html 파일과 같이 정적인 파일의 경로를 지정 -->
    <!-- 스프링 MVC 에서 처리하지 않는다는 의미 -->
    <!-- location 속성 값은 webapp 폴더에 만들어둔 폴더를 의미 -->
    <mvc:resources mapping="/resources/**" location="/resources/"></mvc:resources>

    <!-- 뷰(view) 를 어떻게 결정하는지에 대한 설정 -->
    <!-- /WEB_INF/....jsp 와 같은 설정을 생략 가능 -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/views/"></property>
        <property name="suffix" value=".jsp"></property>
    </bean>
```

### web.xml 파일

```xml
    <servlet>
        <servlet-name>appServlet</servlet-name>
        <!-- DispatcherServlet 이 로딩할 때 servlet-context.xml을 이용하도록 설정 -->
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/servlet-context.xml</param-value>
        </init-param>

        <init-param>
            <param-name>throwExceptionIfNoHandlerFound</param-name>
            <param-value>true</param-value>
        </init-param>

        <!-- 톰캣 로딩 시에 클래스를 미리 로딩해 두기 위한 설정 -->
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>appServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

    <filter>
        <filter-name>encoding</filter-name>
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
        <init-param>
            <param-name>encoding</param-name>
            <param-value>UTF-8</param-value>
        </init-param>
    </filter>

    <filter-mapping>
        <filter-name>encoding</filter-name>
        <servlet-name>appServlet</servlet-name>
    </filter-mapping>
```



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


