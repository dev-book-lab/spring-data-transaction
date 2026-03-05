# readOnly=true의 실제 효과 — FlushMode, 스냅샷, JDBC, 리플리카 라우팅

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Transactional(readOnly = true)`가 Hibernate 내부에서 설정하는 것은 정확히 무엇인가?
- `FlushMode.MANUAL`이 성능에 미치는 실질적인 효과는?
- Dirty Checking(변경 감지) 스냅샷이 readOnly에서 어떻게 달라지는가?
- JDBC 레벨에서 `Connection.setReadOnly(true)`는 실제로 무엇을 하는가?
- readOnly 트랜잭션을 읽기 전용 DB 리플리카로 자동 라우팅하려면 어떻게 하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 조회 전용 메서드에도 변경 감지 오버헤드가 발생한다

```java
// readOnly 없는 조회 — 불필요한 오버헤드 발생
@Transactional  // readOnly 기본값 false
public List<User> findActiveUsers() {
    List<User> users = userRepository.findByStatus(Status.ACTIVE);
    // Hibernate가 users 목록의 스냅샷을 생성 (메모리 점유)
    // flush() 시 스냅샷과 현재 상태 비교 (CPU 소비)
    // 실제로 변경하지 않는데도 이 작업이 수행됨
    return users;
}
```

```
readOnly=true가 해결하는 것:
  FlushMode.MANUAL → flush() 자동 호출 없음 → DB 동기화 생략
  스냅샷 생성 최적화 → 메모리 절감 (Hibernate 버전에 따라 다름)
  Connection.setReadOnly(true) → JDBC 드라이버/DB 힌트
  → 리플리카 라우팅 가능 (LazyConnectionDataSourceProxy 조합 시)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: readOnly=true이면 엔티티 수정이 DB에 반영되지 않는다

```java
// ❌ 잘못된 이해: readOnly 트랜잭션에서 엔티티를 수정하면 예외 발생

// ✅ 실제:
// 예외는 발생하지 않음
// FlushMode.MANUAL이라 자동 flush가 안 될 뿐,
// 명시적으로 em.flush()를 호출하면 UPDATE가 실행됨!

@Transactional(readOnly = true)
public void readOnlyTest() {
    User user = userRepository.findById(1L).get();
    user.setName("변경"); // 메모리 상 변경

    // 자동 flush 없음 → UPDATE 미실행
    // em.flush() 호출 시: UPDATE 실행됨 (readOnly가 완전한 쓰기 차단이 아님!)
}

// readOnly의 실제 역할:
// "이 트랜잭션에서는 쓰기가 없을 것"을 선언 → Hibernate와 JDBC에 힌트 제공
// 강제적인 쓰기 차단이 아님 (일부 DB/드라이버는 차단하기도 함)
```

### Before: readOnly=true이면 Dirty Checking이 완전히 비활성화된다

```java
// ❌ 잘못된 이해: readOnly에서 Hibernate는 스냅샷을 아예 안 만든다

// ✅ Hibernate 버전별로 다름:
// Hibernate 5.x: readOnly 세션에서도 스냅샷 생성 (단, flush 안 함)
// Hibernate 6.x: readOnly 힌트로 스냅샷 생성 최소화 개선
// Spring Data SimpleJpaRepository: @Transactional(readOnly=true)가 기본
//   → 조회 메서드는 readOnly, 쓰기 메서드는 @Transactional(readOnly=false)

// 확실한 최적화: FlushMode.MANUAL → flush() 비용 제거
// 스냅샷 비용은 Hibernate 버전과 설정에 의존
```

---

## ✨ 올바른 이해와 패턴

### After: readOnly=true의 실제 효과 3가지

```
효과 1: Hibernate FlushMode.MANUAL
  → flush() 자동 호출 없음
  → 조회 전 flush (변경사항 DB 반영) 생략
  → 조회 시 JPA가 트랜잭션 커밋 전 flush 생략

효과 2: Connection.setReadOnly(true)
  → JDBC 드라이버에 힌트
  → 일부 DB: 읽기 전용 최적화 모드
  → LazyConnectionDataSourceProxy와 조합 시 리플리카 라우팅

효과 3: Hibernate Session hint
  → Hibernate가 내부 최적화 적용
  → Entity state tracking 최소화 (Hibernate 6.x)
  → 1차 캐시 유지하되 변경 감지 스킵 가능
```

---

## 🔬 내부 동작 원리 — readOnly 처리 소스 추적

