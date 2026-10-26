# 인프런 강의

해당 저장소의 `README.md`는 인프런 김영한님의 Spring Boot 강의 시리즈를 듣고 Spring 프레임워크의 방대한 기술들을 복기하고자 공부한 내용을 가볍게 정리한 것입니다. 문제가 될 시 삭제하겠습니다.



## 해당 프로젝트에서 배우는 내용

* 섹션 3 | 데이터 접근 기술 - 스프링 JdbcTemplate
* 섹션 4 | 데이터 접근 기술 - 테스트
* 섹션 5 | 데이터 접근 기술 - MyBatis
* 섹션 6 | 데이터 접근 기술 - JPA



# 섹션 3 | 데이터 접근 기술 - 스프링 JdbcTemplate

<details>
<summary><b>핵심 정리 펼쳐보기</b></summary>

<br>

> SQL을 직접 작성하면서 JDBC의 반복 작업(커넥션 획득·반납, `PreparedStatement`·`ResultSet` 처리, 예외 변환, 트랜잭션 커넥션 동기화)을 템플릿 콜백 패턴으로 대신 처리해주는 기술.
> `spring-boot-starter-jdbc` 만 추가하면 별도 설정 없이 사용할 수 있다.

### 한눈에 보기

| 구분 | API | 핵심 |
|---|---|---|
| 템플릿 | `JdbcTemplate` | `?` 순서 기반 파라미터 바인딩 |
| 템플릿 | `NamedParameterJdbcTemplate` | `:name` 이름 기반 파라미터 바인딩 **(권장)** |
| 파라미터 | `Map` | `Map.of("id", id)` 처럼 단순 전달 |
| 파라미터 | `MapSqlParameterSource` | `addValue()` 메서드 체인, SQL 타입 지정 가능 |
| 파라미터 | `BeanPropertySqlParameterSource` | 객체의 getter로 파라미터 자동 생성 (`getItemName()` → `itemName`) |
| 결과 매핑 | `RowMapper` | `ResultSet` 한 행 → 객체 변환 로직만 구현 (루프는 템플릿이 돌림) |
| 결과 매핑 | `BeanPropertyRowMapper` | 기본 생성자로 객체를 만들고, 컬럼명과 일치하는 프로퍼티의 setter를 호출 (`item_name` → `setItemName()`) |
| 키 조회 | `KeyHolder` (`GeneratedKeyHolder`) | DB가 생성한 PK(identity)를 INSERT 후 조회 |
| INSERT | `SimpleJdbcInsert` | INSERT SQL 없이 테이블명·키 컬럼만 지정, `executeAndReturnKey()` 로 키 반환 |
| 프로시저 | `SimpleJdbcCall` | 스토어드 프로시저 호출 |

### 주요 메서드

| 메서드 | 용도 | 반환 / 예외 |
|---|---|---|
| `queryForObject()` | 단건 조회 (객체 또는 `Integer.class`·`String.class` 같은 단일 값) | 결과 0건 → `EmptyResultDataAccessException`<br>결과 2건 이상 → `IncorrectResultSizeDataAccessException` |
| `query()` | 목록 조회 | `List<T>`, 결과 없으면 빈 리스트 |
| `update()` | INSERT · UPDATE · DELETE | 영향받은 row 수 (`int`) |
| `execute()` | DDL 등 임의 SQL | - |

### 파라미터 전달 방식 비교

| 방식 | 장점 | 쓰기 어려운 경우 |
|---|---|---|
| `Map` | 가장 단순 | 파라미터가 많으면 장황해짐 |
| `MapSqlParameterSource` | 객체에 없는 값도 자유롭게 추가 | 필드를 일일이 `addValue()` 해야 함 |
| `BeanPropertySqlParameterSource` | 객체 하나로 자동 생성 | SQL에 필요한 값이 객체에 없을 때<br>(예: `ItemUpdateDto` 에는 `id` 가 없어서 `update()` 에서는 `MapSqlParameterSource` 사용) |

### 간단한 사용법

