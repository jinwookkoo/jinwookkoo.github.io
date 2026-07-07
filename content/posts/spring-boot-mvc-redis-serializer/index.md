+++
title = "[Spring Boot] Redis 직렬화(Serializer)"
date = "2025-04-17"
draft = false
tags = ["Spring Boot", "Java", "Redis"]
+++

#### Redis 캐싱을 쓰게 된 이유

DB에서 데이터를 조회하거나 API를 호출하면서 빈번하게 호출되지 않아도 되는 정보에 대해서는 캐싱(Caching)이 필요했다. 그래서 Redis를 설정해서 사용하게 되었고, 이번 글에서는 그 과정과 함께 데이터를 저장할 때 선택한 직렬화(Serializer) 방식에 대해서도 정리해본다.

#### dependency 추가

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
}
```

gradle 설정에 Redis 관련 라이브러리를 추가해주었다. 위 라이브러리를 통해 Redis DB와 상호작용할 수 있는 기능이 제공된다.

#### 설정 파일

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
```

Redis에 대한 설정 클래스를 구현하기 전에, 환경 설정 파일의 `spring.data.redis`에서 `host`와 `port`를 입력해준다.

#### RedisConfig 클래스

```java
@Configuration
public class RedisConfig {
    @Value("${spring.data.redis.host}")
    private String host;

    @Value("${spring.data.redis.port}")
    private int port;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(host, port);
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        return redisTemplate;
    }
}
```

환경 설정 파일에서 입력한 Redis 정보를 사용하는 `RedisConfig` 클래스를 구현했다. `@Configuration`을 사용하여 빈(Bean)을 등록하고, `@Value`를 사용하여 환경 설정 파일의 Redis 정보를 각 변수에 주입했다. 이 변수는 `LettuceConnectionFactory`를 생성할 때 사용했는데, Redis Connection을 맺는 방식에는 `Lettuce`와 `Jedis` 두 가지가 일반적으로 사용된다. `Lettuce`는 Spring Boot에서 기본으로 채택하고 있는 클라이언트이다. 지금은 Spring MVC(동기) 기반이라 직접 체감하는 이점은 아니지만, `Jedis`와 달리 `ReactiveRedisConnectionFactory`를 통한 리액티브 방식도 지원하기 때문에 나중에 서버가 WebFlux 같은 리액티브 스택으로 바뀌더라도 Redis 클라이언트를 바꾸지 않고 그대로 갈 수 있다는 점까지 고려해서 `Lettuce`를 선택했다.

`Lettuce`는 Netty 기반이라 하나의 커넥션 위에서 여러 요청을 동시에 처리할 수 있고, `LettuceConnectionFactory`는 기본적으로 이 커넥션 하나를 공유(`shareNativeConnection = true`)해서 쓴다. 덕분에 요청마다 커넥션을 새로 맺거나 풀에서 빌려올 필요가 없지만, `BLPOP`처럼 응답이 올 때까지 대기하는 Blocking 커맨드나 `MULTI`/`WATCH`를 사용하는 트랜잭션, Pub/Sub 구독처럼 커넥션 하나를 독점해야 하는 작업에는 공유 커넥션을 그대로 쓰면 다른 요청과 뒤섞일 수 있다. 이런 경우에는 `shareNativeConnection`을 `false`로 두고 전용 커넥션을 써야 한다. 지금은 단순 캐싱 용도로만 쓰고 있어서 별도 설정 없이 기본값인 공유 커넥션을 그대로 사용했다.

그리고 `RedisTemplate`을 통해 Redis DB와 상호작용할 수 있는 설정을 입력했는데, `setKeySerializer`와 `setValueSerializer` 메소드로 Redis DB에 저장되는 직렬화 형식을 지정할 수 있었다. 직렬화 방식에는 아래와 같은 선택지가 있었다.

| 방식 | 설명 |
|------|------|
| `Jackson2JsonRedisSerializer` | 클래스 타입을 지정해주어야 한다. |
| `GenericJackson2JsonRedisSerializer` | 클래스 타입을 지정할 필요 없이 모든 객체에 대한 직렬화가 가능하다. 다만 객체의 클래스 타입 정보까지 함께 저장한다. |
| `StringRedisSerializer` | 클래스 타입을 지정할 필요가 없다. 대신 저장할 때 직접 데이터를 Encoding·Decoding 해야 한다. |

예를 들어 아래와 같이 `name`과 `age`를 가진 객체를 저장한다고 하면,

```java
@Getter
@AllArgsConstructor
public class MemberDto {
    private String name;
    private int age;
}
```

`name = "User"`, `age = 20`일 때 Redis에는 실제로 다음과 같이 저장된다.

```text
# Jackson2JsonRedisSerializer, StringRedisSerializer(직접 변환)
{"name":"User","age":20}

# GenericJackson2JsonRedisSerializer
{"@class":"com.example.dto.MemberDto","name":"User","age":20}
```

