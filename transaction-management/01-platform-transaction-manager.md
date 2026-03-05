# PlatformTransactionManager 구조와 구현체 — 트랜잭션 추상화의 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `PlatformTransactionManager`가 존재하는 이유 — JPA, JDBC, JTA를 왜 동일한 인터페이스로 추상화하는가?
- `JpaTransactionManager`와 `DataSourceTransactionManager`의 내부 동작 차이는?
- `TransactionStatus` 객체는 어떤 정보를 담고 있으며 어떻게 사용되는가?
- `@Transactional`이 결국 어느 시점에 `PlatformTransactionManager`를 호출하는가?
- 멀티 데이터소스 환경에서 트랜잭션 매니저를 두 개 쓸 때 어떤 문제가 생기는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 트랜잭션 API가 기술마다 전부 다르다

```java
// JDBC 직접 사용
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false);
try {
    // 작업
    conn.commit();
} catch (Exception e) {
    conn.rollback();
} finally {
    conn.close();
}

// Hibernate 직접 사용
Session session = sessionFactory.openSession();
Transaction tx = session.beginTransaction();
try {
    // 작업
    tx.commit();
} catch (Exception e) {
    tx.rollback();
} finally {
    session.close();
}

// JTA (분산 트랜잭션)
UserTransaction utx = ctx.lookup("java:comp/UserTransaction");
utx.begin();
try {
    // 작업
    utx.commit();
} catch (Exception e) {
    utx.rollback();
}
```

```
문제:
  기술마다 트랜잭션 시작/커밋/롤백 API가 다름
  비즈니스 코드가 특정 영속성 기술에 종속
  JPA → JTA 전환 시 코드 전면 수정

Spring의 해결:
  PlatformTransactionManager 인터페이스로 트랜잭션 관리 추상화
  → 비즈니스 코드는 인터페이스만 의존
  → 구현체(JpaTransactionManager, DataSourceTransactionManager, JtaTransactionManager) 교체 가능
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: JPA를 쓰면 JpaTransactionManager, JDBC를 쓰면 DataSourceTransactionManager만 쓴다

```java
// ❌ 잘못된 이해: JPA와 JdbcTemplate을 같이 쓸 때 매니저를 두 개 등록
@Bean
public JpaTransactionManager jpaTransactionManager(...) { ... }

@Bean
public DataSourceTransactionManager jdbcTransactionManager(...) { ... }
// → 두 매니저가 각자 별개의 Connection을 사용 → 같은 트랜잭션에 묶이지 않음!

// ✅ JPA + JdbcTemplate 혼용 시 JpaTransactionManager 하나만 등록
// JpaTransactionManager는 내부적으로 DataSource도 관리함
// → EntityManager와 JdbcTemplate이 같은 Connection을 공유
@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
    return new JpaTransactionManager(emf);
    // JdbcTemplate은 DataSourceUtils를 통해 동일 Connection 재사용
}
```

### Before: TransactionManager를 직접 호출해서 트랜잭션을 제어해야 한다

```java
// ❌ 불필요한 직접 호출 (선언적 방식이 있는데 프로그래밍 방식 사용)
@Autowired
PlatformTransactionManager txManager;

public void doSomething() {
    TransactionStatus status = txManager.getTransaction(new DefaultTransactionDefinition());
    try {
        // 작업
        txManager.commit(status);
    } catch (Exception e) {
        txManager.rollback(status);
    }
}

// ✅ @Transactional 선언적 방식이 대부분의 경우 더 적합
// 프로그래밍 방식이 필요한 경우: 조건부 트랜잭션, 루프 내 개별 트랜잭션
// → TransactionTemplate 사용 권장
```

---

## ✨ 올바른 이해와 패턴

### After: 계층 구조와 각 구현체의 역할 파악

```
PlatformTransactionManager (인터페이스)
├── AbstractPlatformTransactionManager (추상 클래스 — Propagation 처리 로직 포함)
│   ├── JpaTransactionManager
│   │     EntityManagerFactory 관리, JPA + JDBC 통합 Connection 공유
│   ├── DataSourceTransactionManager
│   │     JDBC 전용, JdbcTemplate / MyBatis 환경
│   └── JtaTransactionManager
│         분산 트랜잭션 (XA), 여러 DB / MQ 를 하나의 트랜잭션으로
│
ReactiveTransactionManager (인터페이스 — WebFlux/R2DBC용, 별도 계층)
└── R2dbcTransactionManager
```

---

## 🔬 내부 동작 원리 — PlatformTransactionManager 소스 추적

### 1. PlatformTransactionManager 인터페이스

```java
// Spring의 트랜잭션 추상화 핵심 인터페이스
public interface PlatformTransactionManager extends TransactionManager {

