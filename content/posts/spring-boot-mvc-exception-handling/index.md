+++
title = "[Spring Boot] 예외(Exception) 종류와 공통 예외 처리"
date = "2025-04-18"
draft = false
tags = ["Spring Boot", "Java"]
+++

#### 예외 종류를 정리하게 된 이유

API를 구현하다 보면 원하는 시점에 직접 예외를 발생시켜야 할 때도 있고, 예상하지 못한 예외가 발생했을 때도 정해진 형식으로 응답해야 할 때가 많다. 이 과정에서 Exception에도 여러 종류가 있다는 걸 알게 되어 이번 기회에 정리해보고, 이를 바탕으로 공통 예외 처리는 어떻게 구현했는지도 함께 남겨본다.

#### Checked Exception vs Unchecked Exception

| 종류 | 설명 |
|------|------|
| **Checked Exception** | `java.lang.Exception`을 상속받는다. `try-catch` 또는 `throws`를 사용해서 반드시 예외 처리를 해줘야 한다. |
| **Unchecked Exception** | `java.lang.RuntimeException`을 상속받는다. 처리하지 않아도 컴파일 에러가 나지 않고, 런타임 단계에서 확인할 수 있다. |

```java
// Checked Exception
public MemberDto convert(String json) throws JsonProcessingException {
    return objectMapper.readValue(json, MemberDto.class);
}

// Unchecked Exception
public int parse(String value) {
    return Integer.parseInt(value);
}
```

`objectMapper.readValue`가 던지는 `JsonProcessingException`은 Checked Exception이라, `throws`로 선언하지 않으면 컴파일 에러가 난다. 반면 `Integer.parseInt`가 던지는 `NumberFormatException`은 Unchecked Exception이라 별도로 선언하거나 처리하지 않아도 컴파일에는 문제가 없다.

`Checked Exception`은 컴파일 시점에 처리를 강제하기 때문에 놓치는 예외 없이 안전하지만, 그만큼 매번 `try-catch`나 `throws`를 붙여줘야 한다. 반면 `Unchecked Exception`은 컴파일러가 처리를 강제하지 않기 때문에, 코드가 실행되는 도중 원하는 시점에 자유롭게 예외를 발생시킬 수 있다.

#### CustomException 만들기

원하는 시점에 직접 예외를 발생시키려면 커스텀 예외 클래스가 필요했다. 컴파일러가 처리를 강제하지 않아야 코드 흐름 중 원하는 곳에서 자유롭게 던질 수 있으므로, `RuntimeException`을 상속받는 `CustomException` 클래스를 구현했다.

```java
@Getter
@AllArgsConstructor
public class CustomException extends RuntimeException {
    private String errorCode;
    private String errorMsg;
}
```

```java
throw new CustomException("ERROR-AUTH", "인증 오류입니다.");
```

에러 코드와 에러 메시지를 필드로 가지도록 했고, `@AllArgsConstructor`로 두 값을 모두 채워서 생성하도록 했다. 이렇게 만든 `CustomException`을 `throw`로 던지면, 원하는 시점에 원하는 정보를 담아 예외를 발생시킬 수 있다.

#### 예외를 전역으로 관리하기

`throw`로 예외를 발생시킬 수는 있게 되었지만, 이 예외를 여기저기서 각자 처리하는 게 아니라 한 곳에서 전역으로 관리하고 싶었다. Spring에서는 `@ExceptionHandler`와 `@ControllerAdvice`로 이를 지원한다. `@ExceptionHandler`는 발생한 예외를 메소드 단위로 처리할 수 있게 해주고, `@ControllerAdvice`는 이 처리를 여러 컨트롤러에 걸쳐 전역으로 적용할 수 있게 해준다. 그리고 `@ControllerAdvice`에 `@ResponseBody`가 결합된 `@RestControllerAdvice`를 쓰면, 처리 결과를 뷰가 아니라 JSON 같은 데이터로 바로 응답할 수 있다.

```java
@Slf4j
@RestControllerAdvice(basePackages = "com.example.domain.controller")
public class GlobalExceptionHandler {
}
```