```java
// 관례: DataSource를 주입받아 생성자에서 템플릿 생성 (템플릿을 빈으로 등록해서 주입받아도 됨)
public JdbcTemplateItemRepositoryV3(DataSource dataSource) {
    this.template = new NamedParameterJdbcTemplate(dataSource);
    this.jdbcInsert = new SimpleJdbcInsert(dataSource)
            .withTableName("item")
            .usingGeneratedKeyColumns("id");
}

// 저장: INSERT SQL 없이 키까지 반환
Number key = jdbcInsert.executeAndReturnKey(new BeanPropertySqlParameterSource(item));

// 단건 조회: 결과 없음 예외 → Optional.empty()
try {
    Item item = template.queryForObject(sql, Map.of("id", id), BeanPropertyRowMapper.newInstance(Item.class));
    return Optional.of(item);
} catch (EmptyResultDataAccessException e) {
    return Optional.empty();
}
```

```properties
# src/main/resources/application.properties (test 설정은 분리되어 있으므로 필요하면 양쪽에 추가)
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
# JdbcTemplate이 실행하는 SQL 로그
logging.level.org.springframework.jdbc=debug
```

`spring.datasource.*` 만 설정하면 스프링 부트가 커넥션 풀(`DataSource`)과 트랜잭션 매니저를 빈으로 자동 등록한다.

### 해결하는 것 vs 한계

| 해결하는 것 | 한계 |
|---|---|
| JDBC 반복 코드 제거 (커넥션·리소스 정리, 결과 루프) | **동적 쿼리 작성이 어렵다** — 조건에 따라 `where`/`and` 를 문자열로 직접 조립해야 함 (최대 단점) |
| 스프링 예외 변환기로 `DataAccessException` 계층으로 변환 | SQL을 자바 문자열로 작성 → 줄이 길어지면 `+` 로 이어 붙여야 함 |
| 트랜잭션 커넥션 동기화 (`@Transactional` 과 함께 동작) | 결과 매핑은 여전히 개발자 몫 (`BeanPropertyRowMapper` 로 일부 자동화) |
| ORM(JPA)을 쓰면서도 SQL을 직접 써야 할 때 함께 사용 가능 | → 동적 쿼리와 SQL 작성 편의는 다음 섹션의 **MyBatis** 가 해결 |

### 주의사항

| 주의 | 내용 |
|---|---|
| 순서 기반 바인딩 | SQL의 `?` 순서를 바꾸고 인자 순서를 안 바꾸면 `price` ↔ `quantity` 처럼 **데이터가 조용히 잘못 저장**된다. 컴파일·실행 에러가 나지 않아 발견이 늦고, DB 데이터 복구까지 필요 → 이름 기반(`NamedParameterJdbcTemplate`) 사용 |
| `BeanPropertyRowMapper` 매칭 규칙 | 프로퍼티 이름을 소문자(`itemname`)와 snake_case(`item_name`) 두 가지로 등록해두고, 컬럼명을 소문자로 바꿔 비교한다. 그래서 `item_name`·`ITEM_NAME`·`itemName` 모두 매핑된다. 이름 자체가 다르면 SQL 별칭으로 맞춘다 (`select member_name as username`) |
| 매핑 실패가 조용함 | 일치하는 프로퍼티가 없는 컬럼은 **예외 없이 무시**되어 필드가 `null`·기본값으로 남는다. 매핑 대상 클래스에는 기본 생성자와 setter가 필요 |
| `queryForObject()` 결과 없음 | `null` 이 아니라 예외가 발생 → 인터페이스가 `Optional` 을 반환한다면 예외를 잡아서 `Optional.empty()` 로 변환 |
| `SimpleJdbcInsert` 메타데이터 조회 시점 | 생성자에서는 설정값만 저장하고, **첫 INSERT 실행 시점**에 테이블 메타데이터를 조회해 INSERT SQL을 만든다. 그래서 테이블이 없거나 이름이 틀려도 애플리케이션은 정상 기동하고, 첫 저장 시점에 실패한다. `usingColumns()` 는 특정 컬럼만 저장할 때만 지정 |
| 설정 파일 위치 | DB 접속 정보는 `src/main` 의 `application.properties` 에 설정 (`src/test` 와 분리되어 있음) |
| 데이터 누적 | 메모리 저장소와 달리 DB는 서버를 내려도 유지 → `local` 프로필에서 `TestDataInit` 이 재시작마다 `itemA`, `itemB` 를 계속 추가 |