    // 트랜잭션 시작 또는 기존 트랜잭션 참여
    // TransactionDefinition: Propagation, Isolation, Timeout, readOnly 정보 포함
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException;

    // 커밋
    void commit(TransactionStatus status) throws TransactionException;

    // 롤백
    void rollback(TransactionStatus status) throws TransactionException;
}

// TransactionDefinition — 트랜잭션 속성 명세
public interface TransactionDefinition {
    int PROPAGATION_REQUIRED     = 0;   // 기본값
    int PROPAGATION_SUPPORTS     = 1;
    int PROPAGATION_MANDATORY    = 2;
    int PROPAGATION_REQUIRES_NEW = 3;
    int PROPAGATION_NOT_SUPPORTED= 4;
    int PROPAGATION_NEVER        = 5;
    int PROPAGATION_NESTED       = 6;

    int ISOLATION_DEFAULT        = -1;  // DB 기본값 사용
    int ISOLATION_READ_UNCOMMITTED = 1;
    int ISOLATION_READ_COMMITTED   = 2;
    int ISOLATION_REPEATABLE_READ  = 4;
    int ISOLATION_SERIALIZABLE     = 8;

    int getPropagationBehavior();
    int getIsolationLevel();
    int getTimeout();
    boolean isReadOnly();
}
```

### 2. TransactionStatus — 트랜잭션 실행 상태 객체

```java
// TransactionStatus — getTransaction() 반환값, 현재 트랜잭션 상태 표현
public interface TransactionStatus extends TransactionExecution, SavepointManager {

    boolean isNewTransaction();  // 새 트랜잭션인가? false면 기존 참여
    boolean hasSavepoint();      // Savepoint가 있는가? (NESTED에서 사용)
    void setRollbackOnly();      // 롤백만 가능하도록 마킹
    boolean isRollbackOnly();    // 롤백 전용 상태인가?
    boolean isCompleted();       // 커밋/롤백 완료됐는가?
    void flush();                // 영속성 컨텍스트 flush (JPA 전용)
}

// DefaultTransactionStatus — 실제 구현체
class DefaultTransactionStatus extends AbstractTransactionStatus {

    @Nullable
    private final Object transaction;  // 실제 트랜잭션 객체 (Connection 등)

    private final boolean newTransaction;   // 새 트랜잭션 여부
    private final boolean newSynchronization; // ThreadLocal 동기화 등록 여부
    private final boolean readOnly;
    private final boolean debug;

