# Testcontainers vs H2 선택 기준 — 호환 모드의 한계와 실제 DB 구성 전략

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- H2 호환 모드(`MODE=MySQL`)가 실제 MySQL과 다르게 동작하는 구체적인 사례는?
- `@DynamicPropertySource`가 Spring 컨텍스트에 DB 접속 정보를 주입하는 정확한 시점은?
- Testcontainers `static @Container`와 인스턴스 `@Container`의 생명주기 차이는?
- CI 환경에서 컨테이너 재사용(`withReuse(true)`)을 활성화하는 방법과 주의점은?
- 테스트 클래스 간 컨테이너를 공유해 속도를 높이는 패턴은?

---

## 🔍 왜 이게 존재하는가

### 문제: H2가 실제 DB를 완전히 대체하지 못한다

```java
// MySQL 전용 기능 — H2에서 실패하는 사례들

// [사례 1] JSON 함수
@Query(value = "SELECT * FROM products WHERE JSON_CONTAINS(tags, :tag)",
       nativeQuery = true)
List<Product> findByTag(@Param("tag") String tag);
// H2: JSON_CONTAINS 미지원 → org.h2.jdbc.JdbcSQLSyntaxErrorException
// MySQL 8.0: 정상 동작

// [사례 2] DB 전용 문자열 함수
@Query(value = "SELECT DATE_FORMAT(created_at, '%Y-%m') FROM users",
       nativeQuery = true)
List<String> findMonthlyStats();
// H2: DATE_FORMAT 미지원 (H2는 FORMATDATETIME 사용)

// [사례 3] ON DUPLICATE KEY UPDATE (MySQL Upsert)
@Modifying
@Query(value = "INSERT INTO stats(date, count) VALUES(:date, 1) " +
               "ON DUPLICATE KEY UPDATE count = count + 1", nativeQuery = true)
void upsertStats(@Param("date") LocalDate date);
// H2: 문법 오류 → MERGE INTO 구문 사용해야 함

// [사례 4] FULLTEXT INDEX 검색
@Query(value = "SELECT * FROM articles WHERE MATCH(content) AGAINST(:keyword)",
       nativeQuery = true)
List<Article> fullTextSearch(@Param("keyword") String keyword);
// H2: FULLTEXT INDEX 개념 없음

// → H2로 통과한 테스트가 운영(MySQL)에서 실패하는 이유
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: H2 MySQL 호환 모드(MODE=MySQL)로 모든 차이를 해결할 수 있다

```yaml
# ❌ MODE=MySQL로 완전 호환될 것이라는 기대
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=MySQL;DB_CLOSE_DELAY=-1
```

```java
// 호환 모드로 해결되는 것:
// - AUTO_INCREMENT → IDENTITY (H2가 인식)
// - TINYINT(1) → BOOLEAN 처리
// - 일부 MySQL 키워드

// 여전히 해결 안 되는 것:
// - JSON_CONTAINS, JSON_EXTRACT 등 JSON 함수
// - DATE_FORMAT 등 날짜 함수
// - FULLTEXT 검색
// - ON DUPLICATE KEY UPDATE
// - MySQL 전용 인덱스 힌트
// - 저장 프로시저, 트리거 (일부)
// - 문자셋/콜레이션 차이 (대소문자 구분)

// → 호환 모드는 "약간의 차이"를 줄이는 것, 완전 대체 불가
```

### Before: Testcontainers는 Docker가 느려서 CI에서 쓰기 어렵다

```
❌ 잘못된 이해: 매 테스트마다 컨테이너 시작 → 느림

✅ 올바른 전략:
  static @Container: 테스트 클래스당 1회만 시작
  AbstractContainerBase 공유: 클래스 간 컨테이너 공유
  withReuse(true): JVM 재시작 간 컨테이너 재사용
  GitHub Actions: runner에 Docker 기본 포함 → 별도 설정 불필요

실제 시간:
  컨테이너 시작 (MySQL 8.0): ~5~8초 (1회)
  이후 모든 테스트: 컨테이너 공유 → 쿼리 실행 시간만
  → @SpringBootTest 전체 컨텍스트 로딩(10~30초)보다 짧을 수 있음
```

---

## 🔬 내부 동작 원리 — @DynamicPropertySource 주입 시점

### 1. Testcontainers 컨테이너 생명주기

```java
// @Container + @Testcontainers → TestcontainersExtension 등록 (JUnit 5)