### 1. readOnly → Hibernate FlushMode 설정 경로

```java
// JpaTransactionManager.doBegin() — readOnly 처리
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {
    // ...
    EntityManager em = ...; // EntityManager 획득

    // readOnly 설정
    if (definition.isReadOnly()) {
        // [1] Hibernate Session에 FlushMode 설정
        em.unwrap(Session.class).setHibernateFlushMode(FlushMode.MANUAL);
        // → 자동 flush 비활성화

        // [2] Hibernate 5.x: Session을 readOnly로 설정
        em.unwrap(Session.class).setDefaultReadOnly(true);
        // → 로드되는 엔티티를 readOnly로 마킹
    }

    // [3] JDBC Connection readOnly 설정
    DataSourceUtils.prepareConnectionForTransaction(con, definition);
    // → con.setReadOnly(definition.isReadOnly())
}

// DataSourceUtils.prepareConnectionForTransaction()
public static Integer prepareConnectionForTransaction(Connection con,
                                                       TransactionDefinition definition) {
    // readOnly 설정
    if (definition != null && definition.isReadOnly()) {
        con.setReadOnly(true);
        // → JDBC 드라이버에 전달
        // → MySQL: 내부 최적화 힌트 (실질적 차단 없음)
        // → PostgreSQL: 읽기 전용 모드 (쓰기 시도 시 예외)
        // → Oracle: 일부 버전에서 읽기 전용 커넥션 모드
    }
    // ...
}
```

### 2. FlushMode.MANUAL의 실제 효과

```java
// Hibernate의 자동 flush 발생 시점 (FlushMode.AUTO일 때):
//   1. 트랜잭션 커밋 전
//   2. JPQL/Criteria 쿼리 실행 전 (변경사항과 쿼리 결과 일관성 보장)

// FlushMode.MANUAL일 때:
//   자동 flush 없음 → 명시적 em.flush() 없이는 DB에 반영 안 됨

// 성능 효과 비교:
@Service
public class UserQueryService {

    // ❌ readOnly 없음 — 불필요한 flush 발생 가능
    @Transactional
    public List<UserDto> findAllForReport() {
        List<User> users = userRepository.findAll(); // 1000명
        // FlushMode.AUTO: 이 쿼리 실행 전 flush 가능
        // → 다른 곳에서 변경된 엔티티가 있으면 flush 발생

        return users.stream().map(UserDto::from).toList();
        // 트랜잭션 커밋 시 flush → 1000개 엔티티 dirty checking
        // 변경 없어도 스냅샷 비교 수행
    }

    // ✅ readOnly=true — flush 없음
    @Transactional(readOnly = true)
    public List<UserDto> findAllForReportOptimized() {
        List<User> users = userRepository.findAll();
        // FlushMode.MANUAL: 쿼리 전 flush 없음
        // 커밋 시 flush 없음 → dirty checking 없음
        return users.stream().map(UserDto::from).toList();
    }
}
```

### 3. Hibernate readOnly 엔티티 — 스냅샷 최적화

```java
// Session.setDefaultReadOnly(true) 효과
// → 로드되는 모든 엔티티가 readOnly로 마킹

// Hibernate의 readOnly 엔티티 처리:
// 일반 엔티티: EntityEntry 생성 + loadedState(스냅샷) 저장
// readOnly 엔티티: EntityEntry 생성, loadedState = null (스냅샷 없음)

// 스냅샷 메모리 절감 계산 예시:
// User 엔티티 10개 필드, 1000건 조회
// 스냅샷: 10 fields × 1000 records × ~50 bytes = ~500KB
// readOnly: loadedState = null → 스냅샷 메모리 거의 없음

// 단, Hibernate 버전별 차이:
// Hibernate 5.x: setDefaultReadOnly(true)로 스냅샷 생략
// Hibernate 6.x: 추가 최적화, @Immutable 엔티티와 연계
```

### 4. Connection.setReadOnly(true) — DB별 실제 동작