    @Nullable
    private Object suspendedResources;  // REQUIRES_NEW 시 일시정지된 이전 트랜잭션
}
```

### 3. AbstractPlatformTransactionManager — Propagation 처리 핵심

```java
// AbstractPlatformTransactionManager — 모든 구현체가 상속하는 추상 클래스
// Propagation 분기, Synchronization 관리, 예외 처리를 여기서 담당
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager {

    @Override
    public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) {

        // 현재 스레드에 진행 중인 트랜잭션이 있는지 확인
        Object transaction = doGetTransaction();  // 구현체별로 다름

        if (isExistingTransaction(transaction)) {
            // 기존 트랜잭션 존재 → Propagation에 따라 분기
            return handleExistingTransaction(definition, transaction, debugEnabled);
        }

        // 기존 트랜잭션 없음 → REQUIRED, REQUIRES_NEW → 새 트랜잭션 시작
        //                    SUPPORTS, NOT_SUPPORTED, NEVER → 트랜잭션 없이 실행
        //                    MANDATORY → 예외 발생
        if (def.getPropagationBehavior() == PROPAGATION_MANDATORY) {
            throw new IllegalTransactionStateException("No existing transaction found");
        }

        if (def.getPropagationBehavior() == PROPAGATION_REQUIRED
                || def.getPropagationBehavior() == PROPAGATION_REQUIRES_NEW
                || def.getPropagationBehavior() == PROPAGATION_NESTED) {
            // 새 트랜잭션 시작
            SuspendedResourcesHolder suspendedResources = suspend(null);
            try {
                return startTransaction(def, transaction, debugEnabled, suspendedResources);
            } catch (...) { ... }
        }

        // SUPPORTS, NOT_SUPPORTED, NEVER: 트랜잭션 없이 실행
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }

    // 커밋 — rollbackOnly 체크 포함
    @Override
    public final void commit(TransactionStatus status) {

        if (status.isCompleted()) throw new IllegalTransactionStateException("already completed");

        DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;

        if (defStatus.isLocalRollbackOnly()) {
            processRollback(defStatus, false);  // rollbackOnly면 롤백
            return;
        }

        // 전역 rollbackOnly(내부 트랜잭션이 마킹) 체크
        if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
            processRollback(defStatus, true);
            // → UnexpectedRollbackException 발생
            return;
        }

        processCommit(defStatus);
    }
}
```

### 4. JpaTransactionManager — JPA 환경에서의 동작

```java
// JpaTransactionManager — EntityManagerFactory 기반 트랜잭션 관리
public class JpaTransactionManager extends AbstractPlatformTransactionManager
        implements ResourceTransactionManager, BeanFactoryAware, InitializingBean {

    private EntityManagerFactory entityManagerFactory;

    // 현재 스레드의 트랜잭션 객체 조회
    @Override
    protected Object doGetTransaction() {
        JpaTransactionObject txObject = new JpaTransactionObject();

        // ThreadLocal에서 현재 EntityManagerHolder 조회
        EntityManagerHolder emHolder = (EntityManagerHolder)
            TransactionSynchronizationManager.getResource(obtainEntityManagerFactory());

        if (emHolder != null) {
            txObject.setEntityManagerHolder(emHolder, false);
        }

        // DataSource Connection도 ThreadLocal에서 조회 (JdbcTemplate 연동)
        if (getDataSource() != null) {
            ConnectionHolder conHolder = (ConnectionHolder)
                TransactionSynchronizationManager.getResource(getDataSource());
            txObject.setConnectionHolder(conHolder);
        }

        return txObject;
    }

    // 새 트랜잭션 시작
    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {

        JpaTransactionObject txObject = (JpaTransactionObject) transaction;

        // 1. EntityManager 생성 (또는 기존 것 재사용)
        EntityManager em = createEntityManagerForTransaction();

        // 2. EntityManager를 ThreadLocal에 등록
        //    → 이후 @PersistenceContext 주입된 EntityManager 프록시가 이것을 사용
        TransactionSynchronizationManager.bindResource(
            obtainEntityManagerFactory(), new EntityManagerHolder(em));

        // 3. JDBC Connection 획득 및 설정
        Connection con = em.unwrap(Connection.class); // Hibernate → JDBC Connection
        Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        // → Isolation Level, readOnly 설정

        // 4. Connection도 ThreadLocal에 등록
        //    → JdbcTemplate이 동일 Connection 사용 가능
        if (getDataSource() != null) {
            TransactionSynchronizationManager.bindResource(
                getDataSource(), new ConnectionHolder(con));
        }

        txObject.getEntityManagerHolder().setSynchronizedWithTransaction(true);
    }

    @Override
    protected void doCommit(DefaultTransactionStatus status) {
        JpaTransactionObject txObject = (JpaTransactionObject) status.getTransaction();
        EntityManager em = txObject.getEntityManagerHolder().getEntityManager();

        em.getTransaction().commit();
        // → Hibernate: flush() 후 JDBC commit()
    }
}
```

### 5. DataSourceTransactionManager — JDBC 전용 동작

```java
// DataSourceTransactionManager — JDBC Connection 직접 관리
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager {

    @Override
    protected void doBegin(Object transaction, TransactionDefinition definition) {

        DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

        // Connection 획득
        Connection con = obtainDataSource().getConnection();

        // autoCommit 해제 (트랜잭션 시작)
        if (con.getAutoCommit()) {
            con.setAutoCommit(false);
        }

        // readOnly, Isolation 설정
        if (definition.isReadOnly()) {
            con.setReadOnly(true);
        }
        if (definition.getIsolationLevel() != ISOLATION_DEFAULT) {
            con.setTransactionIsolation(definition.getIsolationLevel());
        }

        // ThreadLocal에 Connection 등록
        TransactionSynchronizationManager.bindResource(obtainDataSource(),
            new ConnectionHolder(con));
    }

    @Override
    protected void doCommit(DefaultTransactionStatus status) {
        DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
        Connection con = txObject.getConnectionHolder().getConnection();
        con.commit();
    }
}
```

### 6. TransactionSynchronizationManager — ThreadLocal 기반 자원 관리

```java
// TransactionSynchronizationManager — 트랜잭션 컨텍스트의 ThreadLocal 저장소
public abstract class TransactionSynchronizationManager {

