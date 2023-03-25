---
title: "BeanFactory와 ApplicationContext"
category: [Spring]
tag: [spring, 기초]
toc: true
toc_sticky: true
published: true
---
# BeanFactory와 ApplicationContext

## NOTE

출처 : 김영한 선생님의 스프링 강의
[강의 링크](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)

스프링 프레임워크를 사용하면서 빈(Bean) 이라는 단어를 무조건 들어봤을것이다.  
스프링을 깊이 이해하기위해 필요한 개념으로 생각되어 강의 내용을 참고하여 정리해보았다.

애플리케이션 개발중 getBean()을 직접 호출하는 일은 없을것이다.
스프링에서 제공하는 @Autowired 를 사용하거나 스프링 내부에서 자동으로 Bean 관련 처리를 해준다.
그럼에도 개념 정리는 꼭 필요하다고 생각한다.



## BeanFactory

- 스프링 컨테이너의 최상위 인터페이스다.
- 스프링 빈을 관리하고 조회하는 역할을 담당한다.
- getBean() 을 제공한다.

## ApplicationContext

- BeanFactory 기능을 모두 상속받아서 제공한다.
- 빈을 관리하고 검색하는 기능을 BeanFactory가 제공해주는데, 그러면 둘의 차이가 뭘까?
- 애플리케이션을 개발할 때는 빈을 관리하고 조회하는 기능은 물론이고, 수 많은 부가기능이 필요하다.

## ApplicatonContext가 제공하는 부가기능

### 메시지소스를 활용한 국제화 기능

- 예를 들어서 한국에서 들어오면 한국어로, 영어권에서 들어오면 영어로 출력

### 환경변수

- 로컬, 개발, 운영등을 구분해서 처리

### 애플리케이션 이벤트

- 이벤트를 발행하고 구독하는 모델을 편리하게 지원

### 편리한 리소스 조회

- 파일, 클래스패스, 외부 등에서 리소스를 편리하게 조회

## 정리

- ApplicationContext는 BeanFactory의 기능을 상속받는다.
- ApplicationContext는 빈 관리기능 + 편리한 부가 기능을 제공한다.
- BeanFactory를 직접 사용할 일은 거의 없다. 부가기능이 포함된 ApplicationContext를 사용한다.
- BeanFactory나 ApplicationContext를 스프링 컨테이너라 한다.