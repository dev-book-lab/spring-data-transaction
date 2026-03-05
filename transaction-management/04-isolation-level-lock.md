# Isolation Level과 Database Lock — 동시성 문제와 격리 수준의 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Dirty Read / Non-Repeatable Read / Phantom Read 세 가지 동시성 문제는 어떤 상황에서 발생하는가?
- `@Transactional(isolation = REPEATABLE_READ)`는 내부적으로 어떻게 JDBC에 전달되는가?
- MySQL InnoDB와 PostgreSQL의 기본 Isolation Level이 다른 이유는?
- Optimistic Lock(`@Version`)과 Pessimistic Lock(`LockModeType`)은 어떤 기준으로 선택하는가?
- `SELECT ... FOR UPDATE`는 Spring Data JPA에서 어떻게 사용하는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 여러 트랜잭션이 동시에 같은 데이터에 접근하면 이상 현상이 발생한다

```java
// 동시 실행 시나리오 — 재고 차감
// Thread 1 (트랜잭션 A): 재고 조회 → 차감
// Thread 2 (트랜잭션 B): 재고 조회 → 차감

// 재고 10개 있는 상태에서 A, B 동시에 5개씩 주문
// 최악의 경우: A조회(10) → B조회(10) → A차감(5저장) → B차감(5저장)
// 결과: 재고 5개 (10-5=5가 두 번 → 실제 재고 0이어야 하는데)

// 이런 동시성 문제를 해결하는 두 가지 축:
//   1. Isolation Level — "얼마나 다른 트랜잭션의 변경을 볼 수 있는가"
//   2. Lock — "다른 트랜잭션의 접근을 물리적으로 막는가"
```

```
Isolation이 필요한 이유:
  트랜잭션이 완전히 격리되면 (SERIALIZABLE) → 성능 저하
  완전히 열려있으면 (READ UNCOMMITTED) → 데이터 정합성 파괴
  → 성능과 정합성 사이의 트레이드오프를 수준으로 조절
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: Isolation Level을 높이면 Lock이 더 많이 걸린다

```
❌ 잘못된 이해:
  SERIALIZABLE → 모든 읽기에 Lock 걸림
  READ COMMITTED → Lock 거의 없음

✅ DB마다 구현 방식이 다름:

MySQL InnoDB:
  REPEATABLE_READ (기본): MVCC 기반 스냅샷 읽기 → 읽기 Lock 없음
  SERIALIZABLE: SELECT에 공유 Lock (S-Lock) 추가

PostgreSQL:
  READ COMMITTED (기본): MVCC 기반
  REPEATABLE_READ: MVCC 기반 스냅샷 → Lock 없음
  SERIALIZABLE: SSI(Serializable Snapshot Isolation) → Lock 최소화

→ MVCC 기반 DB에서는 Isolation Level을 올려도 읽기 Lock이 추가되지 않는 경우가 많음
```

### Before: @Transactional의 isolation 설정만으로 충분하다

```java
// ❌ Isolation Level로 해결 안 되는 경우
// Lost Update (갱신 손실): 두 트랜잭션이 같은 값을 읽고 각자 수정 → 하나가 덮어씀
// REPEATABLE_READ도 갱신 손실을 완전히 막지 못함 (MVCC 환경)

// ✅ Lost Update 해결은 Lock이 필요
// Optimistic Lock (@Version) → 충돌 감지 후 재시도
// Pessimistic Lock (SELECT FOR UPDATE) → 읽는 순간 배타 Lock
```

---

## ✨ 올바른 이해와 패턴

### After: 동시성 문제 유형과 해결 수단 매핑

```
동시성 문제           발생 조건                    해결 수단
─────────────────────────────────────────────────────────
Dirty Read            미커밋 데이터 읽기           READ COMMITTED 이상
Non-Repeatable Read   같은 행 두 번 읽기, 다른 값  REPEATABLE_READ 이상
Phantom Read          범위 조회 시 새 행 출현      SERIALIZABLE (또는 Gap Lock)
Lost Update           동시 수정 → 하나가 덮어씀   Optimistic/Pessimistic Lock
Deadlock              두 트랜잭션이 서로 Lock 대기  Lock 순서 통일, 타임아웃

