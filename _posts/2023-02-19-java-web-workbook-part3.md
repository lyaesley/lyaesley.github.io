---
title: "[java-web-workbook] 3장 세션/쿠키/필터/리스너"
category: [Book]
tag: [java-web-workbook, db]
toc: true
toc_sticky: true
published: true
---

# 3.1 세션과 필터
    - 웹은 기본적으로 과거의 상태를 유지하지 않는 무상태(stateless) 연결이다
    - 과거의 상태를 기억하기 위해 세션(HttpSession)이나 쿠키(Cookie)를 이용하기도 하고 특정한 문자(토큰)을 이용한다.
    - 로그인 유지를 위한 모등 기능을 세션 트랙킹(session tracking)이라고 한다.

### 쿠키를 생성하는 방법
    - 서버에서 자동으로 발행
        - WAS 에서 발행되며 이름은 WAS 마다 고유한 이름 사용
        - 톰캣은 'JSESSIONID'라는 이름을 이용
        - 기본적으로 브라우저의 메모리상에 보관. 브라우저를 종료하면 서버에 삭제
    - 개발자가 코드로 직접 발행
        - 이름, 유효기간 지정 가능
        - 반드시 직접 응답(Response)에 추가해야 한다
        - 경로나 도메인 등을 지정 가능

### 필터
    - 필터는 한개 이상 적용 가능
    - Filter 인터페이스의 doFilter 추상메소드를 구현하여 적용


### EL의 Scope와 HttpSession 접근하기
    - EL에서 HttpServletRequest에 setAttribute()로 저장한 객체를 사용할 수 있다.
    - HttpServletRequest에 저장된 객체를 찾을 수 없다면 HttpSession 에 저장된 객체를 찾는다
    - Page Scope: JSP에서 EL을 이용해 <c:set>으로 저장한 변수
    - Request Scope : HttpServletRequest에 setAttribute()로 저장한 변수
    - Session Scope : HttpSession을 이용해서 setAttribute()로 저장한 변수
    - Application Scope : SelvletContext를 이용해서 setAttribute()로 저장한 변수
    - Page -> Request -> Session -> Application 의 순서대로 객체를 찾는다

### 3-2 사용자 정의 쿠키(Cookie)

### 3-3 리스너(Listener)
    - 서블릿 API에는 리스너(Listener)라는 인터페이스들이 존재함
    - 이벤트(Event)라는 특정한 데이터가 발생하면 자동으로 실행되는 특징
    - 옵저버(observer) 패턴 : 특정한 변화를 '구독(subscribe)'하는 객체들을 보관하고 있다가 변화가 발생(발행(publish))이라고 포현)하면 구독 객체들을 실행하는 방식
    


