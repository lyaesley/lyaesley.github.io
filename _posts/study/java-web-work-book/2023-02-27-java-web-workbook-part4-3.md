---
title: "[java-web-workbook] 4장 스프링과 스프링 Web MVC 4.3장"
category: [Book]
tag: [java-web-workbook, spring]
toc: true
toc_sticky: true
published: true
---

## 4.3 스프링 Web MVC 기초

스프링 MVC가 기존 구조에 약간의 변화를 주는 부분
- Front-Controller 패턴을 이용해서 모든 흐름의 사전/사후 처리를 가능하도록 설계된 점
- 어노테이션을 적극적으로 활용해서 최소한의 코드로 많은 처리가 가능하도록 설계된 점
- HttpServletRequest/HttpServletResponse를 이용하지 않아도 될 만큼 추상화된 방식으로 개발 가능

> 스프링 MVC 에서 가장 중요한 사실은 모든 요청(Request)이 반드시 DispatcherServlet 이라는 존재를 통해서 실행된다

- 3 tier 구조를 분리하듯 별도의 설정파일로 구성 (servlet-context.xml)

### DispatcherServlet 과 Front Controller

- 스프링 MVC 에서는 DispatcherServlet 이라는 객체가 프론트 컨트롤러의 역할을 수행한다.

### Formatter를 이용한 파라미터의 커스텀 처리

- 기본적으로 HTTP는 문자열로 데이터를 전달함
- 컨트롤러는 문자열을 기준으로 특정한 클래스의 객체로 처리하는 작업이 진행됨
- 가장 문제가 되는 타입이 날짜 관련 타입
- Formatter 인터페이스를 구현 parse(), print() 메소드 존재

### 객체 자료형의 파라미터 수집

```java
@PostMapping("/register")
    // TodoDTO 와 같이 다양한 타입의 멤버 변수들이 자동 처리된다.
    public void registerPost(TodoDTO todoDTO) {
        log.info("POST todo register.....");
        log.info(todoDTO);
    }
```

### Model 객체

- 뷰(View)(JSP) 까지 데이터 전달. addAttribute(이름,객체) 전달
- Model 에 담기는 데이터는 내부적으로 HttpServletRequest의 setAttribute()와 동일한 동작을 수행
- 초기 스프링 MVC 에서는 ModelAndView 를 사용했는데 스프링 MVC 3 버전 이후에는 Model 사용
- Java Beans 와 @ModelAttribute
    - 스프링 MVC의 컨트롤러는 파라미터로 getter/setter를 이용하는 **Java Beans**의 형식의 사용자 정의 클래스가 파라미터인 경우 **자동으로 화면까지 객체를 전달**
    ```java
    // 별도의 처리(model.addAttribute('todoDTO', todoDTO)) 하지 않아도
    // View 에서 ${todoDTO}를 사용 가능.
    // 특정한 이름을 사용하려면 ModelAttribute로 변수명 지정
    @GetMapping("/ex4_1")
    public void ex4Extra(@ModelAttribute("dto") TodoDTO todoDTO, Model model) {
        log.info("ex4_1.....");
        log.info(todoDTO);
    }
    ```

### RedirectAttributes와 리다이렉션

- RedirectAttributes 역시 Model 과 마찬가지로 파라미터로 추가하면 자동으로 생성
- addAttribute(키,값) : 리다이렉트할 때 쿼리 스트링이 되는 값을 지정
- addFlashAttribute(키,값) : 일회용으로만 데이터를 전달하고 삭제되는 값을 지정

### 다양한 리턴 타입

- void 
    - @RequestMapping 값과 @GetMapping 등 메소드에서 선언된 값을 그대로 뷰(View)의 이름으로 사용
    - 상황에 관계없이 동일한 화면을 보여주는 경우
- 문자열(String)
    - 상황에 따라서 다른 화면을 보여주는 경우
    - 특별한 접두어 사용 가능
        - redirect: 리다이렉션을 이용하는경우
        - forward: 브라우저의 URL은 고정하고 내부적으로 다른 URL로 처리하는 경우
- 객체나 배열, 기본 자료형
- ResponseEntity
- 화면이 따로 있는 경우 주로 void나 문자열 사용
- JSON 타입을 활용할 때는 객체나 ResponseEntity

### 스프링 MVC에서 주로 사용하는 어노테이션들

- 컨트롤러 선언부에 사용하는 어노테이션
    - @Controller: 스프링 빈의 처리됨을 명시
    - @RestController : REST 방식의 처리를 위한 컨트롤러임을 명시
    - @RequestMapping : 특정한 URL 패턴에 맞는 컨트롤러임을 명시

- 메소드 선언부에 사용하는 어노테이션
    - @GetMapping/@PostMapping/@DeleteMapping/@putMapping
        - HTTP 전송 방식(method)에 따라 해당 메소드를 지정하는 경우 사용
    - @RequestMapping : GET/POST 방식 모두를 지원하는 경우 사용
    - @ReponseBody : REST 방식에서 사용

- 메소드의 파라미터에 사용하는 어노테이션
    - @RequestParam : Request에 있는 특정한 이름의 데이터를 파라미터로 받아서 처리하는 경우 사용
    - @PathVariable : URL 경로의 일부를 변수로 삼아서 처리하기 위해서 사용
    - **@ModelAttribute** : 해당 파라미터는 반드시 Model에 포함되어서 다시 뷰(View)로 전달됨을 명시(주로 기본 자료형이나 Wrapper 클래스, 문자열에 사용)
    - 기타 : @SessionAttribute, @Valid, @RequestBody 등

### 스프링 MVC의 예외 처리

- 가장 일반적인 방식은 `@ControllerAdvice` 를 이용
- `@ControllerAdvice`가 선언된 클래스 역시 스프링의 빈(Bean)으로 처리

```java
@ControllerAdvice
@Log4j2
public class CommonExceptionAdvice {

    @ResponseBody
    //해당 타입의 예외를 파라미터로 전달받을 수 있다.
    @ExceptionHandler(NumberFormatException.class)
    public String exceptNumber(NumberFormatException numberFormatException) {

        log.error("------------------------------");
        log.error(numberFormatException.getMessage());

        return "NUMBER FORMAT EXCEPTION";
    }
}
```

### 404에러 페이지와 @ResponseStatus

- `@ResponseStatus`를 이용하면 404 상태에 맞는 화면을 별도로 작성 가능
- `web.xml <servlet>` 태그에 `<init-param> throwExceptionIfNoHandlerFound` 파라미터 추가 

```java
    @ExceptionHandler(NoHandlerFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public String notFound() {
        return "custom404";
    }
```