선택 기준:
  충돌이 드문 경우   → Optimistic Lock (@Version)
  충돌이 잦은 경우   → Pessimistic Lock (SELECT FOR UPDATE)
  읽기 일관성 필요   → REPEATABLE_READ
  완전 직렬화 필요   → SERIALIZABLE (성능 저하 감수)
```

---

## 🔬 내부 동작 원리 — Isolation Level 설정 소스 추적

### 1. @Transactional isolation → JDBC 설정 경로

```java
// Spring이 Isolation Level을 JDBC에 전달하는 경로

// [1] @Transactional 파싱
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void process() { ... }
// → TransactionAttribute.getIsolationLevel() = Connection.TRANSACTION_REPEATABLE_READ (4)

// [2] AbstractPlatformTransactionManager.startTransaction()
//     → doBegin() 호출

// [3] DataSourceTransactionManager.doBegin()
protected void doBegin(Object transaction, TransactionDefinition definition) {
    Connection con = obtainDataSource().getConnection();

    // Isolation Level 설정
    Integer previousIsolationLevel = null;
    if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
        previousIsolationLevel = con.getTransactionIsolation();
        con.setTransactionIsolation(definition.getIsolationLevel());
        // → JDBC: con.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ)
        // → DB로 전송: SET TRANSACTION ISOLATION LEVEL REPEATABLE READ
    }
    // 트랜잭션 종료 시 이전 Isolation Level로 복원 (Connection 재사용 대비)
}

// [4] JpaTransactionManager.doBegin()
protected void doBegin(Object transaction, TransactionDefinition definition) {
    // ...
    // DataSourceUtils.prepareConnectionForTransaction()을 통해 동일하게 처리
    Integer previousIsolationLevel =
        DataSourceUtils.prepareConnectionForTransaction(con, definition);
    // → con.setTransactionIsolation() 호출
}
```

### 2. 네 가지 Isolation Level과 동시성 문제

#### READ UNCOMMITTED

```sql
-- 트랜잭션 A (커밋 전)
UPDATE accounts SET balance = 0 WHERE id = 1;
-- (아직 커밋 안 함)

-- 트랜잭션 B (READ UNCOMMITTED)
SELECT balance FROM accounts WHERE id = 1;
-- 결과: 0  ← Dirty Read! 커밋되지 않은 데이터 읽음

-- 트랜잭션 A가 이후 ROLLBACK하면
-- 트랜잭션 B는 실제로 존재하지 않는 데이터를 읽은 것
```

```java
// Spring에서 READ_UNCOMMITTED 설정
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
// 실무: 거의 사용 안 함 (데이터 정합성 포기)
// 예외: 정확도보다 속도가 중요한 대시보드, 통계 (근사값으로 충분)
```

#### READ COMMITTED (PostgreSQL 기본값)

```sql
-- 트랜잭션 A
UPDATE accounts SET balance = 500 WHERE id = 1;  -- balance: 1000 → 500
COMMIT;

-- 트랜잭션 B (READ COMMITTED)
-- 첫 번째 읽기 (A 커밋 전)
SELECT balance FROM accounts WHERE id = 1;  -- 결과: 1000

-- A가 커밋 후 B의 두 번째 읽기
SELECT balance FROM accounts WHERE id = 1;  -- 결과: 500 ← Non-Repeatable Read!
-- 같은 트랜잭션에서 같은 행을 읽었는데 값이 다름
```

```java
// 실무: 대부분의 OLTP 시스템에서 충분
// Dirty Read는 없지만 Non-Repeatable Read는 허용
```

#### REPEATABLE READ (MySQL InnoDB 기본값)

```sql
-- 트랜잭션 B가 시작 시점의 스냅샷을 유지 (MVCC)
-- 트랜잭션 A가 커밋해도 B는 자신의 스냅샷에서 읽음