// static @Container → 클래스 생명주기
// JUnit 수행 순서:
// 1. @BeforeAll → container.start() (TestcontainersExtension 처리)
// 2. @DynamicPropertySource 호출 (컨테이너 시작 후 → getJdbcUrl() 가능)
// 3. Spring ApplicationContext 초기화 (DB URL 주입됨)
// 4. @BeforeEach → 테스트 데이터 준비
// 5. @Test 실행
// 6. @AfterEach → 정리
// 7. @AfterAll → container.stop() (Ryuk 컨테이너가 담당하거나 명시 종료)

// 인스턴스 @Container → 메서드 생명주기
// @BeforeEach → container.start()
// @AfterEach → container.stop()
// → 테스트마다 컨테이너 재시작 → 매우 느림 (권장하지 않음)
```

### 2. @DynamicPropertySource — 컨텍스트 연결

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryMySQLTest {

    // static → 클래스 전체에서 1개 컨테이너 공유
    @Container
    static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("testuser")
        .withPassword("testpass");

    // 컨테이너 시작 후, Spring Context 초기화 전 호출
    @DynamicPropertySource
    static void mysqlProperties(DynamicPropertyRegistry registry) {
        // 이 시점: MYSQL.start() 완료 → getJdbcUrl() 안전하게 호출 가능
        registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
        // getJdbcUrl() = "jdbc:mysql://localhost:49152/testdb"
        // 49152 = Docker가 랜덤 할당한 호스트 포트

        registry.add("spring.datasource.username", MYSQL::getUsername);
        registry.add("spring.datasource.password", MYSQL::getPassword);
        registry.add("spring.datasource.driver-class-name",
            () -> "com.mysql.cj.jdbc.Driver");

        // JPA 설정
        registry.add("spring.jpa.database-platform",
            () -> "org.hibernate.dialect.MySQL8Dialect");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
    }

    @Autowired UserRepository userRepository;

    @Test
    void findByEmailOnRealMySQL() {
        // 실제 MySQL 8.0에서 실행
        userRepository.save(new User("홍길동", "hong@test.com"));
        Optional<User> found = userRepository.findByEmail("hong@test.com");
        assertTrue(found.isPresent());
    }

    @Test
    void jsonContainsWorks() {
        // MySQL 전용 JSON 함수 — H2에서는 실패하지만 여기서는 OK
        userRepository.save(new User("홍길동", List.of("admin", "user")));
        List<User> admins = userRepository.findByTag("\"admin\"");
        assertEquals(1, admins.size()); // ✅
    }
}
```

### 3. 컨테이너 공유 패턴 — AbstractContainerBase

```java
// 여러 테스트 클래스에서 같은 컨테이너(+ApplicationContext) 재사용

// 베이스 클래스 — 공통 컨테이너 정의
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
abstract class MySQLContainerBase {

    @Container
    static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
        // withReuse(true) 추가 시 JVM 간 재사용 가능

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
        registry.add("spring.datasource.username", MYSQL::getUsername);
        registry.add("spring.datasource.password", MYSQL::getPassword);
        registry.add("spring.datasource.driver-class-name",
            () -> "com.mysql.cj.jdbc.Driver");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
    }
}

// 테스트 클래스들 — 베이스 상속 → 같은 컨테이너 + ApplicationContext 공유
class UserRepositoryTest extends MySQLContainerBase {
    @Autowired UserRepository userRepository;

    @Test
    void findByEmail() { ... }
}

class TeamRepositoryTest extends MySQLContainerBase {
    @Autowired TeamRepository teamRepository;

    @Test
    void findByName() { ... }
}

// Spring ApplicationContext 캐시 조건:
// 동일한 @ContextConfiguration + 동일한 @DynamicPropertySource 결과
// → 같은 베이스 클래스 상속 → 동일 컨텍스트 → 1회 초기화, 공유
```

### 4. withReuse(true) — JVM 간 컨테이너 재사용

```java
// withReuse(true): 테스트 실행 후 컨테이너를 종료하지 않고 유지
// 다음 실행 시 같은 컨테이너 재사용 → 시작 시간 ~0.3초로 단축

@Container
static final MySQLContainer<?> MYSQL = new MySQLContainer<>("mysql:8.0")
    .withDatabaseName("testdb")
    .withUsername("test")
    .withPassword("test")
    .withReuse(true); // ← 재사용 활성화

// 추가 설정 파일 필요:
// ~/.testcontainers.properties
// testcontainers.reuse.enable=true

// 동작 원리:
// 컨테이너에 재사용 라벨 부착 (해시 기반)
// 다음 실행 시 같은 라벨의 컨테이너 탐색
// 발견하면 start() 없이 그대로 사용

// 주의: 컨테이너가 재사용되면 데이터가 남아있음
// → ddl-auto=create-drop: 스키마 재생성으로 데이터 초기화
// → @BeforeEach TRUNCATE: 행 레벨 초기화
// 둘 중 하나 반드시 적용 필요

// CI 환경에서 withReuse:
// GitHub Actions: 매 실행마다 새 환경 → withReuse 효과 없음 (매번 새 컨테이너)
// 로컬 개발 반복 실행: 큰 효과 (컨테이너 시작 시간 절약)
```