```java
// JDBC readOnly 힌트의 DB별 동작:

// MySQL (Connector/J):
// setReadOnly(true) → com.mysql.cj.Connection 내부 플래그 설정
// 실제 SQL 레벨 차단 없음 (힌트만)
// Replication 드라이버 사용 시: 리플리카로 자동 라우팅
// MySQL URL: jdbc:mysql:replication://master,replica/db
//            → readOnly=true Connection → 자동으로 replica 사용

// PostgreSQL (pgjdbc):
// setReadOnly(true) → SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY
// 실제로 쓰기 시도 시 예외: "cannot execute INSERT in a read-only transaction"

// Oracle:
// setReadOnly(true) → Connection.setReadOnly() → 일부 드라이버만 지원
// 표준 JDBC 스펙에서는 "힌트"이므로 드라이버가 무시할 수 있음

// 결론:
// readOnly=true가 "완전한 쓰기 차단"이 되는가는 DB/드라이버에 의존
// Spring 레벨의 readOnly는 주로 Hibernate 최적화 힌트로 의미 있음
```

### 5. LazyConnectionDataSourceProxy — 리플리카 자동 라우팅

```java
// 리플리카 라우팅 설정 — readOnly=true → replica DB 사용
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource routingDataSource(
            @Qualifier("masterDataSource") DataSource master,
            @Qualifier("replicaDataSource") DataSource replica) {

        ReplicationRoutingDataSource routingDataSource = new ReplicationRoutingDataSource();
        routingDataSource.setDefaultTargetDataSource(master);
        routingDataSource.setTargetDataSources(Map.of(
            "master", master,
            "replica", replica
        ));
        return routingDataSource;
    }

    // LazyConnectionDataSourceProxy가 핵심
    // → 실제 Connection 획득을 최대한 지연
    // → @Transactional의 readOnly 여부 결정 후 적절한 DataSource 선택
    @Bean
    @Primary
    public DataSource dataSource(@Qualifier("routingDataSource") DataSource routing) {
        return new LazyConnectionDataSourceProxy(routing);
    }
}

// 커스텀 라우팅 DataSource
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        // 현재 트랜잭션이 readOnly이면 replica 선택
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        return isReadOnly ? "replica" : "master";
    }
}

// 사용:
@Transactional(readOnly = true)  // → replica DB 사용
public List<User> findAll() { ... }

@Transactional  // → master DB 사용
public User save(User user) { ... }
```

```
LazyConnectionDataSourceProxy가 필요한 이유:
  일반 DataSourceProxy: doBegin()에서 즉시 Connection 획득
  → 이 시점에 아직 readOnly 여부가 TransactionSynchronizationManager에 없을 수 있음

  LazyConnectionDataSourceProxy: 실제 SQL 실행 시까지 Connection 획득 지연
  → @Transactional의 readOnly 설정이 ThreadLocal에 완전히 등록된 후 Connection 획득
  → ReplicationRoutingDataSource.determineCurrentLookupKey()가 정확한 값 반환
```

### 6. SimpleJpaRepository의 readOnly 설계

```java
// SimpleJpaRepository — Spring Data JPA의 기본 readOnly 전략
@Repository
@Transactional(readOnly = true)  // ← 클래스 레벨: 모든 메서드 readOnly 기본값
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {

    // 조회 메서드: readOnly=true (클래스 레벨 상속)
    @Override
    public Optional<T> findById(ID id) { ... }

    @Override
    public List<T> findAll() { ... }

    // 쓰기 메서드: @Transactional으로 readOnly 해제
    @Override
    @Transactional  // readOnly=false (기본값)
    public <S extends T> S save(S entity) { ... }

    @Override
    @Transactional
    public void delete(T entity) { ... }

    @Override
    @Transactional
    public void deleteById(ID id) { ... }
}

// 이 설계의 의미:
// 커스텀 Repository 메서드를 만들 때도 동일 원칙 적용
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // 조회: readOnly=true 상속 (별도 @Transactional 불필요)
    List<User> findByStatus(Status status);

    // 쓰기: @Transactional 명시
    @Modifying
    @Transactional  // readOnly 해제 필수
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    void updateStatus(@Param("id") Long id, @Param("status") Status status);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: FlushMode 확인

```java
@Test
void flushModeWithReadOnly() {
    // readOnly=true 트랜잭션에서 FlushMode 확인
    transactionTemplate.execute(status -> {
        // readOnly 설정
        return null;
    });

    // @Transactional(readOnly = true)에서 Session FlushMode 확인
    // Session 직접 접근
    EntityManager em = entityManagerFactory.createEntityManager();
    em.getTransaction().begin();
    Session session = em.unwrap(Session.class);
    session.setHibernateFlushMode(FlushMode.MANUAL);
    System.out.println("FlushMode: " + session.getHibernateFlushMode());
    // 출력: MANUAL
}
```

### 실험 2: readOnly 트랜잭션에서 수정 시도

```java
@Transactional(readOnly = true)
public void readOnlyModifyTest(Long userId) {
    User user = userRepository.findById(userId).get();
    user.setName("수정됨");  // 메모리 변경

    // flush() 없음 → DB 미반영
    // 트랜잭션 종료 후 DB 확인: "수정됨"이 저장되지 않음
}