-- 트랜잭션 A: balance 1000 → 500 으로 UPDATE + COMMIT
-- 트랜잭션 B (REPEATABLE READ):
SELECT balance FROM accounts WHERE id = 1;  -- 1000 (스냅샷 유지)
-- A가 커밋 후에도:
SELECT balance FROM accounts WHERE id = 1;  -- 여전히 1000 (Non-Repeatable Read 없음)

-- 단, Phantom Read는 가능:
-- 트랜잭션 A가 새 행 INSERT + COMMIT
SELECT COUNT(*) FROM accounts WHERE balance > 0;  -- 처음: 5, 나중: 6 (Phantom!)
-- MySQL InnoDB: Gap Lock으로 Phantom Read도 방지 (REPEATABLE_READ에서도)
```

#### SERIALIZABLE

```sql
-- 모든 트랜잭션이 순서대로 실행된 것처럼 보임
-- MySQL: 모든 SELECT에 S-Lock 추가
-- PostgreSQL: SSI 알고리즘으로 직렬화 가능성 추적

-- 성능 영향:
--   Lock 경합 증가 → 처리량 감소
--   Deadlock 가능성 증가
-- 사용 사례: 금융 결산, 재고 정합성이 절대적으로 중요한 경우
```

### 3. DB별 기본 Isolation Level 차이

```
MySQL InnoDB:
  기본: REPEATABLE READ
  이유: 바이너리 로그 기반 복제 시 일관성 보장
       Statement 기반 복제에서 READ COMMITTED는 비결정적 결과 가능
  Gap Lock으로 Phantom Read도 기본적으로 방지

PostgreSQL:
  기본: READ COMMITTED
  이유: MVCC 구현이 충분히 강력하여 READ COMMITTED도 실용적
       REPEATABLE READ부터 트랜잭션 시작 시 스냅샷 고정
       SSI로 SERIALIZABLE 오버헤드 최소화

Oracle:
  기본: READ COMMITTED
  MVCC 기반, REPEATABLE READ 미지원 (SERIALIZABLE로 건너뜀)

Spring 기본값:
  ISOLATION_DEFAULT (-1) → DB의 기본 Isolation Level 사용
  별도 설정 없으면 DB 기본값 그대로
```

### 4. Optimistic Lock — @Version

```java
// @Version — 낙관적 락 구현
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;

    private int stock;

    @Version
    private Long version;  // 수정 시마다 자동 증가
}

// 동작 원리:
// 조회: SELECT id, stock, version FROM products WHERE id = 1
//       → version = 5

// 수정 (UPDATE):
// UPDATE products SET stock = ?, version = 6
//   WHERE id = 1 AND version = 5
//       ↑ version 조건 포함

// 다른 트랜잭션이 먼저 수정하면 version = 6이 됨
// 나의 UPDATE: WHERE id=1 AND version=5 → 0 rows affected
// → OptimisticLockException 발생 → 재시도 또는 실패 처리

// Spring Data JPA에서 OptimisticLockException 처리
@Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
@Transactional
public void updateStock(Long productId, int quantity) {
    Product product = productRepository.findById(productId).orElseThrow();
    product.decreaseStock(quantity);
    // save() 호출 또는 dirty checking으로 UPDATE 실행
    // 충돌 시 OptimisticLockingFailureException → @Retryable로 재시도
}
```

### 5. Pessimistic Lock — SELECT FOR UPDATE

```java
// Spring Data JPA에서 비관적 락 사용
public interface ProductRepository extends JpaRepository<Product, Long> {

    // @Lock 어노테이션으로 락 유형 지정
    @Lock(LockModeType.PESSIMISTIC_WRITE)  // SELECT ... FOR UPDATE
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);

    // PESSIMISTIC_READ: SELECT ... FOR SHARE (공유 락, 읽기만 허용)
    @Lock(LockModeType.PESSIMISTIC_READ)
    Optional<Product> findById(Long id);  // Query Method에도 적용 가능
}

// Lock 타임아웃 설정
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "javax.persistence.lock.timeout", value = "5000"))
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithLockAndTimeout(@Param("id") Long id);