</details>



# 섹션 4 | 데이터 접근 기술 - 테스트

<details>
<summary><b>핵심 정리 펼쳐보기</b></summary>

<br>

> DB에 연동하는 테스트는 **다른 테스트와 격리**되고 **반복 실행**할 수 있어야 한다.
> 테스트마다 트랜잭션을 시작하고 끝나면 롤백하는 방식(`@Transactional`)과, 테스트 전용 임베디드 DB로 이 두 원칙을 지킨다.

### 문제 → 해결 흐름

| 단계 | 방법 | 남는 문제 |
|---|---|---|
| 1 | 로컬 서버와 같은 DB로 테스트 | 서버 실행 중 쌓인 데이터가 조회 결과에 섞여 테스트 실패 |
| 2 | 테스트 전용 DB 분리 (`~/testcase`) | 이전 테스트가 남긴 데이터 때문에 **두 번째 실행부터 실패** |
| 3 | 테스트 끝에 `DELETE` | 테스트 도중 예외·강제 종료 시 삭제가 실행되지 않아 데이터가 남음 |
| 4 | `@BeforeEach` 트랜잭션 시작 + `@AfterEach` 롤백 | 커밋하지 않으므로 중간에 죽어도 데이터가 반영되지 않음. 단, 테스트마다 반복 코드 |
| 5 | 테스트에 `@Transactional` | 4번을 애노테이션 하나로 해결. 단, 별도 DB 서버를 띄워야 함 |
| 6 | 임베디드 H2 + `schema.sql` | DB 서버 없이 JVM 메모리 안에서 실행, 테이블은 스크립트로 자동 생성 |
| 7 | DB 접속 설정 제거 | 스프링 부트가 임베디드 DB `DataSource` 를 자동 생성 |

### 한눈에 보기

| 구분 | API / 설정 | 핵심 |
|---|---|---|
| 테스트 컨텍스트 | `@SpringBootTest` | `@SpringBootApplication` 을 찾아 설정으로 사용 → 애플리케이션과 같은 빈 구성(`@Import` 된 Config 포함)으로 테스트 |
| 수동 트랜잭션 | `PlatformTransactionManager` | `getTransaction(new DefaultTransactionDefinition())` 으로 시작, `rollback(status)` 으로 되돌림 |
| 테스트 트랜잭션 | `@Transactional` (테스트 클래스·메서드) | 테스트를 트랜잭션 안에서 실행하고 **끝나면 항상 롤백** |
| 커밋 강제 | `@Commit` / `@Rollback(false)` | 롤백 대신 커밋. DB에 남은 결과를 직접 확인하고 싶을 때만 |
| 임베디드 DB 직접 등록 | `jdbc:h2:mem:db;DB_CLOSE_DELAY=-1` | `mem` = JVM 메모리에서 동작, `DB_CLOSE_DELAY=-1` = 커넥션이 모두 닫혀도 DB를 유지 |
| 스키마 초기화 | `src/test/resources/schema.sql` | 애플리케이션 시작 시 스프링 부트가 실행해서 테이블 생성 |
| 부트 임베디드 DB | `spring.datasource.url` 미설정 | 클래스패스에 H2가 있으면 `jdbc:h2:mem:<랜덤 UUID>;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE` 로 자동 생성 |
| DB 이름 고정 | `spring.datasource.generate-unique-name=false` | 랜덤 이름 대신 `testdb` 사용 |

### 테스트에서 `@Transactional` 동작

| 순서 | 동작 |
|---|---|
| 1 | 테스트 메서드 실행 전 트랜잭션 시작 (`@BeforeEach` 보다 먼저) |
| 2 | 테스트 로직 실행. 리포지토리의 `JdbcTemplate` 도 같은 트랜잭션(= 같은 커넥션)을 사용 → 방금 저장한 데이터를 조회 가능 |
| 3 | 서비스·리포지토리의 `@Transactional` 은 새 트랜잭션을 만들지 않고 테스트 트랜잭션에 참여 |
| 4 | 테스트 종료 후(`@AfterEach` 다음) 트랜잭션 롤백 → 저장한 데이터 제거 |