    // 핵심 ThreadLocal 필드들
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");
    // key: DataSource 또는 EntityManagerFactory
    // value: ConnectionHolder 또는 EntityManagerHolder

    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
        new NamedThreadLocal<>("Transaction synchronizations");
    // 커밋/롤백 후 콜백 등록

    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");

    private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");

    // 자원 바인딩 (트랜잭션 시작 시)
    public static void bindResource(Object key, Object value) {
        resources.get().put(key, value);
    }

    // 자원 조회 (JdbcTemplate, EntityManager 프록시가 사용)
    public static Object getResource(Object key) {
        return resources.get().get(key);
    }

    // 자원 해제 (트랜잭션 종료 시)
    public static Object unbindResource(Object key) {
        return resources.get().remove(key);
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: TransactionManager 구현체 타입 확인

```java
@Autowired
PlatformTransactionManager transactionManager;

@Test
void transactionManagerType() {
    System.out.println(transactionManager.getClass().getName());
    // JPA 환경: org.springframework.orm.jpa.JpaTransactionManager
    // JDBC만: org.springframework.jdbc.datasource.DataSourceTransactionManager
}
```

### 실험 2: ThreadLocal 자원 확인

```java
@Transactional
@Test
void threadLocalResources() {
    // 트랜잭션 시작 후 ThreadLocal에 바인딩된 자원 확인
    Map<Object, Object> resources =
        (Map<Object, Object>) ReflectionTestUtils.getField(
            TransactionSynchronizationManager.class, "resources");

    System.out.println("활성 트랜잭션: " +
        TransactionSynchronizationManager.isActualTransactionActive()); // true
    System.out.println("트랜잭션 이름: " +
        TransactionSynchronizationManager.getCurrentTransactionName());
    System.out.println("readOnly: " +
        TransactionSynchronizationManager.isCurrentTransactionReadOnly());
}
```

### 실험 3: 프로그래밍 방식 트랜잭션 — TransactionTemplate

```java
// TransactionTemplate — 프로그래밍 방식의 권장 패턴
@Autowired
TransactionTemplate transactionTemplate;

public void conditionalTransaction(boolean needsTx) {
    if (!needsTx) {
        doWork(); // 트랜잭션 없이 실행
        return;
    }

    transactionTemplate.execute(status -> {
        try {
            doWork();
            return null;
        } catch (Exception e) {
            status.setRollbackOnly();  // 명시적 롤백 마킹
            throw e;
        }
    });
}

// TransactionTemplate 설정
@Bean
public TransactionTemplate transactionTemplate(PlatformTransactionManager tm) {
    TransactionTemplate template = new TransactionTemplate(tm);
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    template.setTimeout(30);
    return template;
}
```

---

## ⚡ 성능 임팩트

```
트랜잭션 시작/종료 비용:

doBegin() 비용:
  ThreadLocal 조회/등록:        ~수백 ns
  Connection 획득 (Pool):       ~수 μs~수십 μs (대기 없을 시)
  setAutoCommit(false):         ~수십 μs (드라이버 호출)
  Isolation 설정:               ~수십 μs (변경 시에만)
  합계: 약 100μs~수ms

doCommit() 비용:
  Hibernate flush():            ~수십 μs~수ms (변경 건수에 비례)
  Connection.commit():          ~수ms (DB 왕복)
  ThreadLocal 정리:             ~수백 ns

총 트랜잭션 오버헤드: 수ms
→ DB 작업(쿼리) 비용이 수십ms라면 트랜잭션 오버헤드는 10% 미만

readOnly 트랜잭션 이점:
  FlushMode.MANUAL → flush() 생략 → ~수십 μs 절감
  스냅샷 비교 생략 → 엔티티 수에 비례한 비교 비용 절감
  일부 DB/드라이버: 읽기 전용 Connection → 리플리카 자동 라우팅
```

---

## 🤔 트레이드오프

```
JpaTransactionManager:
  JPA + JdbcTemplate 동시 사용 가능 (같은 Connection 공유)
  Hibernate Session 생명주기 관리
  JPA 없이 JDBC만 사용하면 과도한 의존성

DataSourceTransactionManager:
  JPA 없는 순수 JDBC/MyBatis 환경에 적합
  경량, 단순
  JPA와 함께 쓰면 Connection 공유 안 됨 → 데이터 불일치

JtaTransactionManager:
  여러 DataSource / MQ를 하나의 트랜잭션으로 묶을 수 있음
  XA 프로토콜 오버헤드 상당 (2PC)
  대부분의 웹 서비스에서는 과도함
  → Saga 패턴, Outbox 패턴으로 대체하는 추세

TransactionTemplate vs @Transactional:
  @Transactional: 선언적, 간결, AOP 프록시 제약 존재 (private 불가)
  TransactionTemplate: 프로그래밍 방식, 조건부/루프 내 트랜잭션 제어 가능
  → 대부분 @Transactional, 특수 상황 TransactionTemplate
```

---

## 📌 핵심 정리

```
PlatformTransactionManager 3메서드
  getTransaction(def)  → TransactionStatus (새 트랜잭션 or 기존 참여)
  commit(status)       → rollbackOnly 체크 후 커밋
  rollback(status)     → 롤백

계층 구조
  AbstractPlatformTransactionManager: Propagation 분기, Synchronization 관리
  JpaTransactionManager: EntityManager + JDBC Connection 통합 관리
  DataSourceTransactionManager: JDBC Connection만 관리

ThreadLocal 기반 자원 공유
  TransactionSynchronizationManager:
    bindResource(key, holder) → 트랜잭션 시작 시
    getResource(key)          → EntityManager/JdbcTemplate이 조회
    unbindResource(key)       → 트랜잭션 종료 시
  → 같은 트랜잭션 내에서 EntityManager와 JdbcTemplate이 동일 Connection 사용

JPA + JDBC 혼용 시
  JpaTransactionManager 하나만 등록 (DataSourceTransactionManager 불필요)
  → JpaTransactionManager가 DataSource Connection도 ThreadLocal에 등록
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional` 없이 `JdbcTemplate`을 사용하면 쿼리마다 새 Connection을 가져오고 반환하는가? Connection Pool에는 어떤 영향을 주는가?

**Q2.** `JpaTransactionManager`를 사용하는 환경에서 `JdbcTemplate`으로 INSERT를 실행하고 같은 트랜잭션에서 JPA로 해당 데이터를 조회하면 조회되는가? 그 이유는?

**Q3.** 두 개의 `DataSource`가 있는 멀티 데이터소스 환경에서 하나의 `@Transactional` 메서드 안에서 두 DB에 모두 쓰기를 했을 때, 하나가 실패하면 다른 하나도 롤백되는가?

> 💡 **해설**
>
> **Q1.** 트랜잭션 없이 `JdbcTemplate`을 사용하면 `DataSourceUtils.getConnection()`이 현재 ThreadLocal에 바인딩된 Connection이 없음을 확인하고, Pool에서 새 Connection을 가져온다. 쿼리 실행 후 `DataSourceUtils.releaseConnection()`이 즉시 Connection을 Pool로 반환한다. 이 방식은 쿼리마다 Connection 획득/반환을 반복하므로 Pool 경합이 증가하고, 여러 쿼리가 원자적으로 실행되지 않아 데이터 일관성도 보장되지 않는다. `@Transactional`을 붙이면 하나의 Connection이 메서드 전체에서 재사용된다.
>
> **Q2.** 조회된다. `JpaTransactionManager.doBegin()`은 EntityManager의 JDBC Connection을 가져와 DataSource 키로 ThreadLocal에도 등록한다. `JdbcTemplate`이 `DataSourceUtils.getConnection()`을 호출하면 같은 ThreadLocal에서 이미 바인딩된 Connection을 가져온다. 즉 JPA EntityManager와 JdbcTemplate이 동일 JDBC Connection을 공유한다. 단, JdbcTemplate INSERT는 영속성 컨텍스트를 거치지 않으므로 JPA 1차 캐시에는 없지만, 같은 Connection(같은 DB 세션)에서 조회하면 DB에서 직접 읽어온다.
>
> **Q3.** 롤백되지 않는다. 단일 `PlatformTransactionManager`는 하나의 DataSource만 관리한다. 두 DataSource에 각각 `DataSourceTransactionManager`를 등록하면 서로 독립적인 트랜잭션이 된다. `@Transactional`은 `PrimaryTransactionManager`로 지정된 하나의 매니저만 사용하므로 나머지 DataSource의 작업은 트랜잭션 밖에서 auto-commit으로 실행된다. 두 DB를 원자적으로 묶으려면 JTA(XA 트랜잭션)나, 보상 트랜잭션 기반의 Saga 패턴, 또는 Outbox 패턴을 사용해야 한다.

---

<div align="center">

**[다음: @Transactional 프록시 생성 메커니즘 ➡️](./02-transactional-proxy-mechanism.md)**

</div>