// 사용 예시
@Transactional
public void decreaseStock(Long productId, int quantity) {
    Product product = productRepository.findByIdWithLock(productId)
        .orElseThrow();
    // SELECT ... FOR UPDATE → 이 행에 대한 배타 락 획득
    // 다른 트랜잭션은 이 행 수정 불가 (대기)

    product.decreaseStock(quantity);
    // dirty checking → UPDATE 실행, 트랜잭션 종료 시 락 해제
}
```

### 6. Deadlock 발생 패턴과 방지

```
Deadlock 발생 예시:
  트랜잭션 A: Lock(Product 1) → Lock(Product 2) 시도
  트랜잭션 B: Lock(Product 2) → Lock(Product 1) 시도
  → A는 B가 Product 2 해제 대기
  → B는 A가 Product 1 해제 대기
  → 무한 대기 → DB가 하나를 Deadlock victim으로 선택 → 롤백

방지 전략:
  1. 항상 같은 순서로 Lock 획득 (Product ID 오름차순)
  2. 트랜잭션을 짧게 유지 (Lock 점유 시간 최소화)
  3. Lock 타임아웃 설정 (무한 대기 방지)
  4. Optimistic Lock 우선 고려 (Lock 없음)

Spring에서 Deadlock 감지:
  CannotAcquireLockException (Spring) ← DataAccessException 하위
  → PlatformTransactionManager가 DB 예외를 Spring 예외로 변환
  → @Retryable로 재시도 구현 가능
```

---

## 💻 실험으로 확인하기

### 실험 1: Non-Repeatable Read 재현 (READ COMMITTED)

```java
@Test
void nonRepeatableRead() throws InterruptedException {
    // 초기 데이터
    Product product = productRepository.save(new Product("A", 100));

    CountDownLatch latch = new CountDownLatch(1);

    // 트랜잭션 B: READ COMMITTED로 같은 데이터 두 번 읽기
    new Thread(() -> {
        transactionTemplate.execute(status -> {
            // 첫 번째 읽기
            Product p1 = productRepository.findById(product.getId()).get();
            System.out.println("첫 번째 읽기: " + p1.getStock()); // 100

            // 트랜잭션 A가 수정하도록 대기
            latch.await(1, TimeUnit.SECONDS);

            // 두 번째 읽기
            Product p2 = productRepository.findById(product.getId()).get();
            System.out.println("두 번째 읽기: " + p2.getStock()); // 50 (Non-Repeatable Read)
            return null;
        });
    }).start();

    // 트랜잭션 A: 수정 + 커밋
    Thread.sleep(100);
    product.setStock(50);
    productRepository.save(product);
    latch.countDown();
}
```

### 실험 2: @Version으로 충돌 감지

```java
@Test
void optimisticLockConflict() {
    Product product = productRepository.save(new Product("B", 10));

    // 두 트랜잭션이 같은 엔티티를 조회
    Product p1 = productRepository.findById(product.getId()).get(); // version=0
    Product p2 = productRepository.findById(product.getId()).get(); // version=0

    // 트랜잭션 1이 먼저 수정
    p1.setStock(5);
    productRepository.save(p1); // UPDATE ... WHERE version=0 → version=1

    // 트랜잭션 2가 이후 수정 시도
    p2.setStock(8);
    assertThrows(OptimisticLockingFailureException.class, () -> {
        productRepository.save(p2); // UPDATE ... WHERE version=0 → 0 rows → 예외
    });
}
```

### 실험 3: Isolation Level 확인

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
@Test
void checkIsolationLevel() {
    // 현재 트랜잭션의 Isolation Level 확인
    TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
    // → 4 (TRANSACTION_REPEATABLE_READ)

    // 실제 JDBC Connection의 Isolation Level
    DataSource ds = applicationContext.getBean(DataSource.class);
    Connection con = DataSourceUtils.getConnection(ds);
    System.out.println("JDBC Isolation: " + con.getTransactionIsolation());
    // → 4 (Connection.TRANSACTION_REPEATABLE_READ)
}
```

---

## ⚡ 성능 임팩트