@Test
void verifyReadOnlyNotPersisted() {
    User user = userRepository.save(new User("원래이름"));
    testService.readOnlyModifyTest(user.getId());

    // DB에서 다시 조회
    User found = userRepository.findById(user.getId()).get();
    assertEquals("원래이름", found.getName()); // 변경 안 됨
}
```

### 실험 3: 리플리카 라우팅 확인

```java
@Test
void replicaRoutingWithReadOnly() {
    // readOnly=true → replica DataSource 사용
    String replicaResult = testService.queryFromReadOnly();

    // readOnly=false → master DataSource 사용
    String masterResult = testService.queryFromMaster();

    // DataSource URL 확인
    System.out.println("readOnly 쿼리 DataSource: " + getDataSourceUrl(replicaDs));
    System.out.println("쓰기 쿼리 DataSource: " + getDataSourceUrl(masterDs));
}
```

---

## ⚡ 성능 임팩트

```
readOnly=true 성능 개선 효과 측정 예시
(User 엔티티 20개 필드, 1000건 조회, JMH 기준):

[FlushMode.AUTO vs MANUAL]
  AUTO:   155ms (flush + dirty checking 포함)
  MANUAL: 120ms (약 23% 개선)
  → dirty checking 생략 효과 (엔티티 수와 필드 수에 비례)

[스냅샷 메모리]
  readOnly=false: 1000건 × 20 fields × ~64 bytes = ~1.2MB
  readOnly=true:  loadedState=null → ~100KB (약 92% 감소)
  → GC 압력 감소, 특히 대량 조회 시 효과적

[리플리카 라우팅 효과]
  Master 단일: 읽기/쓰기 모두 처리 → 병목
  Master + Replica: 읽기 부하를 Replica로 분산
  → Master 쓰기 처리량 향상
  → 읽기 응답 시간 감소 (Replica가 지리적으로 가까운 경우)

readOnly 효과가 미미한 경우:
  조회 건수가 적을 때 (10건 미만)
  이미 @Transactional(readOnly=false)인 곳에서 조회 + 수정이 섞인 경우
  DB가 MVCC를 충분히 최적화하는 경우
```

---

## 🤔 트레이드오프

```
readOnly=true 사용:
  ✅ FlushMode.MANUAL → flush 비용 제거
  ✅ 스냅샷 메모리 최적화 (Hibernate 버전 의존)
  ✅ JDBC readOnly 힌트 → 드라이버 최적화
  ✅ 리플리카 라우팅 가능 (설정 필요)
  ✅ 코드 의도 명확 ("이 메서드는 읽기 전용")
  ❌ 실수로 수정해도 예외 없음 (일부 DB 제외)
  ❌ 완전한 쓰기 차단이 아님

readOnly=false (기본):
  ✅ 읽기/쓰기 모두 안전
  ✅ 조회 후 수정이 섞인 경우 적합
  ❌ 불필요한 flush 비용
  ❌ 스냅샷 메모리 낭비

실무 권장 패턴:
  서비스 레벨: @Transactional(readOnly = true) 클래스 레벨 기본값
               쓰기 메서드에 @Transactional (readOnly=false) 오버라이드
  Repository 레벨: SimpleJpaRepository 기본 전략 따름 (이미 최적화됨)
  주의: Service에서 @Transactional 없이 Repository 조회 시
        Repository 메서드별 트랜잭션만 적용 → 의도치 않은 경계 분리
```

---

## 📌 핵심 정리

```
readOnly=true의 실제 효과 3가지

[1] Hibernate FlushMode.MANUAL
  JpaTransactionManager.doBegin()에서 설정
  → session.setHibernateFlushMode(FlushMode.MANUAL)
  → 자동 flush 없음 → dirty checking 없음

[2] Connection.setReadOnly(true)
  DataSourceUtils.prepareConnectionForTransaction()에서 설정
  → JDBC 드라이버에 힌트 전달
  → 드라이버/DB별 동작 다름
  → PostgreSQL: 실제 쓰기 차단 / MySQL: 힌트만

