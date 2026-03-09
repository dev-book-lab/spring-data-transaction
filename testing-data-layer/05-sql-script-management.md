# @Sql 스크립트 관리 — executionPhase, 격리 전략, @SqlConfig 트랜잭션 제어

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Sql`의 `executionPhase`(`BEFORE_TEST_METHOD` / `AFTER_TEST_METHOD`)는 어떤 순서로 실행되는가?
- `@Sql`이 적합한 상황과 ObjectMother/Builder 패턴이 더 나은 상황을 어떻게 구분하는가?
- `@SqlConfig`로 SQL 스크립트의 트랜잭션 처리 방식을 어떻게 제어하는가?
- 클래스 레벨과 메서드 레벨 `@Sql`이 충돌할 때 어떤 규칙으로 처리되는가?
- `@Sql` 스크립트 파일 경로 규칙과 자동 탐색 경로는?

---

## 🔍 왜 이게 존재하는가

### 문제: 코드 기반 픽스처와 SQL 기반 픽스처는 각자 적합한 상황이 다르다

```java
// 코드 기반 픽스처 (ObjectMother, TestEntityManager):
//   장점: 타입 안전, 엔티티 변경 시 컴파일 오류로 즉시 감지
//   단점: 복잡한 FK 관계, 대량 데이터 삽입이 장황함

// SQL 기반 픽스처 (@Sql):
//   장점: 복잡한 레거시 DB 상태 정확 표현, 대량 데이터 빠른 삽입
//   단점: 엔티티 변경 시 SQL 파일 수동 업데이트, 가독성 낮음

// 예: 5개 테이블, 각 50개 행, FK 제약 복잡
// → 코드로 짜면 200줄 셋업 코드
// → SQL 파일 1개면 명확하고 빠름

// 반대: 단순 User 1~2개 삽입
// → @Sql 파일 만들기 vs 한 줄 em.persist()
// → 코드 기반이 훨씬 간단
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Sql은 항상 @Transactional 테스트 롤백 범위에 포함된다

```java
// ❌ 잘못된 이해: @Sql 삽입 데이터도 테스트 롤백 시 같이 사라진다

// ✅ 실제: @Sql의 트랜잭션은 @SqlConfig로 별도 제어 가능

// 기본 동작: ISOLATED (독립 트랜잭션으로 실행 후 커밋)
// → 테스트 @Transactional 롤백과 무관하게 커밋됨
// → 테스트 후 데이터 잔류 가능

// AFTER_TEST_METHOD @Sql로 정리하지 않으면 다음 테스트에 영향!
```

### Before: 클래스 레벨 @Sql과 메서드 레벨 @Sql은 누적 적용된다

```java
// ❌ 잘못된 이해: 클래스 @Sql + 메서드 @Sql이 모두 실행됨

// ✅ 실제: 메서드 레벨이 클래스 레벨을 "대체"한다 (기본값)
// mergeMode 설정으로 변경 가능

@DataJpaTest
@Sql("/sql/common-data.sql") // 클래스 레벨
class UserTest {

    @Test
    @Sql("/sql/specific-data.sql") // 메서드 레벨 → common-data.sql 실행 안 됨!
    void test1() { }

    @Test
    // @Sql 없음 → 클래스 레벨 common-data.sql 실행
    void test2() { }
}
```

---

## 🔬 내부 동작 원리 — @Sql 실행 순서 소스 추적

### 1. executionPhase 실행 순서

```java
// JUnit 5 + Spring 통합 실행 순서:
//
// [1] @BeforeAll (static)
// [2] 컨테이너/트랜잭션 시작 (beforeTestMethod)
// [3] @Sql(phase=BEFORE_TEST_METHOD) ← SQL 실행
// [4] @BeforeEach
// [5] @Test 실행
// [6] @AfterEach
// [7] @Sql(phase=AFTER_TEST_METHOD) ← SQL 실행
// [8] 트랜잭션 롤백/커밋 (afterTestMethod)
// [9] @AfterAll (static)

// 중요: AFTER_TEST_METHOD @Sql은 롤백 전에 실행
// → @Transactional 테스트에서 AFTER_TEST_METHOD SQL도 롤백 대상

// AFTER_TEST_METHOD로 정리 SQL이 효과있으려면:
// @SqlConfig(transactionMode=ISOLATED) 설정 필요
// (독립 트랜잭션으로 커밋 → 롤백 영향 없음)
```