특정 도메인의 컨트롤러에서 발생하는 예외만 처리하고 싶어서, `basePackages` 옵션으로 대상 패키지 경로를 문자열로 지정했다. 다만 문자열로 경로를 적으면 나중에 패키지 구조가 바뀌어도 컴파일 타임에는 알아챌 수 없다는 단점이 있다. 이런 경우에는 `basePackageClasses`로 특정 클래스를 기준 삼아 그 클래스가 속한 패키지를 지정할 수도 있는데, 이렇게 하면 패키지 경로가 바뀌었을 때 컴파일 에러로 바로 확인할 수 있다.

#### 예외별로 처리하고 응답 반환하기

`GlobalExceptionHandler`에 실제 처리 메소드를 채워 넣었다. 먼저 필수 파라미터 누락이나 허용되지 않는 타입처럼, Spring에서 자체적으로 던지는 예외부터 원하는 형식으로 응답하도록 했다.

```java
@ExceptionHandler(value = {
        MissingServletRequestParameterException.class
})
public ResponseEntity<ResponseDto<Object>> missingRequiredParameterException() {
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ResponseDto<>(
                    "ERROR-PARAMETER",
                    "필수 파라미터가 누락되었습니다.",
                    null
            ));
}

@ExceptionHandler(value = {
        NumberFormatException.class,
        MethodArgumentTypeMismatchException.class
})
public ResponseEntity<ResponseDto<Object>> notAllowedTypeException() {
    return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ResponseDto<>(
                    "NOT-ALLOW-TYPE",
                    "허용하지 않는 데이터 타입입니다.",
                    null
            ));
}
```

`@ExceptionHandler`에 처리하고 싶은 예외 클래스를 지정하고, `ResponseEntity.status(HttpStatus.BAD_REQUEST)`로 실제 HTTP 상태 코드까지 함께 지정해서 `ResponseDto`를 응답했다. `MissingServletRequestParameterException`은 필수 파라미터가 누락됐을 때, `NumberFormatException`과 `MethodArgumentTypeMismatchException`은 허용되지 않는 타입의 값이 들어왔을 때 발생한다.

그리고 앞서 만든 `CustomException`을 포함해서, 위에서 따로 처리하지 않은 나머지 모든 예외까지 함께 처리하는 메소드도 추가했다.

```java
@ExceptionHandler(value = {
        Exception.class
})
public ResponseEntity<ResponseDto<Object>> exception(Exception exception) {
    if (exception instanceof CustomException customException) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ResponseDto<>(
                        customException.getErrorCode(),
                        customException.getErrorMsg(),
                        null
                ));
    }

    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ResponseDto<>(
                    "ERR-SERVER",
                    "서버에서 오류가 발생했습니다. (" + exception.getMessage() + ")",
                    null
            ));
}
```

`Exception.class`를 지정해서 위에서 따로 처리하지 않은 모든 예외가 이 메소드로 들어오도록 했다. 그중 `CustomException`이면 안에 담아뒀던 에러 코드·메시지를 그대로 꺼내서 `BAD_REQUEST`로 응답하고, 그 외의 예외는 예상하지 못한 상황이므로 `INTERNAL_SERVER_ERROR`와 함께 예외 메시지를 담아 응답하도록 했다.

#### 실행 결과

```json
// MissingServletRequestParameterException
{
    "resultCode": "ERROR-PARAMETER",
    "resultMsg": "필수 파라미터가 누락되었습니다.",
    "result": null
}

// NumberFormatException, MethodArgumentTypeMismatchException
{
    "resultCode": "NOT-ALLOW-TYPE",
    "resultMsg": "허용하지 않는 데이터 타입입니다.",
    "result": null
}

// CustomException
{
    "resultCode": "ERROR-AUTH",
    "resultMsg": "인증 오류입니다.",
    "result": null
}
```

각 상황에 맞게 발생시키거나 던져진 예외가, `GlobalExceptionHandler`를 거쳐 실제 HTTP 상태 코드와 `ResponseDto` 형식을 모두 갖춘 응답으로 일관되게 내려가는 것을 확인할 수 있었다. 다만 지금은 `resultCode`나 메시지를 메소드마다 문자열로 직접 적고 있는데, 앞으로 응답 코드가 늘어나면 한 곳에서 관리할 수 있도록 개선이 필요할 것 같다.