### 간단한 사용법

```java
@Transactional          // org.springframework.transaction.annotation.Transactional
@SpringBootTest
class ItemRepositoryTest {
    // 테스트마다 트랜잭션 시작 → 종료 후 롤백
}
```

```sql
-- src/test/resources/schema.sql
drop table if exists item CASCADE;
create table item
(
    id        bigint generated by default as identity,
    item_name varchar(10),
    price     integer,
    quantity  integer,
    primary key (id)
);
```

```properties
# src/test/resources/application.properties
spring.profiles.active=test
# spring.datasource.url 을 지정하지 않으면 임베디드 H2 사용
logging.level.org.springframework.jdbc=debug
```

### 해결하는 것 vs 한계

| 해결하는 것 | 한계 |
|---|---|
| 테스트 후 데이터 정리 코드가 필요 없음 | 테스트 트랜잭션 **밖에서 커밋된 데이터**는 롤백되지 않음 (별도 스레드, 새 트랜잭션을 여는 코드 등) |
| 중간에 실패·강제 종료돼도 커밋 전이라 데이터가 남지 않음 | 항상 롤백되므로 **커밋 시점에만 일어나는 동작**은 검증하지 못함 |
| DB 서버 설치·운영 없이 테스트 실행 (임베디드 H2) | 임베디드 H2는 운영 DB(MySQL 등)와 SQL 문법·제약·락 동작이 달라, 통과해도 운영 DB에서 실패할 수 있음 |

### 주의사항

| 주의 | 내용 |
|---|---|
| 설정 파일 | 테스트 실행 시 클래스패스에서 `src/test/resources/application.properties` 가 먼저 발견되어 사용되고, `src/main` 의 같은 이름 파일은 **읽히지 않는다 (병합 아님)**. DB 접속·로그 설정이 필요하면 test 쪽에 따로 적어야 함 |
| 애노테이션 위치 | 롤백 동작은 `@Transactional` 이 **테스트 클래스나 테스트 메서드**에 있을 때만 적용. 서비스에만 붙어 있으면 테스트에서도 정상 커밋된다 |
| `schema.sql` 실행 조건 | `spring.sql.init.mode` 기본값이 `embedded` 라서 **임베디드 DB일 때만** 실행된다. TCP로 접속하는 H2 서버 DB에는 실행되지 않음 (필요하면 `always`) |
| `schema.sql` 위치·이름 | 클래스패스 루트의 `schema.sql` (또는 `schema-${platform}.sql`) 을 찾는다. 테스트 전용이면 `src/test/resources` 에 둔다 |
| 같은 DB를 공유하는 테스트 | 테스트 전용 DB라도 롤백 없이 데이터를 남기면 다른 테스트·다음 실행에 영향 → `findItems()` 처럼 전체 조회 결과를 검증하는 테스트가 먼저 깨진다 |
| 임베디드 DB 수명 | 애플리케이션(JVM)이 종료되면 데이터와 DB가 함께 사라짐. 직접 URL을 쓸 때 `DB_CLOSE_DELAY=-1` 이 없으면 마지막 커넥션이 닫히는 순간 DB가 사라진다 |

</details>



# 섹션 5 | 데이터 접근 기술 - MyBatis

<details>
<summary><b>핵심 정리 펼쳐보기</b></summary>

<br>

> SQL을 XML(또는 애노테이션)에 작성하고, 매퍼 인터페이스 메서드를 호출하면 그 SQL을 실행해주는 SQL Mapper.
> JdbcTemplate의 기능에 더해 **동적 쿼리**와 **여러 줄 SQL**을 편하게 작성할 수 있다. 대신 의존성과 약간의 설정이 필요하다.
> (사용 버전: `mybatis-spring-boot-starter 2.2.0` → mybatis 3.5.7, mybatis-spring 2.0.6. 스프링 부트가 버전을 관리하지 않아 버전을 직접 적는다)

### 한눈에 보기