### 2. @Sql 기본 사용 패턴

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE) // 실제 DB
@Testcontainers
class SqlAnnotationTest extends MySQLContainerBase {

    @Autowired UserRepository repo;

    // [패턴 1] BEFORE만 — @Transactional 롤백으로 정리
    @Test
    @Sql("/sql/insert-users.sql") // 기본: BEFORE_TEST_METHOD
    @Transactional
    void queryWithPreparedData() {
        List<User> users = repo.findAllByStatus(ACTIVE);
        assertEquals(3, users.size());
        // 테스트 종료 → 롤백 → SQL로 삽입한 데이터 사라짐
    }

    // [패턴 2] BEFORE + AFTER 쌍
    @Test
    @SqlGroup({
        @Sql(
            scripts = "/sql/insert-orders.sql",
            executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD
        ),
        @Sql(
            scripts = "/sql/cleanup-orders.sql",
            executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD
        )
    })
    void orderQueryTest() {
        // insert-orders.sql로 준비된 데이터로 테스트
        List<Order> orders = orderRepo.findPendingOrders();
        assertEquals(5, orders.size());
    } // cleanup-orders.sql 실행 → 정리

    // [패턴 3] 클래스 레벨 공통 데이터
    // @Sql("/sql/common-categories.sql") // 클래스 레벨에 붙임
    // → 각 테스트 메서드 전에 공통 데이터 삽입
}
```

### 3. @SqlConfig — 트랜잭션 처리 방식 제어

```java
// TransactionMode 옵션:
// DEFAULT (기본): 테스트의 트랜잭션 동기화 상태에 따름
//   → @Transactional 테스트: 테스트 트랜잭션 내에서 실행 (같이 롤백)
//   → 비트랜잭션 테스트: 독립 트랜잭션으로 실행 후 커밋
// ISOLATED: 항상 독립 트랜잭션으로 실행 후 커밋 (롤백 무관)
// INLINED: 현재 트랜잭션 내에서 실행 (트랜잭션 없으면 예외)

// [사용 예 1] ISOLATED — 테스트 롤백 후에도 데이터 유지
@Test
@Sql(
    scripts = "/sql/reference-data.sql",
    config = @SqlConfig(transactionMode = SqlConfig.TransactionMode.ISOLATED)
)
void testWithPersistentRefData() {
    // reference-data.sql은 독립 커밋 → 테스트 롤백 후에도 남아있음
    // 단, AFTER_TEST_METHOD cleanup @Sql도 ISOLATED로 설정해야 확실히 정리됨
}

// [사용 예 2] 인코딩, 구분자 설정
@Test
@Sql(
    scripts = "/sql/korean-data.sql",
    config = @SqlConfig(
        encoding = "UTF-8",
        separator = ";",          // SQL 구문 구분자 (기본 ";")
        commentPrefix = "--",     // 주석 접두사
        transactionMode = SqlConfig.TransactionMode.DEFAULT
    )
)
void koreanDataTest() { ... }

// [사용 예 3] 클래스 레벨 @SqlConfig 공유
@DataJpaTest
@SqlConfig(transactionMode = SqlConfig.TransactionMode.ISOLATED)
class IsolatedSqlTest {
    // 클래스 내 모든 @Sql이 ISOLATED 모드로 실행
}
```

### 4. 클래스 레벨 vs 메서드 레벨 @Sql 병합 규칙

```java
// 기본: 메서드 레벨이 클래스 레벨 대체

@DataJpaTest
@Sql("/sql/common.sql") // 클래스 레벨
class MergeTest {

