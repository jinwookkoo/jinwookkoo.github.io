+++
title = "[Spring Boot] 서버 환경별 설정 분리하기"
date = "2025-04-12"
draft = false
tags = ["Spring Boot", "Java"]
+++

Spring Initializr로 Spring Boot 프로젝트를 처음 생성하면 `src/main/resources` 경로 아래에 `application.properties` 파일이 기본으로 만들어져 있다.

```text
src
└── main
    ├── java
    └── resources
        └── application.properties
```

`application.properties`는 Spring Boot 애플리케이션이 구동될 때 필요한 설정 값들을 정의해두는 파일이다. 서버 포트, 데이터베이스 연결 정보, 로깅 레벨 등을 `key=value` 형식으로 관리하며, 별도의 설정이 없으면 애플리케이션 실행 시 이 파일이 자동으로 적용된다.

```properties
server.port=8080
spring.output.ansi.enabled=always
logging.level.root=INFO
```

```yaml
server:
  port: 8080
spring:
  output:
    ansi:
      enabled: always
logging:
  level:
    root: INFO
```

Spring Boot는 위와 같은 두 가지 형식을 지원한다. `application.yml`이 가독성이 더 좋기 때문에 변경하여 적용하기로 하였다.

실무에서는 서버 환경(local, qa, prod)에 따라 데이터베이스 접속 정보, 로깅 레벨 등의 설정 값이 달라지는 경우가 많다. 그래서 공통으로 적용되는 설정은 `application.yml`에 그대로 두고, 환경별로 달라지는 설정만 아래와 같이 별도의 파일로 분리했다.

```text
resources
├── application.yml
├── application-local.yml
├── application-qa.yml
└── application-prod.yml
```

`application.yml`에는 환경에 상관없이 공통으로 적용되는 설정을 두고, `application-local.yml`, `application-qa.yml`, `application-prod.yml`에는 각 환경에서만 달라지는 설정을 정의했다.

Spring Boot는 애플리케이션 실행 시 `application.yml`을 먼저 읽어 공통 설정을 적용한 뒤, 활성화된 프로파일에 해당하는 `application-{profile}.yml`을 이어서 적용한다. 이때 두 파일에 동일한 설정이 있으면 프로파일 파일 쪽 값으로 덮어씌워진다.

각 파일이 어떤 프로파일에서 활성화될지는 `spring.config.activate.on-profile` 값으로 지정했다.

```yaml
# application-local.yml
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: ${LOCAL_DB_HOST}
    username: ${LOCAL_DB_USERNAME}
    password: ${LOCAL_DB_PASSWORD}
```

```yaml
# application-qa.yml
spring:
  config:
    activate:
      on-profile: qa
  datasource:
    url: ${QA_DB_HOST}
    username: ${QA_DB_USERNAME}
    password: ${QA_DB_PASSWORD}
```

```yaml
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${PROD_DB_HOST}
    username: ${PROD_DB_USERNAME}
    password: ${PROD_DB_PASSWORD}
```

`application-{profile}.yml`에 정의한 `${LOCAL_DB_URL}` 같은 환경 변수는 실행 시점에 실제 값으로 주입되어야 한다. 이 실제 값(DB 접속 정보, API 키 등)을 저장소에 커밋하지 않기 위해, 프로젝트 최상단에 `.env` 파일을 만들고 `.gitignore`에 추가해서 관리하고 있다.

```text
project
├── .env
├── .gitignore
├── build.gradle
└── src
```

```text
# .env
LOCAL_DB_HOST=local_db_host
LOCAL_DB_USERNAME=local_user
LOCAL_DB_PASSWORD=local_password
```

`spring.config.import`를 이용하면 `.env` 파일을 properties 형식으로 불러올 수 있다. `application-local.yml`에 아래와 같이 설정했다.

```yaml
# application-local.yml
spring:
  config:
    activate:
      on-profile: local
    import: optional:file:.env[.properties]
```

`optional:` 접두사를 붙였기 때문에 `.env` 파일이 없어도 애플리케이션 구동에 실패하지 않는다.

qa, prod 환경에서는 배포 파이프라인에서 각 환경에 맞는 `.env` 파일을 읽어 서버에 주입하도록 구성해서, 로컬과 동일한 방식으로 설정값이 적용되도록 했다.

이렇게 나눠둔 프로파일은 애플리케이션 실행 시 JVM 옵션으로 활성화했다.

```bash
-Dspring.profiles.active=ENV
```

배포 스크립트에서는 `ENV` 자리에 서버 환경에 맞는 `local`, `qa`, `prod` 값을 넘겨주도록 구성했다. 이렇게 하면 실행 시점에 넘겨준 값에 해당하는 `application-{profile}.yml`이 활성화되어, 공통 설정 위에 해당 환경의 설정이 적용된다.
