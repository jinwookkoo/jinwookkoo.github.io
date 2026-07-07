+++
title = "[Spring Boot] MapStruct로 객체 매핑하기"
date = "2025-05-16"
draft = false
tags = ["Spring Boot", "Java"]
+++

#### MapStruct를 적용하게 된 이유

클래스 간의 책임이 명확하게 분리되면 객체에서 다른 객체로 매핑하는 코드가 많아진다. 그래서 코드의 복잡성을 줄이고 객체 간의 매핑을 편리하게 진행할 수 있도록 MapStruct를 적용하는 방법을 정리해본다.

#### MapStruct

> MapStruct is a code generator that greatly simplifies the implementation of mappings between Java bean types based on a convention over configuration approach.
>
> 출처: [mapstruct.org](https://mapstruct.org/)

MapStruct는 공식 사이트에 나와있는 것처럼 Java Bean 유형 간의 매핑 구현을 단순화하는 코드 생성기다. 컴파일 시점에 코드를 생성해서(런타임 안정성 보장) 객체 간의 매핑 작업을 자동화하고, 다른 매핑 라이브러리보다 빠르다는 장점도 있다.

#### dependency 추가

```groovy
dependencies {
    annotationProcessor 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
}
```

Gradle 기준으로 위 의존성을 추가해주면 되는데, `org.projectlombok:lombok-mapstruct-binding`을 함께 추가해야 Lombok과의 충돌을 피할 수 있다. 이 라이브러리를 사용하지 않는다면, 대신 어노테이션 프로세서 의존성 순서를 Lombok → MapStruct 순으로 맞춰줘야 한다.

#### 수동 매핑

지난 글에서 Controller DTO(`DocumentsRequest`)와 Service DTO(`DocumentsRequestModel`)를 나눴는데, 그 사이를 연결하려면 `DocumentController`에서 이렇게 수동으로 매핑하는 코드가 필요하다.

> 이전 글: [Controller와 Service DTO 분리하기](/posts/spring-boot-controller-service-dto/)

```java
@GetMapping
public ResponseEntity<ApiResponse<List<DocumentResponse>>> getDocuments(
        @ModelAttribute DocumentsRequest documentsRequest
) {
    DocumentsRequestModel documentsRequestModel = DocumentsRequestModel.builder()
            .userId(documentsRequest.userId())
            .tagIds(documentsRequest.tagIds())
            .page(documentsRequest.page())
            .pageSize(documentsRequest.pageSize())
            .build();

    List<DocumentResponse> result = documentInboundPort.getDocuments(documentsRequestModel);
    return ResponseEntity.ok(new ApiResponse<>(200, "SUCCESS", result));
}
```

MapStruct를 적용하면 이 코드를 다음과 같이 대체할 수 있다.

#### Mapper 인터페이스 생성하기

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface DocumentMapper {
    DocumentsRequestModel toModelFromRequest(DocumentsRequest documentsRequest);
}
```

| `@Mapper` 옵션 | 설명 |
|------|------|
| `componentModel` | 생성되는 Mapper를 어떤 방식으로 등록할지 지정한다. `componentModel = "spring"`으로 지정하면 스프링 빈으로 등록된다. |
| `unmappedTargetPolicy` | 매핑되지 않는 필드에 대한 정책을 지정한다. `ERROR`(매핑 코드 생성 실패), `WARN`(빌드 시 경고), `IGNORE`(무시) 중 선택할 수 있다. |

`@Mapper`를 붙이면 MapStruct가 이 인터페이스의 구현체를 컴파일 시점에 자동으로 생성해준다. 필요하다면 `@Mapping`을 함께 사용해서 필드별 매핑 규칙을 직접 지정할 수도 있다.

```java
@Mapping(source = "no", target = "id")
DocumentResponse toResponseFromDomain(Document document);
```

`source`는 변환할 필드를, `target`은 저장할 필드를 나타낸다. `Document`의 `no` 필드를 `DocumentResponse`의 `id` 필드로 매핑하는 것처럼, 필드 이름이 서로 다를 때 이렇게 명시적으로 지정해줄 수 있다.

```java
@Mapping(target = "useYn", ignore = true)
DocumentResponse toResponseFromDomain(Document document);
```

`ignore`를 `true`로 지정하면 해당 필드는 매핑하지 않는다.

```java
@Mapping(target = "useYn", expression = "java(document.isEnabled() ? \"Y\" : \"N\")")
DocumentResponse toResponseFromDomain(Document document);
```

`expression`을 사용하면 단순 필드 매핑이 아니라, 원하는 Java 표현식으로 값을 직접 정의할 수 있다.

```java
// @Mappings로 묶어서 나열
@Mappings({
        @Mapping(source = "no", target = "id"),
        @Mapping(target = "useYn", expression = "java(document.isEnabled() ? \"Y\" : \"N\")")
})
DocumentResponse toResponseFromDomain(Document document);

// @Mapping을 반복해서 나열
@Mapping(source = "no", target = "id")
@Mapping(target = "useYn", expression = "java(document.isEnabled() ? \"Y\" : \"N\")")
DocumentResponse toResponseFromDomain(Document document);
```

한 메서드에 여러 개의 `@Mapping`을 적용해야 한다면 `@Mappings`로 묶어서 배열 형태로 나열할 수 있다. 그런데 `@Mapping`은 `@Repeatable`로 선언되어 있어서, `@Mappings`로 감싸지 않고 여러 번 나열해도 완전히 동일하게 동작한다.

#### 자동으로 생성되는 구현체

```java
@Component
public class DocumentMapperImpl implements DocumentMapper {
    @Override
    public DocumentsRequestModel toModelFromRequest(DocumentsRequest documentsRequest) {
        if (documentsRequest == null) {
            return null;
        }

        DocumentsRequestModel.DocumentsRequestModelBuilder documentsRequestModel = DocumentsRequestModel.builder();

        documentsRequestModel.userId(documentsRequest.userId());
        documentsRequestModel.tagIds(documentsRequest.tagIds());
        if (documentsRequest.page() != null) {
            documentsRequestModel.page(documentsRequest.page());
        }
        if (documentsRequest.pageSize() != null) {
            documentsRequestModel.pageSize(documentsRequest.pageSize());
        }

        return documentsRequestModel.build();
    }
}
```

`DocumentMapper`를 컴파일하면 위와 같은 구현체가 자동으로 생성되며, `build/generated/sources/annotationProcessor` 아래 `DocumentMapper` 인터페이스가 위치한 패키지에서 직접 확인할 수 있다. 앞서 "수동 매핑"에서 직접 작성했던 코드와 거의 같은 모습인데, MapStruct가 바로 이 반복 작업을 대신 생성해주는 것이다.

#### DocumentMapper 적용하기

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/documents")
public class DocumentController {
    private final DocumentMapper documentMapper;
    private final DocumentInboundPort documentInboundPort;

    @GetMapping
    public ResponseEntity<ApiResponse<List<DocumentResponse>>> getDocuments(
            @ModelAttribute DocumentsRequest documentsRequest
    ) {
        DocumentsRequestModel documentsRequestModel = documentMapper.toModelFromRequest(documentsRequest);

        List<DocumentResponse> result = documentInboundPort.getDocuments(documentsRequestModel);
        return ResponseEntity.ok(new ApiResponse<>(200, "SUCCESS", result));
    }
}
```

`DocumentMapper`를 주입받아서, 앞서 직접 작성했던 변환 코드를 `documentMapper.toModelFromRequest(documentsRequest)` 한 줄로 대체했다.

#### 주의점

MapStruct는 컴파일 시점에 구현체를 자동으로 생성해주기 때문에, 실제로 실행해보기 전까지는 매핑이 제대로 되었는지 확인하기 어렵다. 그래서 구현체가 생성된 이후에는 `build/generated` 아래의 실제 구현체 코드를 직접 열어서, 필드가 의도한 대로 매핑되었는지 확인해보는 것이 좋다.

그리고 필드 이름에 따라 자동 매핑이 예상과 다르게 동작할 수도 있다. 예를 들어 `eMail`이라는 필드가 있다고 해보자. 이 필드의 getter는 관례상 `getEMail()`이 된다.

자바의 `java.beans.Introspector`는 getter 이름에서 프로퍼티 이름을 추론할 때, 기본적으로 `get`을 뗀 나머지 이름의 첫 글자만 소문자로 바꾼다. 그런데 그 나머지 이름의 첫 두 글자가 모두 대문자인 경우에는 예외적으로 첫 글자를 소문자로 바꾸지 않고 그대로 둔다. 예를 들어 `getURL()`은 `get`을 떼면 `URL`이 남는데, 이걸 규칙대로 첫 글자만 소문자로 바꾸면 `uRL`이라는 어색한 이름이 되어버린다. 그래서 첫 두 글자(`U`, `R`)가 모두 대문자인 경우에는 그대로 두어 프로퍼티 이름이 `URL`이 되도록 예외를 둔 것이다.

문제는 이 예외 규칙이 `eMail`처럼 의도치 않은 이름에도 똑같이 적용된다는 점이다. `getEMail()`에서 `get`을 떼면 `EMail`이 남는데, 이 역시 첫 두 글자(`E`, `M`)가 모두 대문자이기 때문에 같은 예외 규칙에 걸려서 그대로 `EMail`로 인식된다. 결국 실제 필드 이름은 `eMail`이지만, Introspector가 인식하는 프로퍼티 이름은 `eMail`이 아니라 `EMail`이 되어버리는 것이다. 이 문제는 MapStruct뿐만 아니라, 자바 빈 규약을 따르는 Jackson이나 Hibernate 같은 다른 라이브러리에서도 똑같이 발생할 수 있다.

MapStruct도 이 규칙을 그대로 따르기 때문에, `eMail` 필드는 이름이 자동으로 매핑되지 않을 수 있다. 이럴 때는 아래처럼 실제 필드 이름이 아니라 Introspector가 인식하는 이름을 명시적으로 지정해줘야 한다.

```java
@Mapping(source = "EMail", target = "EMail")
DocumentResponse toResponseFromDomain(Document document);
```