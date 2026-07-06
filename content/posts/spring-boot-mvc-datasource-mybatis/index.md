+++
title = "[Spring Boot] DataSource 설정과 MyBatis 연동하기"
date = "2025-04-13"
draft = false
tags = ["Spring Boot", "Java", "MySQL"]
+++

#### Connection Pool이 필요한 이유

Java에서는 JDBC(**Java Database Connectivity**)라는 API를 통해 데이터베이스에 접근할 수 있지만, 매번 Connection을 새로 맺는 데 비용이 많이 들기 때문에 Connection Pool을 사용하는 것이 일반적이다. 기존에 사용하던 Codeigniter(PHP)에서는 Connection Pool을 사용할 수 없었는데, Spring Boot로 컨버팅하면서 이 부분을 어떻게 설정했는지 기록해보려고 한다.

Spring Boot에서는 `DataSource`라는 표준 인터페이스를 통해 DB Connection Pool을 관리하며, Spring Boot 2.0부터는 기본 설정으로 [HikariCP](https://github.com/brettwooldridge/HikariCP-benchmark)를 사용할 수 있다.

#### Dependency 추가

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    runtimeOnly 'com.mysql:mysql-connector-j'
    implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
}
```

gradle 설정에 JDBC, MySQL, MyBatis 관련 라이브러리를 추가했다. MySQL 관련 라이브러리는 애플리케이션이 실행될 때만 필요하기 때문에 `runtimeOnly`로 지정했다.

> `mybatis-spring-boot-starter`는 3.0.x부터 Java 17 이상과 Spring Boot 3.x를 기준으로 하고, 2.3.x까지는 Java 8 이상과 Spring Boot 2.7 기준이다. 프로젝트의 Spring Boot 버전에 맞춰 starter 버전을 선택해야 한다.

#### DataSource 설정

```yaml
spring:
  datasource:
    hikari:
      db:
        master:
          jdbc-url: jdbc:mysql://${MASTER_DB_HOST}
          username: ${MASTER_DB_USERNAME}
          password: ${MASTER_DB_PASSWORD}
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave:
          jdbc-url: jdbc:mysql://${SLAVE_DB_HOST}
          username: ${SLAVE_DB_USERNAME}
          password: ${SLAVE_DB_PASSWORD}
          driver-class-name: com.mysql.cj.jdbc.Driver
```

`DataSource` 객체를 만들기 전에 `application.yml`의 `spring.datasource`에 DB 설정 값을 입력했다. AWS RDS에서 해당 DB가 리더·라이터 인스턴스로 나뉘어 있어서, 라이터 인스턴스를 master, 리더 인스턴스를 slave로 구분했다. `spring.datasource` 다음에 `hikari`를 넣어야 HikariCP 설정 옵션을 사용할 수 있었다.

주로 사용한 HikariCP 설정 옵션은 다음과 같다.

- `connection-timeout`: 커넥션 타임아웃 설정 (ms), 기본값 30000
- `maximum-pool-size`: 커넥션 풀의 최대 크기, 기본값 10
- `minimum-idle`: 유휴 커넥션을 유지할 최소 커넥션 수, 기본값 5
- `idle-timeout`: 유휴 커넥션의 최대 유지 시간 (ms), 기본값 600000
- `pool-name`: 커넥션 풀의 이름
- `validation-timeout`: 커넥션 유효성 검사 타임아웃 (ms), 기본값 5000
- `auto-commit`: 자동 커밋 모드 여부, 기본값 true
- `max-lifetime`: 커넥션의 최대 수명 시간 (ms), 기본값 1800000

#### DataSourceConfig 구현

`DataSource`는 DB Connection Pool을 추상화한 표준 인터페이스로, 실제로 커넥션을 맺고 관리하는 역할을 한다. Spring Boot는 `DataSource` 빈이 하나만 있을 때는 자동으로 등록해주지만, 지금처럼 master/slave 두 개의 커넥션 풀을 따로 관리해야 하는 경우에는 자동 설정만으로는 처리할 수 없어서 `DataSource` 빈을 직접 등록해줘야 했다.

```java
@Configuration
public class DataSourceConfig {
    @Bean(name = "dbMasterDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.hikari.db.master")
    public DataSource dbMasterDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean(name = "dbSlaveDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.hikari.db.slave")
    public DataSource dbSlaveDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }
}
```

`@ConfigurationProperties`를 사용해서 `application.yml`에 입력해둔 DB 정보를 매핑했다.

#### MybatisConfig 구현

```java
@Configuration
public class MybatisConfig {
    @Bean(name = "dbMasterSqlSessionTemplate")
    public SqlSessionTemplate dbMasterSqlSessionTemplate(
            @Qualifier("dbMasterDataSource") DataSource dbMasterDataSource
    ) {
        return new SqlSessionTemplate(sqlSessionFactory(dbMasterDataSource));
    }