| 구분 | API / 설정 | 핵심 |
|---|---|---|
| 매퍼 | `@Mapper` 인터페이스 | 메서드 하나 = SQL 하나. 구현체는 직접 만들지 않음 |
| SQL 정의 | 매퍼 XML (`<select>`·`<insert>`·`<update>`·`<delete>`) | `id` = 매퍼 메서드 이름, `namespace` = 매퍼 인터페이스의 전체 이름 |
| 파라미터 바인딩 | `#{name}` | `PreparedStatement` 의 `?` 로 바인딩 (안전) |
| 문자열 치환 | `${name}` | 값을 SQL 문자열에 그대로 삽입 → **SQL 인젝션 위험** |
| 키 조회 | `useGeneratedKeys="true" keyProperty="id"` | INSERT 후 DB가 생성한 키를 파라미터 객체의 `id` 에 채움 |
| 결과 매핑 | `resultType="Item"` | 컬럼명과 일치하는 프로퍼티에 값을 채움 |
| 결과 매핑 | `<resultMap>` | 컬럼명 ↔ 프로퍼티명을 직접 지정 (별칭 `as` 대신) |
| 여러 파라미터 | `@Param("name")` | 파라미터에 이름을 붙여 `#{updateParam.itemName}` 처럼 참조 |
| 애노테이션 SQL | `@Select` · `@Insert` · `@Update` · `@Delete` | XML 없이 SQL 작성. 동적 쿼리가 어려워 단순한 SQL에만 |
| SQL 재사용 | `<sql>` + `<include>` | SQL 조각을 정의하고 끼워 넣음 |

### 동적 SQL

| 태그 | 동작 |
|---|---|
| `<if test="...">` | 조건(OGNL)이 참이면 내용을 추가 |
| `<choose>` / `<when>` / `<otherwise>` | 자바 `switch` 처럼 첫 번째로 맞는 하나만 추가 |
| `<where>` | 내용이 비어 있으면 `WHERE` 를 붙이지 않고, 있으면 `WHERE` 를 붙인 뒤 맨 앞의 `AND ` / `OR ` 를 제거 |
| `<set>` | UPDATE의 `SET` 절을 만들고 앞뒤에 남는 쉼표를 정리 |
| `<trim prefix prefixOverrides>` | `<where>` · `<set>` 을 직접 정의하는 범용 버전 |
| `<foreach>` | 컬렉션을 반복해 `in (1, 2, 3)` 같은 구문 생성 |

### 설정

```groovy
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
```

```properties
# main, test 의 application.properties 모두에 추가 (test 설정은 main 설정을 대체한다)
# resultType 에 패키지명 생략 (하위 패키지 포함)
mybatis.type-aliases-package=hello.itemservice.domain
# item_name → itemName 매핑
mybatis.configuration.map-underscore-to-camel-case=true
# 매퍼가 실행하는 SQL 로그
logging.level.hello.itemservice.repository.mybatis=trace
# XML을 원하는 위치에 둘 때
#mybatis.mapper-locations=classpath:mapper/**/*.xml
```

### 간단한 사용법

```java
@Mapper
public interface ItemMapper {
    void save(Item item);
    void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto updateParam);
    Optional<Item> findById(Long id);
    List<Item> findAll(ItemSearchCond itemSearch);
}
```

```xml
<!-- src/main/resources/hello/itemservice/repository/mybatis/ItemMapper.xml -->
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">
    <insert id="save" useGeneratedKeys="true" keyProperty="id">
        insert into item (item_name, price, quantity)
        values (#{itemName}, #{price}, #{quantity})
    </insert>
    <select id="findAll" resultType="Item">
        select id, item_name, price, quantity
        from item
        <where>
            <if test="itemName != null and itemName != ''">
                and item_name like concat('%',#{itemName},'%')
            </if>
            <if test="maxPrice != null">
                and price &lt;= #{maxPrice}
            </if>
        </where>
    </select>
</mapper>
```

### 매퍼 인터페이스가 동작하는 원리

