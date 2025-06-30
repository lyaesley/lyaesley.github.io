---
title: "서버 개발 비동기연동"
category: [server]
tag: [server, backend, async]
toc: true
toc_sticky: true
published: true
---

# 서버 개발 실무 지식 - 비동기 연동

## 비동기 방식 연동 예시

1. 연동에 얀간의 시차가 생겨도 문제가 되지 않는다.
2. 일부 기능은 실패했을 때 재시도가 가능하다.
3. 연동에 실패했을 때 나중에 수동으로 처리할 수 있는 기능도 있다.
4. 연동에 실패했을 때 무시해도 되는 기등도 있다.

> 외부 연동이 위 4가지 특징 중 일부에 해당하면 비동기로 처리할 수 잇는지 검토한다.

## 비동기 연동의 다양한 방식

1. 별도 스레드로 실행하기
2. 메시징 시스템 이용하기
3. 트랜잭션 아웃박스 패턴 사용하기
4. 배치로 연동하기
5. CDC 이용하기

### 별도의 스레드로 실행하기

- 별도 스레드를 생성하거나 스프링이라면 @Async 애노테이션을 이용
  ```java
    new Thread(() -> pushClient.sendPush(pushData)).start();
  ```
  ```java
    @Async
    public void sendPushAsync(PushData pushData) {
      pushClient.sendPush(pushData);
      // ... 기타 코드
    }
  ```
- @Async 를 사용할때는 메서드 이름에 비동기 관련된 단어를 추가하는 것이 좋다.
  - 비동기로 실행되면 (비동기를 호출하는 메서드에서) catch 블록이 실행되지 않는다.
  ```java
  public OrderResult placeOrder(OrderRequest req) {
    //주문 생성 처리
    try {
      pushService.sendPush(pushData); // 비동기 메서드
    } catch(Exception ex) {
      //sendPush()가 비동기로 실행되므로 catch 블록은 동작하지 않는다.
      //에러 처리 코드
    }
    return successResult(...); // 푸시 발송을 기다리지 않고 리턴
  }
  ```
- 비동기로 실행되는 코드는 연동 과정에서 발생하는 오류를 직접 처리해야 한다.

  ```java
  @Async
  public void sendPushAsync(PushData pushData) {
      try {
        pushClient.sendPush(pushData);
      } catch(Exception e) {
        try {
            Thread.sleep(500);
        } catch(Exception ex) {}
        try {
          pushClient.sendPush(pushData); // 재시도를 하거나
        } catch(Exception e1) {
          // 실패 로그로 남기거나
        }
      }
  }
  ```


---
**참고 자료:**
1. 최범균: 주니어 백엔드 개발자가 반드시 알아야 할 실무 지식