`Jackson2JsonRedisSerializer`는 `RedisSerializer<T>`를 구현하고 있어서 Object ↔ JSON ↔ byte[] 변환을 대신해주지만, 생성 시점에 다룰 클래스 타입을 미리 지정해야 한다(`new Jackson2JsonRedisSerializer<>(MemberDto.class)`). 저장되는 JSON 자체에는 타입 정보가 없기 때문에, 역직렬화할 때는 이렇게 지정해둔 타입을 그대로 사용하는 구조다. 그래서 하나의 Serializer(= 하나의 `RedisTemplate`)는 사실상 그 타입 전용이 되고, 여러 타입을 저장하려면 타입마다 별도의 Serializer를 만들어야 한다는 제약이 있었다.

`GenericJackson2JsonRedisSerializer`는 이런 제약 없이 클래스 타입을 지정하지 않아도 모든 객체를 직렬화할 수 있는데, 역직렬화할 때 어떤 타입으로 변환할지 알아야 하기 때문에 그 대신 `@class` 필드에 클래스 정보까지 함께 저장한다. 다만 그만큼 저장 용량이 늘어나고, 그 데이터를 Java가 아닌 다른 언어나 서비스에서 그대로 활용하기는 어렵다는 단점이 있었다.

`StringRedisSerializer`는 `RedisSerializer<String>`이라 클래스 타입을 지정할 필요가 없고, String과 byte[] 사이의 변환만 처리한다. 대신 JSON이나 객체에 대해서는 전혀 알지 못하기 때문에, 객체를 저장·조회하려면 JSON 변환을 애플리케이션 코드에서 직접 처리해줘야 한다. 하지만 다른 정보를 포함하지 않아 용량을 최소화할 수 있고 다른 애플리케이션에서도 자유롭게 사용할 수 있다는 장점이 있어서 최종적으로 이 방식을 선택했다.

#### RedisManager 클래스

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class RedisManager {
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    public <T> T get(String key, Class<T> classType) {
        String value = redisTemplate.opsForValue().get(key);
        if (!StringUtils.hasText(value)) return null;

        try {
            if (classType == String.class) {
                return classType.cast(value);
            }

            return objectMapper.readValue(value, classType);
        } catch (Exception e) {
            log.error("""


                    |[REDIS]
                    |>> FAILED TO GET CACHE
                    |>> REASON: {}
                    """,
                    e.getMessage()
            );
            return null;
        }
    }

    public <T> boolean set(String key, T value, long ttl) {
        if (ObjectUtils.isEmpty(value)) return false;

        try {
            String serializedValue = value instanceof String ? (String) value : objectMapper.writeValueAsString(value);
            redisTemplate.opsForValue().set(key, serializedValue, ttl, TimeUnit.SECONDS);
            return true;
        } catch (Exception e) {
            log.error("""


                    |[REDIS]
                    |>> FAILED TO SET CACHE
                    |>> REASON: {}
                    """,
                    e.getMessage()
            );
            return false;
        }
    }
}
```

`StringRedisSerializer`를 쓰면 호출하는 쪽에서 매번 JSON 변환을 신경 써야 하기 때문에, 이를 감싸서 공통으로 편하게 쓸 수 있는 `RedisManager` 클래스를 구현했다. `get`과 `set` 모두 제네릭을 활용해서 다양한 타입을 다룰 수 있게 했고, `ObjectMapper`는 Spring Boot가 기본으로 제공하는 빈을 그대로 주입받아 사용했다. 내부에서 JSON 변환·예외 처리를 전담하기 때문에, 호출하는 쪽에서는 직렬화 방식을 신경 쓰지 않고 캐시를 사용할 수 있다.

한 가지 예외는 `classType`이 `String`인 경우다. `set`에서 값이 `String`이면 `objectMapper.writeValueAsString`을 거치지 않고 원본 그대로 저장하기 때문에, `get`에서도 `String`일 때만큼은 `objectMapper.readValue`를 타지 않고 바로 캐스팅해서 반환하도록 분기했다.

#### 동작 확인

`RedisManager`가 정상적으로 동작하는지 확인하기 위해, 값을 저장하고 그대로 조회되는지 확인하는 간단한 테스트 코드를 작성했다.

```java
@SpringBootTest
@ActiveProfiles("local")
class RedisManagerTest {
    @Autowired
    private RedisManager redisManager;

    @Test
    void getReturnsSavedValue() {
        MemberDto member = new MemberDto("User", 20);

        redisManager.set("test:member", member, 60);
        MemberDto result = redisManager.get("test:member", MemberDto.class);

        assertThat(result.getName()).isEqualTo("User");
        assertThat(result.getAge()).isEqualTo(20);
    }
}
```

`set`으로 60초 TTL을 두고 객체를 저장한 뒤, `get`으로 같은 키를 조회해서 저장했던 값과 동일하게 반환되는지 확인했다. 테스트를 통해 저장한 값이 그대로 캐시되어 조회되는 것을 확인할 수 있었다.