| 순서 | 동작 |
|---|---|
| 1 | 스프링 부트 자동 설정(`MybatisAutoConfiguration`)이 **`@SpringBootApplication` 클래스의 패키지 이하**에서 `@Mapper` 인터페이스를 찾는다. 컴포넌트 스캔 범위(`scanBasePackages`)와는 별개라서, 컴포넌트 스캔을 `web` 패키지로 좁혀도 매퍼는 등록된다 |
| 2 | 찾은 인터페이스마다 `MapperFactoryBean` 을 빈으로 등록한다 |
| 3 | 빈을 꺼낼 때 MyBatis가 `Proxy.newProxyInstance` 로 **JDK 동적 프록시** 구현체를 만든다 |
| 4 | 프록시 메서드 호출 → `namespace` + 메서드 이름으로 SQL을 찾아 실행 → 결과를 반환 타입으로 매핑 |
| 5 | SQL 실행은 `SqlSessionTemplate` 을 거친다. 여기서 스프링 트랜잭션의 커넥션을 함께 쓰고, MyBatis 예외를 스프링의 `DataAccessException` 계층으로 변환한다 |

### JdbcTemplate vs MyBatis

| 비교 | JdbcTemplate | MyBatis |
|---|---|---|
| 설정 | 스프링 내장, 별도 설정 없음 | 의존성 + 프로퍼티 설정 필요 |
| SQL 작성 | 자바 문자열 (`+` 로 이어 붙임) | XML에 여러 줄 그대로 |
| 동적 쿼리 | 자바 코드로 직접 조립 | `<if>`, `<where>`, `<foreach>` 등 태그 |
| 선택 기준 | 단순한 쿼리 위주 | 동적·복잡한 쿼리가 많을 때 |

### 해결하는 것 vs 한계

| 해결하는 것 | 한계 |
|---|---|
| 동적 쿼리를 태그로 선언 (JdbcTemplate의 최대 단점 해결) | SQL은 여전히 직접 작성해야 함 → 테이블·컬럼이 바뀌면 SQL을 모두 고쳐야 함 |
| SQL이 자바 코드와 분리되어 읽기 쉬움 | XML과 인터페이스가 분리되어 오타·불일치가 **실행 시점**에 드러남 |
| 매퍼 인터페이스만으로 DAO 구현 (반복 코드 제거) | 연관 객체 조회(`<association>`, `<collection>`)는 작성 공수가 크고 성능 최적화가 어려움 |

### 주의사항

| 주의 | 내용 |
|---|---|
| XML 위치·이름 | `mapper-locations` 를 지정하지 않으면 매퍼 인터페이스의 **패키지 경로 + 인터페이스 이름 + `.xml`** 을 클래스패스에서 찾는다 (`hello/itemservice/repository/mybatis/ItemMapper.xml`). 경로나 파일명이 다르면 SQL을 찾지 못한다 |
| XML 특수문자 | `<`, `>`, `&` 는 태그로 해석되므로 `&lt;`, `&gt;`, `&amp;` 로 쓰거나 `<![CDATA[ ... ]]>` 로 감싼다. CDATA 안에서는 `<if>`, `<where>` 같은 태그도 문자로 취급된다 |
| `${}` 사용 | 값이 SQL에 그대로 들어가므로 사용자 입력에 쓰면 SQL 인젝션. 정렬 컬럼명처럼 꼭 필요할 때만, 허용 값 목록으로 검증한 뒤 사용 |
| `@Param` | MyBatis는 기본적으로 실제 파라미터 이름을 쓰고(`useActualParamName=true`), 스프링 부트 Gradle 플러그인은 `-parameters` 옵션으로 컴파일한다. 그래서 이 프로젝트에선 `@Param` 없이도 이름으로 찾을 수 있다. 하지만 컴파일 옵션에 의존하므로 파라미터가 2개 이상이면 `@Param` 으로 명시하는 것이 안전 |
| 매핑 실패가 조용함 | `map-underscore-to-camel-case` 는 컬럼명의 `_` 를 지우고 프로퍼티 이름과 대소문자 구분 없이 비교한다. 일치하는 프로퍼티가 없는 컬럼은 기본 설정(`autoMappingUnknownColumnBehavior=NONE`)에서 **예외 없이 무시**된다 |
| 설정 파일 | test 의 `application.properties` 가 main 설정을 대체하므로 MyBatis 설정을 양쪽에 모두 넣어야 테스트에서도 적용된다 |

