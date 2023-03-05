---
title: "[java-web-workbook] 5장 스프링에서 스프링 부트로"
category: [Book]
tag: [java-web-workbook, spring boot]
toc: true
toc_sticky: true
published: true
---
작성중...

## 5.1 스프링 부트 소개

- 중요한 특징은 Auto Configuration(자동 설정) 이다.
- 모듈을 추가하면 자동으로 관련 설정을 찾아서 실행한다.
- 내장 톰캣과 단독 실행 가능한 도구(물론 다른 WAS 에 스프링 부트를 올려서 실행 가능)
- application.properties 또는 application.yml 파일을 이용하여 설정 가능
    - @Configuration이 있는 클래스 파일을 만들어서 필요한 설정을 추가 가능


## 스프링 부트 기본으로 추가되는 기능

- Log4j2 (application.properties 설정 파일에서 로그 설정 가능)
    - ex. logging.level.org.springframework=info
- 테스트 관련 (JUnit..)

### 스프링 부트의 프로젝트 생성 방식

- Spring Initializr 를 이용한 자동 생성 (주로 이용, IDE 에서 지원)
- Maven 이나 Gradle 을 이용한 직접 생성

### 스프링 부트에서 웹 개발

- web.xml 이나 servlet-context.xml 과 같은 웹 관련 설정 파일들이 없기때문에 이를 대신하는 클래스를 작성한다