    @Test
    void usesClassLevel() {
        // @Sql 없음 → 클래스 레벨 common.sql 실행
    }

    @Test
    @Sql("/sql/specific.sql") // 메서드 레벨 → common.sql 실행 안 됨!
    void usesMethodLevel() {
        // specific.sql만 실행 (common.sql 무시)
    }

    @Test
    @SqlGroup({
        @Sql("/sql/common.sql"),   // 명시적으로 클래스 레벨 포함
        @Sql("/sql/specific.sql")  // 추가 SQL
    })
    void usesBoth() {
        // common.sql + specific.sql 둘 다 실행
    }
}

// @Sql 병합 활성화 (Spring 6.2+)
@Test
@Sql(scripts = "/sql/extra.sql", mergeMode = Sql.MergeMode.MERGE_WITH_ENCLOSING)
void mergedWithClassLevel() {
    // common.sql + extra.sql 모두 실행 (MERGE_WITH_ENCLOSING)
}
```

### 5. SQL 파일 경로 규칙과 자동 탐색

```java
// @Sql 경로 유형:

// [1] classpath 루트 기준 절대 경로
@Sql("/sql/insert-users.sql") // src/test/resources/sql/insert-users.sql

// [2] 상대 경로 (테스트 클래스 패키지 기준)
@Sql("insert-users.sql")
// com.example.UserTest → src/test/resources/com/example/insert-users.sql

// [3] 파일명 생략 → 자동 탐색
@Sql // scripts 없음
// UserTest.sql → src/test/resources/com/example/UserTest.sql
// 메서드 기준: UserTest.testMethodName.sql

// 파일 구조 예:
// src/test/resources/
//   sql/
//     cleanup.sql              # 공통 정리
//     insert-base-data.sql     # 기본 참조 데이터
//   com/example/user/
//     UserRepositoryTest.sql   # 자동 탐색 경로
//     insert-users.sql         # 명시적 참조

// 환경별 SQL 분리
// src/test/resources/sql/h2/cleanup.sql
// src/test/resources/sql/mysql/cleanup.sql
// → @ActiveProfiles("mysql") + 프로파일별 경로 분기
```

### 6. @Sql이 적합한 상황과 아닌 상황

```
@Sql 적합한 상황:
  ✅ 대량 데이터 (100건 이상) 빠른 삽입
  ✅ FK 제약이 복잡해 코드 순서 제어가 어려운 경우
  ✅ 레거시 DB 스키마로 ORM 매핑이 어려운 경우
  ✅ 특정 DB 상태(복잡한 집계, 트리거 결과)를 정확히 표현할 때
  ✅ 성능 테스트용 대규모 데이터셋
  ✅ Flyway 마이그레이션 검증 시 기준 데이터

@Sql이 부적합한 상황:
  ❌ 단순 엔티티 1~3개 삽입 → TestEntityManager/ObjectMother가 더 간결
  ❌ 엔티티 변경이 잦은 경우 → SQL 파일 수동 업데이트 부담
  ❌ 테스트 의도를 코드에서 파악해야 하는 경우 (가독성)
  ❌ 타입 안전성이 중요한 경우 (SQL은 컴파일 타임 검증 없음)

코드 기반이 더 나은 경우:
  @BeforeEach + TestEntityManager.persist()
  ObjectMother.activeUser() 등 명명된 픽스처
  Test Builder로 세밀한 커스터마이징
  → 엔티티 변경 시 컴파일 오류로 즉시 감지
```

---

## 💻 실험으로 확인하기

### 실험 1: executionPhase 실행 순서 로그로 확인

```java
@DataJpaTest
class SqlPhaseOrderTest {

    @Autowired UserRepository repo;

