---
title: "Spring Data JPA 설정"
category: [Spring boot]
tag: [jpa, db, spring boot]
toc: true
toc_sticky: true
published: true
---
작성중...

## Spring Data JPA 를 위한 설정

### application.properties 설정 
```properties
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.show-sql=true
```

- spring.jpa.hibernate.ddl-auto

    |   속성값          |    의미    |
    |   :---            |   :---    |
    |    none           |   DDL을 하지 않음                                                 |
    |    create-drop    |   실행할 때 DDL을 실행하고 종료 시에 만들어진 테이블 등을 모두 삭제   |
    |    create         |   실행할 때마다 새롭게 테이블을 생성                                |
    |    update         |   기존과 다르게 변경된 부분이 있을 때는 새로 생성                    |
    |    validate       |   변견된 부분만 알려주고 종료                                      |

    - update : 테이블이 없을 때는 자동으로 생성하고 변경이 필요할 때는 alter table이 실행됨. 인덱스나 외래키 등도 자동 처리됨

- spring.jpa.properties.hibernate.format_sql=true
    - 실제로 실행되는 SQL을 포맷팅해서 알아보기 쉽게 출력

- spring.jpa.show-sql=true
    - JPA가 실행하는 SQL을 같이 출력