```
Isolation Level 성능 영향:

READ UNCOMMITTED:
  Lock 없음 → 최고 성능
  데이터 정합성 없음 → 실무 부적합

READ COMMITTED:
  MVCC 기반 DB (MySQL, PostgreSQL): 읽기 Lock 없음
  쓰기만 Row-Level Lock
  대부분의 OLTP에 적합

REPEATABLE READ:
  MVCC DB: 스냅샷 유지 → 읽기 Lock 없음 (MySQL, PostgreSQL)
  비MVCC DB: 공유 Lock → 처리량 감소
  MySQL: Gap Lock으로 Phantom Read 방지 → Index 없는 범위 쿼리 주의

SERIALIZABLE:
  MySQL: 모든 읽기에 S-Lock → 쓰기 트랜잭션과 경합
  PostgreSQL SSI: 직렬화 충돌 감지 시 abort → 재시도 필요
  처리량: REPEATABLE_READ 대비 20~40% 감소 (워크로드 의존)

Optimistic vs Pessimistic Lock:
  Optimistic: 충돌 없을 때 오버헤드 거의 없음 (version 컬럼 UPDATE만)
              충돌 많으면 재시도 폭증 → Pessimistic보다 느릴 수 있음
  Pessimistic: 항상 Lock 획득 비용 + 대기 가능
               충돌 많은 환경에서 오히려 효율적 (재시도 없음)
               Lock 시간 동안 다른 트랜잭션 블로킹

선택 기준:
  충돌 빈도 < 10% → Optimistic Lock
  충돌 빈도 > 50% → Pessimistic Lock
  중간 → 측정 후 결정
```

---

## 🤔 트레이드오프

```
Isolation Level 선택:
  READ COMMITTED:
    장점: 높은 동시성, 대부분 충분
    단점: Non-Repeatable Read, Phantom Read 가능
    → 일반 OLTP, 조회 중심 서비스

  REPEATABLE READ:
    장점: Non-Repeatable Read 없음
    단점: 장기 트랜잭션 시 스냅샷 유지 → 메모리 사용, Undo Log 증가
    → 정합성이 중요한 조회 트랜잭션

  SERIALIZABLE:
    장점: 완전한 직렬화 보장
    단점: 성능 저하, Deadlock/충돌 증가
    → 금융 결산 등 극히 제한적

Lock 전략:
  @Version (Optimistic):
    장점: Lock 없음, 충돌 없을 때 오버헤드 최소
    단점: 충돌 시 예외 → 재시도 로직 필요, 사용자 경험 저하
    → 읽기 多, 쓰기 少, 충돌 드문 환경

  PESSIMISTIC_WRITE (FOR UPDATE):
    장점: 확실한 배타 제어, 재시도 불필요
    단점: Lock 대기, Deadlock 위험, 처리량 감소
    → 재고, 좌석, 포인트 등 충돌이 잦은 경쟁 자원

  락 없는 설계 (CQRS + Event Sourcing):
    이벤트 로그로 충돌 감지 → 최종 일관성
    → 성능 최우선, 강한 일관성 포기 가능한 경우
```

---

## 📌 핵심 정리

```
Isolation Level → JDBC 설정 경로
  @Transactional(isolation=...) → TransactionAttribute.getIsolationLevel()
  → doBegin() → Connection.setTransactionIsolation()
  → DB로 전송: SET TRANSACTION ISOLATION LEVEL ...

4가지 Isolation Level 허용 이상현상
  READ UNCOMMITTED:  Dirty Read ✓, Non-Repeatable ✓, Phantom ✓
  READ COMMITTED:    Dirty Read ✗, Non-Repeatable ✓, Phantom ✓
  REPEATABLE READ:   Dirty Read ✗, Non-Repeatable ✗, Phantom ✓ (MySQL은 ✗)
  SERIALIZABLE:      모두 ✗

DB별 기본값
  MySQL InnoDB:  REPEATABLE READ (Gap Lock으로 Phantom Read도 방지)
  PostgreSQL:    READ COMMITTED (MVCC 기반, SSI로 SERIALIZABLE 효율화)
  Spring 기본:   ISOLATION_DEFAULT → DB 기본값 그대로

Lock 선택
  @Version:                 충돌 드문 경우, 재시도 가능한 경우
  @Lock(PESSIMISTIC_WRITE): 충돌 잦은 경우, 재시도 불가한 경우
  lock.timeout 설정:        Pessimistic Lock 사용 시 항상 필수
```

