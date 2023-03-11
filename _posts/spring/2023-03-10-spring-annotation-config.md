---
title: "[Spring] annotation 기록"
category: [Springboot]
tag: [config, annotation]
toc: true
toc_sticky: true
published: true
---

사용하면서 헷갈리는 경우가 있어 정리한다.  
상시 업데이트중..

## 컨트롤러 선언부에 사용하는 어노테이션


## 메소드 선언부에 사용하는 어노테이션

###  @PostMapping(value = "/", consumes = MediaType.APPLICATION_JSON_VALUE)

- consumes 
    - 해당 메소드가 어떤 데이터를 처리하는지 명시
    -  MediaType.APPLICATION_JSON_VALUE : JSON 타입의 데이터를 처리하겠다

## 메소드의 파라미터에 사용하는 어노테이션

### @RequestBody 

- JSON 문자열을 해당 오브젝트로 변환해줌
- 아래 코드에서 JSON 형태로 파라미터가 넘어오면 ReplyDTO 로 변환

```java
    @PostMapping(value = "/", consumes = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<Map<String,Long>> register(@RequestBody ReplyDTO replyDTO) {

        log.info(replyDTO);
        Map<String, Long> resultMap = Map.of("rno", 111L);

        return ResponseEntity.ok(resultMap);
    }
```