    @Test
    @SqlGroup({
        @Sql(
            scripts = "/sql/insert-3-users.sql",
            executionPhase = Sql.ExecutionPhase.BEFORE_TEST_METHOD
        ),
        @Sql(
            scripts = "/sql/delete-all-users.sql",
            executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD
        )
    })
    void orderConfirmed() {
        // BEFORE 실행 후: 3명 있음
        assertEquals(3, repo.count());
        // 테스트 종료 → AFTER 실행 → DELETE
    }

    @Test
    void afterCleanupVerify() {
        // 이전 테스트의 AFTER @Sql + 롤백 → 데이터 0
        assertEquals(0, repo.count());
    }
}
```

### 실험 2: @SqlConfig(ISOLATED) vs DEFAULT 차이

```java
@SpringBootTest // @Transactional 없는 통합 테스트
class SqlConfigModeTest {

    @Autowired UserRepository repo;

    @Test
    @Sql(
        scripts = "/sql/insert-users.sql",
        config = @SqlConfig(transactionMode = SqlConfig.TransactionMode.ISOLATED)
    )
    @Transactional
    void isolatedCommitsIndependently() {
        // @Sql ISOLATED: 독립 커밋 → 테스트 롤백 후에도 남음
        assertEquals(3, repo.count()); // 테스트 중: 3명 있음
    }
    // 테스트 종료 → @Transactional 롤백 → 테스트 중 삽입한 데이터는 사라짐
    // but ISOLATED @Sql 데이터는 커밋됐으므로 남아있음 → 다음 테스트 주의!

    @Test
    @Sql(
        scripts = "/sql/insert-users.sql",
        config = @SqlConfig(transactionMode = SqlConfig.TransactionMode.DEFAULT)
    )
    @Transactional
    void defaultRunsInTestTransaction() {
        // @Sql DEFAULT: 테스트 트랜잭션 내 실행
        assertEquals(3, repo.count()); // 테스트 중: 3명 있음
    }
    // 테스트 종료 → @Transactional 롤백 → @Sql 데이터도 같이 롤백 → 깨끗
}
```

---

## ⚡ 성능 임팩트

```
@Sql 실행 비용:

파일 파싱 (첫 실행): ~수 ms (파일 크기에 비례)
SQL 실행: 행 수 × INSERT 비용

비교:
  100건 INSERT via @Sql: ~50ms (배치 미사용, 개별 INSERT)
  100건 INSERT via JdbcTemplate.batchUpdate(): ~10ms
  100건 INSERT via TestEntityManager 반복: ~80ms

대용량 픽스처 최적화:
  SQL 파일에서 단건 INSERT 반복: 느림
  → "INSERT INTO t VALUES (1,...),(2,...),(3,...)" 멀티-VALUES INSERT: 빠름
  → 또는 MySQL LOAD DATA / PostgreSQL COPY 구문

@Sql 캐싱:
  파일 파싱 결과는 캐시됨 → 두 번째 실행부터 파싱 없음
  SQL 실행 자체는 매번 발생
```

---

## 🤔 트레이드오프

```
@Sql:
  ✅ 대량/복잡 데이터 표현 간결
  ✅ 레거시 DB 상태 정확 재현
  ❌ 엔티티 변경 시 수동 업데이트
  ❌ 타입 안전 없음 (런타임 오류)
  ❌ 테스트 의도 코드에서 보이지 않음

코드 기반 픽스처 (TestEntityManager, ObjectMother):
  ✅ 타입 안전, 컴파일 검증
  ✅ 엔티티 변경 즉시 감지
  ✅ 테스트 의도 명확
  ❌ 대량 데이터 장황
  ❌ 복잡한 FK 순서 관리 부담

혼합 전략 (권장):
  단순 도메인 객체: 코드 기반 (ObjectMother, TestEntityManager)
  대량/레거시 데이터: @Sql
  공통 참조 데이터 (코드성 데이터): 클래스 레벨 @Sql
```

---

## 📌 핵심 정리

```
executionPhase 실행 순서
  BEFORE_TEST_METHOD → @BeforeEach → @Test → @AfterEach → AFTER_TEST_METHOD
  → 트랜잭션 롤백 전에 AFTER_TEST_METHOD 실행