    @Bean(name = "dbSlaveSqlSessionTemplate")
    public SqlSessionTemplate dbSlaveSqlSessionTemplate(
            @Qualifier("dbSlaveDataSource") DataSource dbSlaveDataSource
    ) {
        return new SqlSessionTemplate(sqlSessionFactory(dbSlaveDataSource));
    }

    private SqlSessionFactory sqlSessionFactory(DataSource dataSource) {
        try {
            SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
            sqlSessionFactoryBean.setDataSource(dataSource);
            sqlSessionFactoryBean.setMapperLocations(
                    new PathMatchingResourcePatternResolver()
                            .getResources("classpath:mapper/**/*.xml")
            );

            SqlSessionFactory sqlSessionFactory = sqlSessionFactoryBean.getObject();

            if (sqlSessionFactory == null) {
                throw new RuntimeException("SqlSessionFactory 생성 실패");
            }

            org.apache.ibatis.session.Configuration configuration = sqlSessionFactory.getConfiguration();
            configuration.setMapUnderscoreToCamelCase(true);
            configuration.setJdbcTypeForNull(JdbcType.NULL);
            return sqlSessionFactory;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

`@Qualifier`로 `DataSourceConfig`에 등록해둔 `DataSource` 빈을 직접 주입받아 `SqlSessionFactory`를 생성했다. `SqlSessionFactory`는 `sqlSessionFactoryBean`의 `getObject` 메소드로 반환되는 객체를 사용하며, 주입받은 `DataSource`를 참조해 MyBatis와 MySQL 서버를 연동한다. `setMapperLocations`로 mapper 파일(xml)의 위치를 지정했고, DB 컬럼이 Snake Case로 되어 있어서 Java에서 쓰기 편하도록 Camel Case로 자동 매핑되게 `setMapUnderscoreToCamelCase`를 설정했다. 마지막으로 생성한 `SqlSessionFactory`로 `SqlSessionTemplate`을 만들었다.

#### 동작 확인

mapper 파일에 테스트용 쿼리를 작성하고, DAO 인터페이스에 메소드를 선언한 뒤 테스트 코드를 작성해서 호출해봤다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.api.domain.db.dao.CrudTestDao">
    <select id="getRecord" resultType="java.util.Map">
        SELECT
            id
        FROM CRUD_TEST
        LIMIT 1
    </select>
</mapper>
```

```java
public interface CrudTestDao {
    Map<String, Object> getRecord();
}
```

```java
@SpringBootTest
@ActiveProfiles("local")
class CrudTestDaoTest {
    @Autowired
    @Qualifier("dbMasterSqlSessionTemplate")
    private SqlSessionTemplate dbMasterSqlSessionTemplate;

    @Test
    void getRecordFromMaster() {
        Map<String, Object> result = dbMasterSqlSessionTemplate.getMapper(CrudTestDao.class).getRecord();

        assertThat(result).containsKey("id");
    }
}
```

테스트를 실행해보니 DB에 저장된 정보를 정상적으로 조회할 수 있었다. `dbMasterSqlSessionTemplate`을 주입받아 master 커넥션 풀이 정상 동작하는 것을 확인했고, 같은 방식으로 `dbSlaveSqlSessionTemplate`을 주입받는 테스트를 하나 더 작성해 slave 커넥션 풀도 동일하게 테스트했다. 두 커넥션 풀 모두 정상적으로 연동되는 것을 확인했다.
