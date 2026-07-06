+++
title = "[Spring Boot] Filter로 요청·응답 로깅하기"
date = "2025-04-15"
draft = false
tags = ["Spring Boot", "Java"]
+++

#### 로깅 필터를 만든 이유

클라이언트가 어떤 요청을 보냈고 어떤 응답을 받았는지 확인할 수 있다면 문제가 생겼을 때 더 기민하게 대응할 수 있겠다고 생각했다. 그리고 이전에 만들어둔 `CachedContentHttpServletRequest` 덕분에 JSON 요청이 들어오더라도 Body를 여러 번 읽을 수 있었기 때문에, 요청 Body까지 로그로 남길 수 있었다.

> 이전 글: [Filter로 Request Body 다시 읽기](/posts/spring-boot-filter-interceptor/)

#### ApiFilter를 LogFilter로 확장하기

클라이언트 요청이 들어오는 모든 곳에서 로깅이 이루어져야 했기 때문에, 캐싱보다는 로깅이 이 필터의 주된 목적이 되었다. 그래서 클래스 이름도 `ApiFilter`에서 `LogFilter`로 바꿨다.

기존에는 `Filter` 인터페이스를 직접 구현했지만, 이번엔 `OncePerRequestFilter`를 상속받는 방식으로 바꿨다. `Filter`는 요청이 내부적으로 다른 곳에 다시 전달되거나(forward) 비동기로 처리된 뒤 이어질 때, 같은 요청에 대해 한 번 더 실행될 수 있다. 로깅 필터에 이런 문제가 생기면 클라이언트 입장에서는 요청을 한 번 보냈는데 로그가 두 번 찍히는 것처럼 동작할 수 있다. `OncePerRequestFilter`는 이런 경우에도 요청 하나당 로직이 정확히 한 번만 실행되도록 보장해준다.

```java
public class LogFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        CachedContentHttpServletRequest cachingRequest = new CachedContentHttpServletRequest(request);
        filterChain.doFilter(cachingRequest, response);
    }
}
```

`OncePerRequestFilter`를 상속받으면 `doFilterInternal`의 매개변수가 이미 `HttpServletRequest`, `HttpServletResponse` 타입으로 넘어오기 때문에, 기존처럼 `ServletRequest`를 `HttpServletRequest`로 캐스팅하는 코드도 필요 없어졌다.

#### Response도 캐싱하기

로그에 응답 결과까지 남기려면 Response도 여러 번 읽을 수 있어야 했다. Response 역시 한 번 클라이언트에게 내려보내고 나면 그 내용을 다시 꺼내볼 수 없기 때문에, Request와 마찬가지로 캐싱이 가능한 객체로 감싸야 했다.

다만 Response는 `CachedContentHttpServletRequest`처럼 직접 클래스를 만들 필요 없이, Spring이 제공하는 `ContentCachingResponseWrapper`를 그대로 사용했다.

> Spring은 `ContentCachingRequestWrapper`도 제공한다. 다만 `ContentCachingRequestWrapper`는 컨트롤러 등에서 Request Body를 한 번 읽고 난 뒤에야 캐싱된 내용을 꺼내볼 수 있는 구조라서, 인터셉터 단계에서 Body를 먼저 읽어야 하는 이번 경우에는 맞지 않았다. 그래서 Request는 `HttpServletRequestWrapper`를 직접 상속받아 커스텀 클래스로 만들었고, Response는 별다른 제약이 없어 Spring이 제공하는 `ContentCachingResponseWrapper`를 그대로 사용했다.

```java
public class LogFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        CachedContentHttpServletRequest cachingRequest = new CachedContentHttpServletRequest(request);
        ContentCachingResponseWrapper cachingResponse = new ContentCachingResponseWrapper(response);

        filterChain.doFilter(cachingRequest, cachingResponse);
    }
}
```

#### 로그 남기기

Request 정보는 `filterChain.doFilter`를 호출하기 전에 먼저 로그로 남기고, Response 정보는 `doFilter`가 끝난 뒤에 남기도록 나눴다. 하나의 로그로 묶어서 `doFilter` 이후에만 찍으면, 컨트롤러에서 처리 중 예외가 발생했을 때 로그가 아예 안 남을 수 있기 때문이다. Request를 먼저 남겨두면 이런 경우에도 최소한 어떤 요청이 들어왔는지는 추적할 수 있다.

```java
@Slf4j
public class LogFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain
    ) throws ServletException, IOException {
        CachedContentHttpServletRequest cachingRequest = new CachedContentHttpServletRequest(request);

        log.info("""

                |[REQUEST]
                |>> METHOD: {}
                |>> REQUEST URI: {}
                |>> CLIENT IP: {}
                |>> HEADERS: {}
                |>> REQUEST PARAM: {}
                |>> REQUEST BODY: {}
                """,
                cachingRequest.getMethod(),
                cachingRequest.getRequestURI(),
                getClientIp(cachingRequest), // 클라이언트 IP 조회
                getHeaders(cachingRequest), // 요청 헤더 조회
                getParams(cachingRequest), // 쿼리·폼 파라미터 추출
                getBodyParams(cachingRequest) // Body 파라미터 추출
        );

        ContentCachingResponseWrapper cachingResponse = new ContentCachingResponseWrapper(response);

        filterChain.doFilter(cachingRequest, cachingResponse);

        log.info("""

                |[RESPONSE]
                |>> STATUS: {}
                |>> RESPONSE BODY: {}
                """,
                HttpStatusCode.valueOf(cachingResponse.getStatus()),
                new String(cachingResponse.getContentAsByteArray())
        );

        cachingResponse.copyBodyToResponse();
    }
}
```

`ContentCachingResponseWrapper`를 통해 응답을 쓰면, 그 내용이 실제로 클라이언트에게 전달되는 게 아니라 래퍼가 중간에서 가로채서 자기 안에만 저장해둔다. 즉 컨트롤러 입장에서는 응답을 다 썼다고 생각하지만, 실제로는 클라이언트에게 단 한 바이트도 전달되지 않은 상태다. 그래서 마지막에 `copyBodyToResponse()`를 호출해서, 래퍼가 가로채서 들고 있던 내용을 진짜 Response로 옮겨 써줘야 그제야 클라이언트가 응답을 받을 수 있다. 이 호출을 빼먹으면 로그는 잘 남지만, 클라이언트에게는 빈 응답이 내려간다.

#### 동작 확인

실제로 테스트 요청을 보내보니 아래와 같이 로그가 찍혔다.

```text
|[REQUEST]
|>> METHOD: POST
|>> REQUEST URI: /logging/test
|>> CLIENT IP: 127.0.0.1
|>> HEADERS: {content-length=84, host=localhost:8080, connection=keep-alive, content-type=application/x-www-form-urlencoded, accept-encoding=gzip, deflate, br, user-agent=PostmanRuntime/7.41.0, accept=*/*}
|>> REQUEST PARAM: {message=로깅 테스트입니다.}
|>> REQUEST BODY: {}
```

```text
|[RESPONSE]
|>> STATUS: 200 OK
|>> RESPONSE BODY: 로깅 테스트입니다.
```

Request 로그와 Response 로그가 각각 남아서, 클라이언트가 어떤 요청을 보냈는지와 실제로 어떤 응답이 내려갔는지를 모두 확인할 수 있었다.

이후에는 `application.yml`의 `logging.pattern.console`에 `dd.trace_id`, `dd.span_id`를 추가해서, 데브옵스팀에서 이 로그를 Datadog으로 파싱해 연동할 수 있도록 했다.
