---
title: "스프링 빈 스코프"
category: [Spring]
tag: [spring, 기초]
toc: true
toc_sticky: true
published: true
---
# 스프링 빈 스코프

- 싱글톤 : 기본 스코프, 스프링 컨테이너의 시작과 종료까지 유지되는 가장 넓은 범위의 스코프
- 프로토타입 : 스프링 컨테이너는 프로토타입 빈의 생성과 의존관계 주입까지만 관여하고 더는 관리하지 않는 매우 짧은 범위의 스코프
- 웹 전용
    - request : HTTP 요청 하나가 들어오고 나갈 때 까지 유지되는 스코프, 각각의 HTTP 요청마다 별도의 빈 인스턴스가 생성되고, 관리된다.
    - session : HTTP Session과 동일한 생명주기를 가지는 스코프
    - application : 서블릿 컨텍스트( ServletContext )와 동일한 생명주기를 가지는 스코프
    - websocket : 웹 소켓과 동일한 생명주기를 가지는 스코프

## NOTE
출처 : 김영한 선생님의 스프링 강의
[강의 링크](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%ED%95%B5%EC%8B%AC-%EC%9B%90%EB%A6%AC-%EA%B8%B0%EB%B3%B8%ED%8E%B8/dashboard)



## 요약



---

