# spring-db2

김영한 스프링 DB 2편 학습 저장소 (2회차 진행 중, 1회차 기록은 `first-study` 태그)

## Stack

- 구조: 루트 composite build (`settings.gradle`의 `includeBuild`)
  - `itemservice-db/`: 섹션 1~8 (데이터 접근 기술: JdbcTemplate, MyBatis, JPA, Spring Data JPA, Querydsl)
  - `springtx/`: 섹션 9~11 (스프링 트랜잭션 이해, 전파)
- Java 11 (`sourceCompatibility`), 로컬 실행 JDK 17
- Spring Boot 2.6.5, io.spring.dependency-management 1.0.11.RELEASE
- Gradle 7.4.1 (루트 wrapper 하나만 사용, 하위 프로젝트에는 wrapper 없음)
- itemservice-db: web, thymeleaf, lombok (강의 진행에 따라 jdbc·h2·mybatis·jpa·querydsl 추가)
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