</details>



# 섹션 6 | 데이터 접근 기술 - JPA

<details>
<summary><b>핵심 정리 펼쳐보기</b></summary>

<br>

> 객체와 테이블을 매핑해두면 JPA가 SQL을 만들어 실행하는 ORM 표준(구현체: 하이버네이트).
> 개발자는 컬렉션을 다루듯 객체를 저장·조회·수정하고, 변경 감지·예외 변환 같은 반복 작업은 JPA와 스프링이 처리한다.
> (사용 버전: Spring Boot 2.6.5 → Hibernate 5.6.7, spring-data-jpa 2.6.3, `javax.persistence` 패키지)

### 한눈에 보기

| 구분 | API / 설정 | 핵심 |
|---|---|---|
| 엔티티 | `@Entity` | JPA가 관리하는 객체. `public` 또는 `protected` 기본 생성자 필수 |
| PK | `@Id` + `@GeneratedValue(strategy = IDENTITY)` | DB가 PK를 생성 (auto increment) |
| 컬럼 | `@Column(name, length)` | 필드 ↔ 컬럼 매핑. `length` 는 DDL 생성 시 `varchar` 길이로 사용 |
| 핵심 객체 | `EntityManager` | 저장·조회·쿼리 등 JPA의 모든 동작이 이 객체를 통해 이뤄짐 |
| 저장 | `em.persist(entity)` | INSERT. IDENTITY 전략이면 실행 후 생성된 PK를 엔티티 `id` 에 채움 |
| 단건 조회 | `em.find(Entity.class, id)` | PK로 조회. 없으면 `null` |
| 수정 | (메서드 없음) 조회한 엔티티의 값 변경 | **변경 감지**: flush 시점에 바뀐 엔티티를 찾아 UPDATE 실행 |
| 목록 조회 | JPQL `em.createQuery("select i from Item i", Item.class)` | 테이블이 아니라 **엔티티와 필드**를 대상으로 하는 쿼리. 파라미터는 `:name` + `setParameter()` |
| 트랜잭션 | `@Transactional` | 데이터 변경은 트랜잭션 안에서만 가능 (조회는 없어도 됨). 보통 서비스 계층에 붙인다 |
| 예외 변환 | `@Repository` | JPA 예외를 스프링 `DataAccessException` 으로 바꾸는 AOP 프록시 적용 대상 |

### 설정

```groovy
// spring-boot-starter-jdbc 를 포함하므로 jdbc 스타터는 제거해도 된다 (mybatis 스타터도 jdbc 스타터를 포함)
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

```properties
# 하이버네이트가 실행하는 SQL (logger로 출력)
logging.level.org.hibernate.SQL=DEBUG
# SQL에 바인딩되는 파라미터 값 (Hibernate 5 기준 로거 이름)
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
# spring.jpa.show-sql=true 는 System.out 으로 출력하므로 비권장 (logger 와 같이 켜면 중복 출력)
```

### 간단한 사용법

```java
@Repository      // 예외 변환 AOP 대상
@Transactional   // 예제라서 리포지토리에 붙임. 실무에서는 서비스 계층
public class JpaItemRepository implements ItemRepository {

    private final EntityManager em;

    public JpaItemRepository(EntityManager em) { this.em = em; }