@SqlConfig TransactionMode
  DEFAULT: 테스트 트랜잭션 상태에 따름
    → @Transactional 테스트: 같은 트랜잭션 내 실행 (같이 롤백)
    → 비트랜잭션: 독립 커밋
  ISOLATED: 항상 독립 트랜잭션으로 커밋
    → 테스트 롤백 무관하게 데이터 유지

클래스 vs 메서드 @Sql
  메서드 레벨이 클래스 레벨 대체 (기본)
  Spring 6.2+: mergeMode=MERGE_WITH_ENCLOSING으로 병합 가능
  @SqlGroup으로 명시적으로 둘 다 포함 가능

파일 경로
  절대 경로: "/sql/file.sql" (classpath 루트)
  상대 경로: "file.sql" (테스트 클래스 패키지 기준)
  자동 탐색: scripts 생략 → ClassName.sql 또는 ClassName.methodName.sql

@Sql 적합 상황
  대량 데이터, 복잡한 FK, 레거시 DB, 성능 테스트
  단순 엔티티 1~3개: TestEntityManager/ObjectMother 권장
```

---

## 🤔 생각해볼 문제

**Q1.** `@DataJpaTest`(기본 H2)에서 `@Sql`로 MySQL 전용 구문(`ON DUPLICATE KEY UPDATE`)을 실행하면 어떻게 되는가?

**Q2.** 클래스 레벨 `@Sql`과 메서드 레벨 `@SqlGroup`이 같이 있을 때, 메서드 레벨 `@SqlGroup`이 클래스 레벨 `@Sql`을 대체하는가?

**Q3.** `@Sql`을 `@AfterEach` 대신 `AFTER_TEST_METHOD`로 사용할 때, 테스트가 예외로 실패하면 `AFTER_TEST_METHOD` SQL이 실행되는가?

> 💡 **해설**
>
> **Q1.** `JdbcSQLSyntaxErrorException`이 발생한다. `@Sql`은 JDBC를 통해 SQL 파일의 구문을 직접 실행한다. H2는 `ON DUPLICATE KEY UPDATE`를 지원하지 않으므로 구문 오류가 발생하고 테스트가 실패한다. `MODE=MySQL`을 H2 URL에 설정해도 이 구문은 지원되지 않는다. 해결: (1) H2용 대체 구문(`MERGE INTO`)을 별도 파일로 준비하고 프로파일에 따라 분기. (2) Testcontainers로 실제 MySQL 사용. (3) `@Sql` 대신 코드 기반 픽스처로 대체.
>
> **Q2.** 대체된다. 메서드 레벨 `@SqlGroup`이 클래스 레벨 `@Sql`을 완전히 대체하므로, 클래스 레벨 SQL은 실행되지 않는다. `@SqlGroup` 내의 `@Sql`들만 실행된다. 클래스 레벨 SQL을 함께 실행하려면 `@SqlGroup` 내에 클래스 레벨 SQL 경로를 명시적으로 포함시켜야 한다. Spring 6.2+에서는 `mergeMode = MERGE_WITH_ENCLOSING`으로 병합할 수 있다.
>
> **Q3.** 실행된다. `AFTER_TEST_METHOD` SQL은 테스트 성공/실패와 무관하게 항상 실행된다. 이는 JUnit의 `@AfterEach`가 예외 발생 시에도 실행되는 것과 같은 원리다. Spring의 `SqlScriptsTestExecutionListener`가 `afterTestMethod()` 콜백에서 `AFTER_TEST_METHOD` SQL을 실행하며, 이 콜백은 테스트 결과에 관계없이 호출된다. 따라서 정리(cleanup) 목적의 `AFTER_TEST_METHOD` SQL은 테스트 실패 시에도 실행되어 데이터를 정리한다.

---

<div align="center">

**[⬅️ 이전: Repository 테스트 전략](./04-repository-test-strategy.md)** | **[홈으로 🏠](../README.md)**

</div>