### 5. H2 vs Testcontainers 선택 기준

```
H2 인메모리 선택:
  ✅ 표준 JPA/JPQL만 사용하는 경우
  ✅ Spring Data Query Method (findByName, findAllByStatus)
  ✅ 네이티브 쿼리 없거나 매우 단순한 경우
  ✅ 빠른 피드백이 최우선인 경우
  ✅ Docker 없는 환경

  적합한 테스트:
  - @Repository 기본 CRUD
  - JPQL/HQL 정확성
  - JPA 연관관계 매핑
  - 영속성 컨텍스트 동작

Testcontainers (실제 DB) 선택:
  ✅ DB 전용 함수 사용 (JSON, 날짜 포맷, 전문검색)
  ✅ 네이티브 쿼리 (ON DUPLICATE KEY, UPSERT 등)
  ✅ DB 고유 인덱스/제약조건 동작 검증
  ✅ DB 트리거, 뷰, 저장 프로시저
  ✅ 대소문자 구분, 콜레이션 의존 로직
  ✅ 운영 환경과 동일한 신뢰성이 필요한 통합 테스트

  적합한 테스트:
  - 네이티브 쿼리 정확성
  - DB 전용 기능 활용 Repository
  - 마이그레이션 스크립트 검증 (Flyway/Liquibase)
  - 성능 테스트 (실제 인덱스 동작)

혼합 전략 (권장):
  단위/슬라이스 → H2 (@DataJpaTest 기본)
  DB 방언 의존 → Testcontainers (@DataJpaTest + replace=NONE)
  서비스 통합 → Testcontainers (@SpringBootTest + replace=NONE)
```

---

## 💻 실험으로 확인하기

### 실험 1: H2에서 실패하는 MySQL 쿼리 확인

```java
// H2 환경 (@DataJpaTest 기본)
@DataJpaTest
class H2FailureTest {

    @Autowired TestEntityManager em;

    @Test
    void mysqlJsonFunctionFails() {
        // H2에서 JSON_CONTAINS 실행 시도
        assertThrows(Exception.class, () ->
            em.getEntityManager()
              .createNativeQuery(
                  "SELECT * FROM users WHERE JSON_CONTAINS(roles, '\"admin\"')")
              .getResultList()
        );
        // JdbcSQLSyntaxErrorException: Function "JSON_CONTAINS" not found
    }
}

// MySQL 컨테이너 환경
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class MySQLSuccessTest extends MySQLContainerBase {

    @Autowired TestEntityManager em;

    @Test
    void mysqlJsonFunctionWorks() {
        // 동일 쿼리 — MySQL에서는 정상 동작
        assertDoesNotThrow(() ->
            em.getEntityManager()
              .createNativeQuery(
                  "SELECT * FROM users WHERE JSON_CONTAINS(roles, '\"admin\"')")
              .getResultList()
        );
    }
}
```

### 실험 2: 컨테이너 시작 시간 측정

```java
@Testcontainers
class ContainerTimingTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0");

    private static long startTime;

    @BeforeAll
    static void recordStart() {
        startTime = System.currentTimeMillis();
    }

    @Test
    @Order(1)
    void firstTest() {
        long elapsed = System.currentTimeMillis() - startTime;
        System.out.println("컨테이너 준비 시간: " + elapsed + "ms");
        // 최초: ~5000~8000ms (Docker 이미지 준비 + 컨테이너 시작)
        // withReuse + 이미지 캐시: ~300ms
    }
}
```

---

## ⚡ 성능 임팩트

```
컨테이너 시작 비용 (MySQL 8.0):
  Docker 이미지 pull (최초): ~30~60초
  컨테이너 시작 (이후):      ~5~8초
  withReuse=true:           ~0.3초

테스트 클래스 10개, 각 5개 메서드 기준:

인스턴스 @Container:
  10 × 5 × 8초 = 400초 (6분 이상)

static @Container (클래스별):
  10 × 8초 = 80초

AbstractContainerBase (컨테이너 1개 공유):
  1 × 8초 = 8초

AbstractContainerBase + withReuse:
  1 × 0.3초 = 0.3초 (로컬 반복 실행)

H2 (@DataJpaTest 기본):
  컨테이너 없음, 인메모리 → ~0.5초 (컨텍스트 초기화)

결론:
  H2가 가장 빠름 (DB 방언 무관 시)
  Testcontainers는 공유 전략으로 H2와 근접 속도 달성 가능
```

---

