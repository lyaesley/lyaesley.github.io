---
title: "[java-web-workbook] 2장 웹과 데이터베이스"
category: [Book]
tag: [java-web-workbook, db]
toc: true
toc_sticky: true
published: true
---
작성중...

# NOTE

처음 자바 공부를 할 때 JDBC, 드라이버 세팅, 커넥션풀, preparedStatement 에 대해 공부한 기억이 있다. 

스프링 프레임워크를 사용하면 간단한 설정만으로 DB에 커넥션을 연결하고 mybatis를 사용해서 query를 작성할 수 있다. 편리하게 도와주는 라이브러리들로 인해 점점 기초 사용법과 원리를 잊고 있었다. ~~사실 실무에서 쌩으로 코딩하는 경우는 없긴 하지만..~~

해당 장을 읽으면서 스프링과 그 외 라이브러리들이 내부적으로 어떻게 구현되어 있을지 생각할 수 있는 기회가 되었다. 그리고 추후에 실무에서 프로젝트 초기설정 시 언젠가는 도움이 될 거라 생각한다.

# 2.1 JDBC 프로그래밍 준비
    - DAO
    - DTO
    - VO/Entity

# 2.2 프로젝트 내 JDBC 구현

### lombok @Cleanup

### Connection Pool
    - HikariCP 
    - DataSource

### PreparedStatement
    - DML 과 쿼리(select)의 차이
        - DML 은 몇 개의 데이터가 처리되었는지 숫자로 결과 반환
        - select 문은 데이터를 반환
    - DML(insert/update/delete) 를 싱핼 할때는 executeUpdate() 를 실행
    - 쿼리(select) 를 실행 할 떄는 executeQuery() 를 이용해서 ResultSet 을 구한다.

# 2.3 웹 MVC와 JDBC의 결합

