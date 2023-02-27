---
title: "[java-web-workbook] 4장 스프링과 스프링 Web MVC"
category: [Book]
tag: [java-web-workbook, db]
toc: true
toc_sticky: true
published: true
---
작성중...

## 4.1 의존성 주입과 스프링

### 의존성 주입(dependency injection)

- 스프링 프레임워크는 자체적으로 객체를 생성하고 관리하면서 필요한 곳으로 객체를 주입(inject)하는 역할을 한다.
- @Autowired 어노테이션으로 의존성 주입 

```java
// 해당 타입(SampleService)의 빈(Bean)이 존재하면 여기에 주입
@Autowired private SampleService sampleService;
```
    
### ApplicationContext 와 빈(Bean)

- 스프링의 빈 설정은 XML을 이요하거나 별도의 클래스를 이용하는 자바 설정이 가능하다.
- 빈(Bean) 객체들을 관리하기 위해서 `ApplicationContext` 존재(객체(공간))을 활용

### <context:component-scan base-package="com.lyae"/>

- 해당 패키지를 스캔해서 스프링의 어노테이션들을 인식한다
- @Controller, @Service, @Repository, @Component ..

### 생성자 주입방식

- 초기 스프링에서는 `@AutoWried` 를 멤버 변수에 할당하거나, Setter를 작성하는 방식을 많이 이용했지만, 스프링3 이후에는 생성자 주입방식을 많이 사용
- 생성자 주입방식
    - 주입 받아야 하는 객체의 변수는 `final`로 작성한다
    - 생성자를 이용해서 해당 변수를 생정자의 파라미터로 지정한다.
    - 객체를 생성할 때 문제가 발생하는지를 미리 확인할 수 있어 선호되는 방식
    - `Lombok` 의 `@RequiredArgsConstructor`를 이용해 필요한 생성자 함수를 자동 작성 가능

### 웹 프로젝트를 위한 스프링 준비

- `ApplicationContext`가 웹 애플리케이션에서 동작하려면 웹 애플리케이션이 실행될 때 스프링을 로딩해서 해당 웹 애플리케이션 내부에 스프링의 `ApplicationContext`를 생성하는 작업이 필요하다. 
- 이를 위해 web.xml을 이용해서 리스너를 설정한다.
- 스프링 프레임워크의 웹과 관련된 작업은 `spring-webmvc` 라이브러리르 추가해야 설정 가능하다.

## 4.2 MyBatis와 스프링 연동

- 다음과 같은 라이브러리들이 필요하다.
    - 스프링관련 : `spring-jdbc`, `spring-tx`
    - MyBatis관련 : `mybatis`, `mybatis-spring`
- 스프링에 설정해둔 `HikariDataSource`를 이용해서 `SqlSessionFactory`라는 빈(Bean)을 설정

### XML로 SQL분리하기

- 매퍼 인터페이스를 정의하고 메소드를 선언
- 해당 XML 파일을 작성(파일 이름과 인터페이스 이름을 같게)하고 `<select>`와 같은 태그를 이용
- `<select>`, `<insert>` 등의 태그에 id 속성 값을 매퍼 인터페이스의 메소드 이름과 같게 작성
- `root-context.xml`에 있는 MyBatis 설정에 XML 파일들을 인식하도록 설정 추가

## 4.3 스프링 Web MVC 기초

### DispatcherServlet 과 Front Controller

### 파라미터 자동 수집과 변환

### 스프링 MVC에서 주로 사용하는 어노테이션들

## 4.4 스프링 Web MVC 구현하기