## 🤔 트레이드오프

```
H2:
  ✅ 가장 빠름, Docker 불필요
  ❌ DB 방언 차이, 운영 버그 미발견
  → 표준 JPQL, 빠른 피드백 우선

Testcontainers:
  ✅ 운영 동일 환경, 방언 버그 사전 발견
  ✅ CI/CD 친화 (GitHub Actions Docker 기본 지원)
  ❌ Docker 필요, 컨테이너 시작 비용
  → DB 전용 기능, 정확한 통합 테스트

withReuse(true):
  ✅ 로컬 개발 반복 실행 빠름
  ❌ 데이터 잔류 → 정리 전략 필수
  ❌ CI에서는 효과 없음 (매번 새 환경)
  → 로컬 개발 전용
```

---

## 📌 핵심 정리

```
H2 한계
  JSON 함수, DB 전용 날짜 함수 미지원
  ON DUPLICATE KEY UPDATE 문법 불일치
  FULLTEXT 검색 없음
  대소문자 구분 차이
  → MODE=MySQL로도 완전 해결 불가

@DynamicPropertySource 실행 순서
  컨테이너 start() 완료 →
  @DynamicPropertySource 호출 (getJdbcUrl() 가능) →
  Spring ApplicationContext 초기화 (URL 주입됨)

컨테이너 공유 전략
  static @Container: 클래스당 1회 (권장)
  AbstractContainerBase: 클래스 간 컨테이너 + Context 공유
  withReuse(true): JVM 간 재사용 (로컬 개발)

선택 기준
  표준 JPQL만: H2 (@DataJpaTest 기본)
  DB 전용 기능: Testcontainers + replace=NONE
  전체 통합: @SpringBootTest + Testcontainers
```

---

## 🤔 생각해볼 문제

**Q1.** `@DynamicPropertySource`를 인스턴스 메서드(static 없이)로 선언하면 어떻게 되는가?

**Q2.** 두 테스트 클래스가 같은 `MySQLContainerBase`를 상속하지만 한 클래스에서만 `@Sql`로 스키마를 변경하면, 다른 클래스의 테스트에 영향을 주는가?

**Q3.** Testcontainers `withInitScript("schema.sql")`과 `spring.jpa.hibernate.ddl-auto=create-drop`을 동시에 사용하면 어떤 문제가 생기는가?

> 💡 **해설**
>
> **Q1.** 컴파일 오류 또는 런타임 예외가 발생한다. `@DynamicPropertySource`는 반드시 `static` 메서드에 붙여야 한다. Spring은 ApplicationContext 초기화 전에 이 메서드를 호출하는데, 이 시점에 테스트 인스턴스가 아직 생성되지 않았기 때문이다. static이 아닌 경우 `IllegalStateException`이 발생한다.
>
> **Q2.** `ApplicationContext`를 공유하는 경우 스키마 변경은 영향을 준다. 그러나 `@DataJpaTest`는 기본적으로 `ddl-auto=create-drop`을 적용하고, 각 테스트 메서드는 `@Transactional` 롤백으로 데이터를 초기화한다. `@Sql`로 DDL(ALTER TABLE, DROP COLUMN 등)을 실행하면 트랜잭션 롤백이 DDL을 되돌리지 못하는 경우가 있다(DDL은 암묵적 커밋). 이 경우 다른 테스트 클래스의 스키마 상태가 달라져 실패할 수 있다. 해결: DDL을 포함하는 `@Sql`에는 `AFTER_TEST_METHOD`로 원복 스크립트를 추가하거나, 해당 테스트를 별도 컨테이너를 사용하도록 분리한다.
>
> **Q3.** 충돌이 발생할 수 있다. `withInitScript("schema.sql")`은 컨테이너 시작 시 한 번 실행되어 스키마를 생성한다. `ddl-auto=create-drop`은 Spring 컨텍스트 로딩 시 테이블을 다시 생성하고, 종료 시 삭제한다. 컨텍스트 재사용 시 두 번째 로딩에서 `create` 단계가 "이미 존재하는 테이블"에 `CREATE TABLE`을 시도해 오류가 발생할 수 있다. 해결: `ddl-auto=create-drop` 대신 `ddl-auto=validate`(스키마 검증만)를 사용하거나, `withInitScript`와 `ddl-auto` 중 하나만 사용한다. Flyway/Liquibase 사용 시에는 `ddl-auto=none`으로 설정하고 마이그레이션에 완전히 위임하는 것이 권장된다.

---

<div align="center">

**[⬅️ 이전: @Transactional in Test의 함정](./02-transactional-test-pitfall.md)** | **[다음: Repository 테스트 전략 ➡️](./04-repository-test-strategy.md)**

</div>
