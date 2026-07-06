+++
title = "[Spring Boot] RestClient로 외부 API 호출하기"
date = "2025-04-16"
draft = false
tags = ["Spring Boot", "Java"]
+++

#### RestClient를 쓰게 된 이유

외부 API를 호출해야 해서 HTTP Client 요청 방법을 찾고 있었는데, 마침 Spring Boot 3.2 이상을 쓰고 있었고 이 버전부터 새로 추가된 `RestClient`가 있다는 걸 알게 됐다. `WebClient`는 비동기 방식인데, 지금 선택한 서버가 Spring MVC 기반, 즉 동기 방식을 사용하고 있어서 굳이 쓸 이유가 없었다. 게다가 별도 의존성(webflux)까지 추가해야 했다. `RestTemplate`는 더 이상 기능이 추가되지 않고 유지보수도 되지 않는 상태였기 때문에, 결국 `RestClient`를 써보기로 했다.

> **RestClient Document**
> [https://docs.spring.io/spring-framework/reference/integration/rest-clients.html](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html)

#### RestClientConfig 만들기

API 호출은 프로젝트 전반에서 사용 범위가 넓기 때문에, `RestClient`를 빈으로 등록해서 어디서든 주입받아 쓸 수 있도록 `RestClientConfig` 클래스를 구현했다.

```java
@Slf4j
@Configuration
public class RestClientConfig {
    @Bean
    public RestClient restClient() {
        return RestClient.builder()
                .requestInterceptor(clientHttpRequestInterceptor())
                .build();
    }

    private ClientHttpRequestInterceptor clientHttpRequestInterceptor() {
        return (request, body, execution) -> {
            log.info("""


                    |[RESTCLIENT REQUEST]
                    |>> METHOD: {}
                    |>> REQUEST URI: {}
                    |>> HEADERS: {}
                    |>> REQUEST BODY: {}
                    """,
                    request.getMethod(),
                    request.getURI(),
                    request.getHeaders(),
                    new String(body)
            );

            ClientHttpResponse response = execution.execute(request, body);

            log.info("""


                    |[RESTCLIENT RESPONSE]
                    |>> STATUS: {}
                    |>> HEADERS: {}
                    |>> RESPONSE BODY: {}
                    """,
                    response.getStatusCode(),
                    response.getHeaders(),
                    new String(response.getBody().readAllBytes())
            );

            return response;
        };
    }
}
```

`RestClient`의 `builder` 메소드로 빈을 생성했고, `requestInterceptor`에 `ClientHttpRequestInterceptor`를 등록해서 API 요청·응답을 로그로 남기도록 했다. `execution.execute(request, body)`를 호출하기 전에는 요청 정보(메소드, URI, 헤더, Body)를 로그로 남기고, 호출한 뒤에는 그 결과로 받은 응답 정보(상태 코드, 헤더, Body)를 로그로 남긴다.

#### RestClientManager 만들기

`RestClientConfig`에서 등록해둔 `RestClient` 빈을 주입받아 실제로 API를 호출하는 `RestClientManager` 클래스를 만들었다.

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class RestClientManager {
    private final RestClient restClient;
}
```

#### GET 요청 구현하기

```java
public <T> T get(
    String uri,
    Map<String, Object> requestParams,
    ParameterizedTypeReference<T> responseType
) {
    try {
        UriComponentsBuilder uriComponentsBuilder = UriComponentsBuilder.fromHttpUrl(uri);

        for (String key : requestParams.keySet()) {
            uriComponentsBuilder.queryParam(key, requestParams.get(key));
        }

        return restClient.get()
                .uri(new URI(uriComponentsBuilder.toUriString()))
                .retrieve()
                .body(responseType);
    } catch (Exception e) {
        log.error("GET 요청 실패", e);
        return null;
    }
}
```

GET 요청은 URL 뒤에 쿼리 파라미터를 붙여야 하기 때문에 `UriComponentsBuilder` 클래스를 사용해서 추가했다. `restClient`의 `uri` 메소드는 `String` 타입으로 전달하면 인코딩을 한 번 더 진행하고, `URI` 타입으로 전달하면 인코딩을 진행하지 않는다. `UriComponentsBuilder`의 `toUriString` 메소드 내부를 보면 자동으로 인코딩을 진행(`build().encode().toUriString()`)하기 때문에, `String` 타입 그대로 넘기면 이중 인코딩이 발생한다. 그래서 `URI` 타입으로 변환한 뒤에 전달했다. 그리고 제네릭을 활용해서 반환되는 객체의 타입을 매개변수로 지정할 수 있도록 했는데, `Class<T>`가 아니라 `ParameterizedTypeReference<T>`를 사용한 이유는 구체적인 타입 정보를 명시하기 위해서였다.

예외가 발생했을 때 그대로 던지지 않고 `null`을 반환하도록 한 이유는, 외부 API 호출이 실패했다고 해서 전체 로직까지 실패해야 하는 건 아니었기 때문이다. 이후에 나올 POST, PUT, DELETE도 같은 이유로 동일한 방식을 사용했다.

#### POST 요청 구현하기

```java
public <T> T post(
    String uri,
    HttpHeaders headers,
    Map<String, Object> requestParams,
    ParameterizedTypeReference<T> responseType
) {
    try {
        return restClient.post()
                .uri(new URI(uri))
                .headers(httpHeaders -> httpHeaders.addAll(headers))
                .body(MediaType.APPLICATION_FORM_URLENCODED.equals(headers.getContentType())
                        ? new LinkedMultiValueMap<String, Object>() {{ setAll(requestParams); }}
                        : requestParams)
                .retrieve()
                .body(responseType);
    } catch (Exception e) {
        log.error("POST 요청 실패", e);
        return null;
    }
}
```

`headers.getContentType()`으로 꺼낸 값을 가지고, Content-Type이 `application/x-www-form-urlencoded`인 경우에는 `Map` 객체를 `LinkedMultiValueMap` 객체로 변환해서 전달하도록 분기 처리했다.

#### POST 요청에서 발생한 411 에러

그런데 POST 요청을 테스트하는 중에 아래와 같은 오류가 발생했다.

```text
statusCode: 411 LENGTH_REQUIRED
detailMessage: Could not extract response: no suitable HttpMessageConverter found for response type [java.util.Map<java.lang.String, java.lang.Object>] and content type [text/html]
```

반환된 Content-Type이 `text/html`이라서 지정한 `Map` 타입으로 변환할 수 있는 `HttpMessageConverter`가 없다는 내용이었는데, 반환된 HTTP 상태 코드가 411이었다.

> 411: 클라이언트 요청에 Content-Length 헤더가 포함되어야 함을 의미한다.

POST 요청 시에는 요청 본문의 길이를 명시해야 하는데, 디버깅해보니 Content-Length 값을 보내고 있지 않았다. Spring Framework GitHub Wiki에서 관련 내용을 찾을 수 있었다.

> To reduce memory usage in `RestClient` and `RestTemplate`, most `ClientHttpRequestFactory` implementations no longer buffer request bodies before sending them to the server. As a result, for certain content types such as JSON, the contents size is no longer known, and a `Content-Length` header is no longer set. If you would like to buffer request bodies like before, simply wrap the `ClientHttpRequestFactory` you are using in a `BufferingClientHttpRequestFactory`.
>
> 출처: [Upgrading to Spring Framework 6.x](https://github.com/spring-projects/spring-framework/wiki/Upgrading-to-Spring-Framework-6.x#web-applications)

메모리 사용량을 줄이기 위해 더 이상 요청 본문을 버퍼링하지 않아서 Content-Length가 설정되지 않는다는 내용이었다. 이전처럼 요청 본문을 버퍼링하려면 사용 중인 `ClientHttpRequestFactory`를 `BufferingClientHttpRequestFactory`로 감싸서 쓰면 된다는 것이었다. `requestFactory`를 따로 설정하지 않으면 기본적으로 `JdkClientHttpRequestFactory`를 사용한다는 것도 추가로 알게 되어, `RestClientConfig`에 아래처럼 감싸서 설정을 추가했다.

```java
@Bean
public RestClient restClient() {
    return RestClient.builder()
            .requestFactory(new BufferingClientHttpRequestFactory(new JdkClientHttpRequestFactory()))
            .requestInterceptor(clientHttpRequestInterceptor())
            .build();
}
```

디버깅을 통해 Content-Length가 자동으로 계산되어 요청되는 것을 확인했다.

`BufferingClientHttpRequestFactory`는 요청뿐 아니라 응답도 메모리에 버퍼링한다. 덕분에 로깅 인터셉터에서 `response.getBody().readAllBytes()`로 응답 Body를 미리 읽어도, 그 이후 `retrieve().body(responseType)`가 다시 응답 Body를 읽어서 변환하는 데 문제가 없다.

#### 파일 전송(Multipart) POST 요청 구현하기

POST 요청에서 주로 사용하는 Content-Type에는 `application/x-www-form-urlencoded`, `application/json` 말고도 파일을 전송할 때 쓰는 `multipart/form-data`가 있다. 그래서 파일을 전송할 수 있는 `postByMultipart` 메소드를 추가로 구현했다.

```java
public <T> T postByMultipart(
    String uri,
    Map<String, Object> requestParams,
    Map<String, List<MultipartFile>> requestFileParams,
    ParameterizedTypeReference<T> responseType
) {
    try {
        MultipartBodyBuilder multipartBodyBuilder = new MultipartBodyBuilder();

        for (String key : requestParams.keySet()) {
            multipartBodyBuilder.part(key, requestParams.get(key));
        }

        for (String key : requestFileParams.keySet()) {
            for (MultipartFile multipartFile : requestFileParams.get(key)) {
                multipartBodyBuilder.part(key, multipartFile.getResource());
            }
        }

        return restClient.post()
                .uri(new URI(uri))
                .header(HttpHeaders.CONTENT_TYPE, MediaType.MULTIPART_FORM_DATA_VALUE)
                .body(multipartBodyBuilder.build())
                .retrieve()
                .body(responseType);
    } catch (Exception e) {
        log.error("POST(Multipart) 요청 실패", e);
        return null;
    }
}
```

Spring Boot에서는 `MultipartFile` 클래스로 업로드된 파일을 다룰 수 있고, `MultipartBodyBuilder` 클래스로 `multipart/form-data` 요청 본문을 만들 수 있다. 일반 파라미터는 `part(key, value)`로, 파일은 `multipartFile.getResource()`를 통해 같은 방식으로 추가했다.

#### PUT 요청 구현하기

PUT 요청도 POST와 마찬가지로 Content-Type이 `application/x-www-form-urlencoded`인 경우가 있을 수 있어서, 동일하게 분기 처리했다.

```java
public <T> T put(
    String uri,
    HttpHeaders headers,
    Map<String, Object> requestParams,
    ParameterizedTypeReference<T> responseType
) {
    try {
        return restClient.put()
                .uri(new URI(uri))
                .headers(httpHeaders -> httpHeaders.addAll(headers))
                .body(MediaType.APPLICATION_FORM_URLENCODED.equals(headers.getContentType())
                        ? new LinkedMultiValueMap<String, Object>() {{ setAll(requestParams); }}
                        : requestParams)
                .retrieve()
                .body(responseType);
    } catch (Exception e) {
        log.error("PUT 요청 실패", e);
        return null;
    }
}
```

#### DELETE 요청 구현하기

`RestClient`는 `get`, `post`, `put`, `delete` 메소드를 편의상 만들어두었지만, `delete` 메소드로 만든 요청에서는 `body` 메소드를 호출할 수 없었다. DELETE 요청에는 보통 본문을 포함하지 않지만, API 서버 구현 방식에 따라 본문이 필요한 경우도 있어서 `method` 메소드를 사용해 구현했다.

```java
public <T> T delete(
    String uri,
    HttpHeaders headers,
    Map<String, Object> requestParams,
    ParameterizedTypeReference<T> responseType
) {
    try {
        return restClient.method(HttpMethod.DELETE)
                .uri(new URI(uri))
                .headers(httpHeaders -> httpHeaders.addAll(headers))
                .body(MediaType.APPLICATION_FORM_URLENCODED.equals(headers.getContentType())
                        ? new LinkedMultiValueMap<String, Object>() {{ setAll(requestParams); }}
                        : requestParams)
                .retrieve()
                .body(responseType);
    } catch (Exception e) {
        log.error("DELETE 요청 실패", e);
        return null;
    }
}
```

#### 동작 확인

`RestClientManager`가 정상적으로 동작하는지 확인하기 위해, 인증 없이 바로 호출할 수 있는 테스트용 API인 [JSONPlaceholder](https://jsonplaceholder.typicode.com)로 GET 요청을 보내는 테스트 코드를 작성했다.

```java
@SpringBootTest
@ActiveProfiles("local")
class RestClientManagerTest {
    @Autowired
    private RestClientManager restClientManager;

    @Test
    void getFromJsonPlaceholder() {
        Map<String, Object> result = restClientManager.get(
                "https://jsonplaceholder.typicode.com/posts/1",
                new HashMap<>(),
                new ParameterizedTypeReference<>() {}
        );

        assertThat(result).containsKey("title");
    }
}
```
