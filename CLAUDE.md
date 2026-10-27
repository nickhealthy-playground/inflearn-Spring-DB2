# spring-db2

김영한 스프링 DB 2편 학습 저장소 (2회차 진행 중, 1회차 기록은 `first-study` 태그)

## Stack

- 구조: 루트 composite build (`settings.gradle`의 `includeBuild`)
  - 섹션 번호는 인프런 커리큘럼 기준 (섹션 1 = 강의 소개, 교안 PDF 번호 + 1)
  - `itemservice-db/`: 섹션 2~9 (데이터 접근 기술: 시작, JdbcTemplate, 테스트, MyBatis, JPA, Spring Data JPA, Querydsl, 활용 방안)
  - `springtx/`: 섹션 10~12 (스프링 트랜잭션 이해, 전파)
- Java 11 (`sourceCompatibility`), 로컬 실행 JDK 17
- Spring Boot 2.6.5, io.spring.dependency-management 1.0.11.RELEASE
- Gradle 7.4.1 (루트 wrapper 하나만 사용, 하위 프로젝트에는 wrapper 없음)
- itemservice-db: web, thymeleaf, data-jpa (→ Hibernate 5.6.5 고정, spring-data-jpa 2.6.3, jdbc 스타터 포함이라 jdbc 스타터는 주석 처리), h2(runtime), lombok, mybatis-spring-boot-starter 2.2.0 (→ mybatis 3.5.7, mybatis-spring 2.0.6) (강의 진행에 따라 querydsl 추가)
  - `ext["hibernate.version"] = "5.6.5.Final"`: Boot 2.6.5 기본 Hibernate 5.6.7은 LIKE 쿼리 메서드에서 `Parameter value [\] did not match expected type` 버그 (5.6.6~5.6.7). 이 고정을 지우지 말 것
  - Hibernate 5: 바인딩 파라미터 로그는 `org.hibernate.type.descriptor.sql.BasicBinder=TRACE` (Hibernate 6의 `org.hibernate.orm.jdbc.bind` 아님)
  - 테스트(임베디드 H2)에서는 `ddl-auto` 기본값이 `create-drop` → `schema.sql` 실행 후 하이버네이트가 엔티티 기준으로 테이블을 다시 만듦
  - MyBatis 스타터는 Boot BOM 관리 대상이 아니라 버전을 직접 지정. Boot 2.x에는 2.2.x 계열 사용 (3.x는 Boot 3 전용)
- springtx: data-jpa, h2(runtime), lombok

### 명령 (루트에서 실행)
- 테스트: `./gradlew :itemservice-db:test`, `./gradlew :springtx:test`
- 실행: `./gradlew :itemservice-db:bootRun`

### 버전별 주의
- Spring Boot 2.6은 OSS 지원 종료(EOL). 강의 재현 목적이므로 업그레이드하지 않음
- Boot 2.x → `javax.*` 패키지 사용 (`jakarta.*` 아님)
- Querydsl은 Boot 2.x 기준 `com.querydsl:querydsl-jpa` (classifier 없음) + `querydsl-apt:...:jpa`

### 컨벤션
- 학습 진행 커밋: `섹션 N | <강의 제목>`, 그 외 커밋: `<type>: <한국어 설명>`