---

## 🤔 생각해볼 문제

**Q1.** MySQL에서 `REPEATABLE READ` Isolation Level 하에서 `UPDATE products SET stock = stock - 1 WHERE id = 1 AND stock > 0`를 두 트랜잭션이 동시에 실행하면 어떤 결과가 나오는가? `@Version`을 사용하는 경우와 비교하면?

**Q2.** `@Lock(LockModeType.PESSIMISTIC_WRITE)`로 조회한 후 트랜잭션이 길어질 때(예: 외부 API 호출 포함), Lock이 점유된 상태로 유지되는 문제를 어떻게 해결하는가?

**Q3.** Spring의 `@Transactional(isolation = REPEATABLE_READ)`를 HikariCP와 함께 사용할 때, Connection Pool에서 꺼낸 Connection의 Isolation Level을 변경하고 트랜잭션 종료 후 복원하지 않으면 어떤 문제가 발생하는가?

> 💡 **해설**
>
> **Q1.** MySQL `REPEATABLE READ`에서 `UPDATE ... WHERE stock > 0`는 **실제 최신 값**을 기준으로 동작한다. `SELECT`와 달리 `UPDATE`는 현재 커밋된 값(current read)을 읽는다. 두 트랜잭션이 동시에 실행되면 하나가 먼저 행 Lock을 획득하고 `stock = 0`으로 UPDATE한다. 나머지 트랜잭션은 Lock 대기 후 재시도 시 `stock = 0`을 확인하고 `WHERE stock > 0` 조건 불만족으로 0 rows를 반환한다. 즉 재고가 음수가 되지 않는다. `@Version`을 사용하면 두 트랜잭션 모두 SELECT 시 같은 version을 읽고, 나중에 UPDATE하는 쪽이 `WHERE version = 0` 조건 실패로 `OptimisticLockingFailureException`이 발생한다. 재시도 로직이 있다면 재시도 시 stock=0을 확인하고 비즈니스 예외를 던지면 된다.
>
> **Q2.** 핵심 원칙은 Lock 점유 시간을 최소화하는 것이다. 외부 API 호출은 트랜잭션 **밖**에서 실행해야 한다. 패턴: (1) 외부 API 호출 (트랜잭션 없음), (2) 결과를 가지고 트랜잭션 시작 → Lock 획득 → 데이터 수정 → 즉시 커밋. 트랜잭션 분리가 어려우면 `REQUIRES_NEW`로 Lock 구간을 별도 짧은 트랜잭션으로 분리한다. 또한 `lock.timeout`을 반드시 설정하여 Lock 대기 시간에 상한을 두고, 회로 차단기(Circuit Breaker)로 외부 API 실패 시 빠르게 실패 처리하여 Lock 점유 시간을 제한한다.
>
> **Q3.** HikariCP는 Connection을 Pool에 반환할 때 기본적으로 `rollback()`을 호출하여 트랜잭션 상태를 초기화한다. 하지만 Isolation Level은 Connection 레벨 설정이므로 rollback으로 복원되지 않는다. Spring의 `DataSourceUtils.resetConnectionAfterTransaction()`이 트랜잭션 종료 시 이전 Isolation Level로 복원(`con.setTransactionIsolation(previousIsolationLevel)`)하는 역할을 한다. 이 복원이 없으면 Pool에서 꺼낸 Connection이 이전 트랜잭션에서 설정된 높은 Isolation Level을 유지한 채로 다른 요청에서 사용되어, 의도치 않은 격리 수준으로 동작하는 버그가 발생한다. Spring의 `AbstractPlatformTransactionManager`는 이를 `doCleanupAfterCompletion()`에서 처리한다.

---

<div align="center">

**[⬅️ 이전: Propagation 7가지 완전 분석](./03-propagation-seven-types.md)** | **[다음: private 메서드에 @Transactional이 안 되는 이유 ➡️](./05-why-private-transactional-fails.md)**

</div>
