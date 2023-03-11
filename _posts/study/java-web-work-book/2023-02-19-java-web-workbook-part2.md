---
title: "[java-web-workbook] 2장 웹과 데이터베이스"
category: [Book]
tag: [java-web-workbook, db]
toc: true
toc_sticky: true
published: true
---
## NOTE

처음 자바 공부를 할 때 JDBC, 드라이버 세팅, 커넥션풀, preparedStatement 에 대해 공부한 기억이 있다. 

스프링 프레임워크를 사용하면 간단한 설정만으로 DB에 커넥션을 연결하고 mybatis를 사용해서 query를 작성할 수 있다. 편리하게 도와주는 라이브러리들로 인해 점점 기초 사용법과 원리를 잊고 있었다. ~~사실 실무에서 쌩으로 코딩하는 경우는 없긴 하지만..~~

해당 장을 읽으면서 스프링과 그 외 라이브러리들이 내부적으로 어떻게 구현되어 있을지 생각할 수 있는 기회가 되었다. 그리고 추후에 실무에서 프로젝트 초기설정 시 언젠가는 도움이 될 거라 생각한다.

## 2.1 JDBC 프로그래밍 준비

### DTO (Data Transfer Object)

- 여러 개의 데이터를 묶어서 하나의 객채로 전달
- Java Beans 형태로 구성
- 생성자가 없거나 반드시 파라미터가 없는 생성자 함수를 가지는 형태
- 속성(멤버 변수)은 private으로 작성
- getter/setter를 제공할 것

### DAO (Data Access Object)

- 데이터를 전문적으로 처리하는 객체
- 일반적으로 DB의 접근과 처리를 전담하는 객체를 의미 
- VO를 단위로 처리
- DAO를 호출하는 객체는 DAO가 내부에서 어떤식으로 데이터를 처리하는지 알 수없도록 구성

### VO (Value Object) / Entity

- 테이블의 한 row를 자바에서는 하나의 객체가 된다.
- DB에서는 하나의 데이터를 entity라고 하는데 자바에서 유사한 구조의 클래스를 
- 만들어 객체로 처리함 '값을 보관하는 용도' 라는 의미에서 VO 라고함
- 데이터 자체를 의미하기 때문에 getter만을 이용하는 경우가 대부분
        


## 2.2 프로젝트 내 JDBC 구현

### lombok @Cleanup

- try-with-resource 
    - try( resource ) {} 를 이용하여 () 내에 선언된 변수들이 자동으로 close()
    될 수 있는 구조로 작성함
    - try() 내에 선언되는 변수들은 모두 AutoCloseable 라는 인테페이스를 구현한 타입

### Connection Pool

- Connection Pool
    - DB와의 연결을 맺는 작업은 시관과 자원을 쓰기때문에 성능저하가 발생
    - 미리 Connection을 생성해서 보관하고 필요할 때마다 꺼내서 쓰는 방식
- DataSource
    - Connection Pool을 이용하는 라이버러리들은 모두 DataSource 인터페이스를 구현하고있다
- HikariCP 
    - Connection Pool을 사용하기 위한 라이브러리 
    - 스프링 부트에서 기본으로 사용. 안정성이 검증됨
    

### PreparedStatement

- DML 과 쿼리(select)의 차이
    - DML 은 몇 개의 데이터가 처리되었는지 숫자로 결과 반환
    - select 문은 데이터를 반환
- DML(insert/update/delete) 를 싱핼 할때는 executeUpdate() 를 실행
- 쿼리(select) 를 실행 할 떄는 executeQuery() 를 이용해서 ResultSet 을 구한다.

## 2.3 웹 MVC와 JDBC의 결합

### ModelMapper 라이브러리

- DTO -> VO , VO -> DTO 변환시 사용
- getter/setter 등을 이요해서 객체의 정보를 다른 객체로 복사하는 기능 제공

### Log4j2 와 @Log4j2

- 레벨(level) : 중요도  
    FATAL > ERROR > WARN > INFO > DEBUG > TRACE 
- 어펜더(Appender)  
    로그를 어떤 방식으로 기록할 것인지 의미
    콘솔창에 출력할 것인지, 파일로 출력할 것인지 등을 결정함
- Lombok의 경우 @Log4j2 를 이요해서 로그를 적용할 수 있다.