[3] Hibernate Session readOnly 힌트
  session.setDefaultReadOnly(true)
  → 로드 엔티티를 readOnly로 마킹
  → loadedState(스냅샷) 최소화

리플리카 라우팅 구조
  LazyConnectionDataSourceProxy (Connection 획득 지연)
    → ReplicationRoutingDataSource
      → isCurrentTransactionReadOnly() 확인
      → true: replica / false: master

SimpleJpaRepository 설계 원칙
  @Transactional(readOnly = true) 클래스 레벨 (조회 기본값)
  @Transactional 메서드 레벨 (쓰기 메서드 오버라이드)
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional(readOnly = true)` 서비스 메서드 안에서 두 개의 Repository 조회 메서드를 호출한다. 각 Repository 메서드에는 `@Transactional(readOnly = true)`가 이미 붙어있다. 이때 실제로 몇 개의 트랜잭션이 열리고, 영속성 컨텍스트는 몇 개 생성되는가?

**Q2.** `@Transactional(readOnly = true)` 메서드에서 `OSIV(Open Session In View)`가 활성화된 상태로 컨트롤러까지 Lazy Loading을 시도한다면, readOnly 힌트가 OSIV 세션에도 적용되는가?

**Q3.** `LazyConnectionDataSourceProxy`를 사용하지 않고 일반 `AbstractRoutingDataSource`만으로 readOnly 라우팅을 구현할 때 발생하는 문제는 무엇인가?

> 💡 **해설**
>
> **Q1.** 트랜잭션 1개, 영속성 컨텍스트 1개만 생성된다. 서비스 메서드가 `@Transactional(readOnly = true)`로 트랜잭션을 시작하면 ThreadLocal에 트랜잭션 컨텍스트가 등록된다. Repository 메서드의 `@Transactional(readOnly = true)`는 기본 Propagation인 `REQUIRED`이므로 이미 존재하는 트랜잭션에 참여한다(`isNewTransaction() = false`). 따라서 영속성 컨텍스트도 서비스 레벨에서 하나만 생성되어 두 Repository 조회에서 공유된다. 이것이 서비스 레벨 `@Transactional`을 두는 중요한 이유 중 하나다 — 1차 캐시 공유로 동일 엔티티를 중복 조회하지 않는다.
>
> **Q2.** OSIV(Open Session In View, `spring.jpa.open-in-view=true`)가 활성화되면 요청 시작 시 `OpenEntityManagerInViewInterceptor`가 EntityManager를 열고 요청 종료 시 닫는다. 이 EntityManager는 서비스 `@Transactional(readOnly = true)` 트랜잭션이 끝난 후에도 열려있어 Lazy Loading이 가능하다. 그러나 트랜잭션이 종료된 후 OSIV 세션은 readOnly 힌트가 해제된 상태로 동작한다. OSIV 세션 자체에 `FlushMode.MANUAL`은 유지될 수 있지만 readOnly Connection 설정은 트랜잭션 범위에서만 유효하다. 따라서 OSIV 구간의 Lazy Loading은 새 Connection(readOnly 힌트 없음)을 사용할 수 있으며 리플리카 라우팅이 적용되지 않을 수 있다.
>
> **Q3.** `AbstractRoutingDataSource`는 `getConnection()` 호출 시점에 `determineCurrentLookupKey()`를 실행해 어떤 DataSource를 사용할지 결정한다. `JpaTransactionManager.doBegin()`에서 Connection을 가져올 때 이미 `TransactionSynchronizationManager`에 readOnly 여부가 등록되기 전일 수 있다. 구체적으로: `doBegin()`이 시작하면서 Connection을 먼저 획득하고, 이후 `DataSourceUtils.prepareConnectionForTransaction()`에서 readOnly를 설정하는 순서이기 때문에 `determineCurrentLookupKey()` 호출 시점에 `isCurrentTransactionReadOnly()`가 아직 `false`를 반환한다. 결과적으로 readOnly 트랜잭션도 항상 master DataSource를 사용하게 된다. `LazyConnectionDataSourceProxy`는 Connection 획득을 첫 SQL 실행 시까지 지연시켜 이 타이밍 문제를 해결한다.

---

<div align="center">

**[⬅️ 이전: private 메서드에 @Transactional이 안 되는 이유](./05-why-private-transactional-fails.md)** | **[다음: Rollback 규칙 — checked vs unchecked ➡️](./07-rollback-rules.md)**

</div>
