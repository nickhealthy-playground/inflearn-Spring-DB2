# 인프런 강의

해당 저장소의 `README.md`는 인프런 김영한님의 Spring Boot 강의 시리즈를 듣고 Spring 프레임워크의 방대한 기술들을 복기하고자 공부한 내용을 가볍게 정리한 것입니다. 문제가 될 시 삭제하겠습니다.



## 해당 프로젝트에서 배우는 내용

* 섹션 3 | 데이터 접근 기술 - 스프링 JdbcTemplate
* 섹션 4 | 데이터 접근 기술 - 테스트



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