    public Item save(Item item) {
        em.persist(item);                                  // INSERT
        return item;
    }

    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = em.find(Item.class, itemId);
        findItem.setItemName(updateParam.getItemName());   // update 호출 없음 → 변경 감지
    }

    public List<Item> findAll(ItemSearchCond cond) {
        return em.createQuery("select i from Item i where i.price <= :maxPrice", Item.class)
                .setParameter("maxPrice", cond.getMaxPrice())
                .getResultList();
    }
}
```

### 동작 원리

| 동작 | 실제로 일어나는 일 |
|---|---|
| 컬럼 이름 결정 | 스프링 부트가 `CamelCaseToUnderscoresNamingStrategy` 를 기본 적용 → `@Column` 이 없어도 `itemName` 필드는 `item_name` 컬럼에 매핑 |
| `EntityManager` 생성자 주입 | 주입되는 것은 실제 `EntityManager` 가 아니라 **공유 프록시**(`SharedEntityManagerCreator`). 호출 시점의 트랜잭션에 묶인 실제 `EntityManager` 로 위임하므로 싱글톤 빈에 주입해도 안전. 생성자 주입이 되는 건 spring-data-jpa가 이 프록시를 빈으로 등록해주기 때문 |
| `persist()` + IDENTITY | PK를 DB가 만들기 때문에 **`persist()` 호출 즉시 INSERT** 를 실행한다 (커밋까지 미루지 않음) |
| 변경 감지 | 트랜잭션 커밋 직전(flush)에 처음 조회한 상태와 비교해 바뀐 엔티티만 UPDATE |
| 같은 트랜잭션 안의 `find()` | 이미 조회·저장한 엔티티는 DB를 다시 조회하지 않고 같은 객체를 돌려준다 (영속성 컨텍스트, JPA 기본편 주제) |
| 예외 변환 | 스프링 부트가 `PersistenceExceptionTranslationPostProcessor` 를 등록 → **클래스에 `@Repository` 가 붙은 빈**에 예외 변환 프록시를 적용. 컴포넌트 스캔이 아니라 `@Bean` 으로 등록해도 적용된다 |

### 해결하는 것 vs 한계

| 해결하는 것 | 한계 |
|---|---|
| 기본 CRUD SQL을 직접 쓰지 않음 (테이블·컬럼 변경 시 매핑만 수정) | **동적 쿼리는 여전히 문자열 조립** (JPQL도 `where`/`and` 를 직접 붙임) → Querydsl 이 해결 |
| 수정 시 `update` 호출 없이 객체 값만 바꾸면 됨 (변경 감지) | SQL이 언제 실행되는지 코드에 보이지 않음 → 영속성 컨텍스트·flush 시점을 모르면 디버깅이 어려움 |
| JPA 예외도 스프링 예외 계층으로 변환 | 학습량이 많음 (영속성 컨텍스트, 연관관계, 지연 로딩 등은 별도 강의 분량) |

### 주의사항

| 주의 | 내용 |
|---|---|
| 테스트에서 UPDATE 미실행 | 테스트는 끝에 롤백되므로 flush가 일어나지 않아 **UPDATE SQL이 실행되지 않는다**. 같은 트랜잭션의 `find()` 는 DB를 다시 조회하지 않고 메모리의 같은 객체를 돌려주므로, `updateItem()` 테스트는 DB 반영 여부와 상관없이 통과한다. 실제 SQL까지 검증하려면 `em.flush()` 후 `em.clear()` 하고 다시 조회하거나 `@Commit` 사용 |
| 테스트 테이블은 엔티티 기준 | 임베디드 DB이고 마이그레이션 도구가 없으면 `spring.jpa.hibernate.ddl-auto` 기본값이 `create-drop` 이다. `schema.sql` 이 먼저 테이블을 만들어도 하이버네이트가 **지우고 엔티티 매핑대로 다시 만든다** (종료 시 삭제). 그래서 `schema.sql` 과 엔티티 매핑이 달라도 테스트에서는 드러나지 않음. 임베디드가 아닌 DB에서는 기본값이 `none` |
| 기본 생성자 | JPA가 리플렉션으로 엔티티를 생성하므로 `public`/`protected` 기본 생성자가 없으면 조회가 실패한다 |
| JPQL 대소문자 | 엔티티 이름(`Item`)과 필드 이름(`itemName`)은 대소문자를 구분한다 |
| 트랜잭션 위치 | 변경이 트랜잭션 밖에서 일어나면 반영되지 않거나 예외가 발생한다. 예제처럼 리포지토리에 `@Transactional` 을 붙인 건 서비스 로직이 단순해서이고, 원칙은 유스케이스를 시작하는 서비스 계층 |
| SQL 로그 설정 | 파라미터 로거 이름은 하이버네이트 버전에 따라 다르다. Hibernate 5(이 프로젝트)는 `org.hibernate.type.descriptor.sql.BasicBinder` |

</details>
