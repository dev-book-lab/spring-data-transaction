# Propagation 7가지 완전 분석 — 물리 트랜잭션과 논리 트랜잭션의 구분

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 물리 트랜잭션과 논리 트랜잭션의 차이는 무엇이며, 왜 이 구분이 중요한가?
- `REQUIRES_NEW`가 기존 트랜잭션을 "일시 정지"하는 정확한 메커니즘은?
- `NESTED`가 `REQUIRES_NEW`와 다른 점 — Savepoint를 사용한다는 것의 의미는?
- 내부 트랜잭션이 롤백되면 외부 트랜잭션은 어떻게 되는가 — Propagation별 차이?
- `SUPPORTS`, `NOT_SUPPORTED`, `MANDATORY`, `NEVER`는 어떤 실무 상황에서 쓰는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 메서드 호출 시 트랜잭션 참여 방식을 세밀하게 제어해야 한다

```java
// 상황: 주문 처리 + 이메일 발송
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);        // 트랜잭션 A 참여
    paymentService.charge(order);       // 트랜잭션 A 참여? 새 트랜잭션?
    notificationService.sendEmail(order); // 트랜잭션 A와 무관하게?
    auditService.log(order);            // 실패해도 주문은 저장돼야 함
}
```

```
각 메서드가 어떤 트랜잭션 맥락에서 실행될지를 선언하는 것이 Propagation:

paymentService.charge():
  REQUIRED → 주문 트랜잭션과 같이 묶임 (실패 시 주문도 롤백)
  REQUIRES_NEW → 별도 트랜잭션 (결제는 별개로 커밋/롤백)

notificationService.sendEmail():
  NOT_SUPPORTED → 트랜잭션 없이 실행 (DB 락 최소화)
  SUPPORTS → 트랜잭션 있으면 참여, 없으면 그냥 실행

auditService.log():
  REQUIRES_NEW → 별도 트랜잭션 (주문 실패해도 감사 로그는 저장)
  NESTED → Savepoint 기반 (주문 트랜잭션 일부로, 실패해도 주문은 유지)
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: REQUIRES_NEW를 쓰면 항상 완전히 독립적인 트랜잭션이 된다

```java
// ❌ Self-Invocation + REQUIRES_NEW — 실제로 적용 안 됨
@Service
public class OrderService {

    @Transactional
    public void processOrder(Order order) {
        orderRepository.save(order);
        sendNotification(order);  // this.sendNotification() — 프록시 우회!
    }

    @Transactional(propagation = REQUIRES_NEW)  // ← 무시됨
    public void sendNotification(Order order) {
        // 실제로는 processOrder의 트랜잭션에 참여
    }
}

// ✅ REQUIRES_NEW가 동작하려면 별도 빈을 통한 호출이어야 함
@Service
@RequiredArgsConstructor
public class OrderService {
    private final NotificationService notificationService;  // 별도 빈

    @Transactional
    public void processOrder(Order order) {
        orderRepository.save(order);
        notificationService.sendNotification(order);  // 프록시 경유 → REQUIRES_NEW 적용
    }
}
```

### Before: NESTED는 REQUIRES_NEW처럼 완전히 독립적인 트랜잭션이다

```java
// ❌ 잘못된 이해:
// NESTED도 새 트랜잭션을 시작하므로 REQUIRES_NEW와 동일하다

// ✅ 실제 차이:
// REQUIRES_NEW: 외부 트랜잭션 일시정지 → 새 물리 트랜잭션 (새 Connection)
//               내부 롤백해도 외부 영향 없음
//               외부 롤백해도 내부 이미 커밋됨
//
// NESTED: 동일 물리 트랜잭션 내 Savepoint 생성
//         내부 롤백 → Savepoint까지만 롤백 (외부 트랜잭션 유지)
//         외부 롤백 → 전체 롤백 (내부 포함)
//         JPA 환경에서 완전 지원 안 될 수 있음 (드라이버 의존)
```

---

## ✨ 올바른 이해와 패턴

### After: 물리 트랜잭션 vs 논리 트랜잭션

```
물리 트랜잭션:
  실제 JDBC Connection의 트랜잭션
  setAutoCommit(false) ~ commit() or rollback()
  새 물리 트랜잭션 = 새 Connection (또는 같은 Connection에 Savepoint)

논리 트랜잭션:
  Spring이 관리하는 트랜잭션 참여 단위
  하나의 물리 트랜잭션에 여러 논리 트랜잭션이 참여 가능
  각 @Transactional 메서드 호출 = 논리 트랜잭션

예시:
  outer() @Transactional → 물리 트랜잭션 1개
    inner() @Transactional(REQUIRED) → 동일 물리 트랜잭션 참여 (논리 트랜잭션)
    inner() @Transactional(REQUIRES_NEW) → 새 물리 트랜잭션
    inner() @Transactional(NESTED) → 동일 물리 트랜잭션 + Savepoint
```

---

## 🔬 내부 동작 원리 — Propagation별 분기 소스 추적

### 1. 전체 분기 흐름 — handleExistingTransaction()

```java
// AbstractPlatformTransactionManager — 기존 트랜잭션 존재 시 Propagation 분기
private TransactionStatus handleExistingTransaction(
        TransactionDefinition definition, Object transaction, boolean debugEnabled) {

    // NEVER: 트랜잭션이 있으면 예외
    if (definition.getPropagationBehavior() == PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
            "Existing transaction found for transaction marked with propagation 'never'");
    }

    // NOT_SUPPORTED: 기존 트랜잭션 일시정지, 트랜잭션 없이 실행
    if (definition.getPropagationBehavior() == PROPAGATION_NOT_SUPPORTED) {
        Object suspendedResources = suspend(transaction);  // 기존 트랜잭션 일시정지
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(definition, null, false, newSynchronization, debugEnabled,
            suspendedResources);  // transaction=null → 트랜잭션 없이 실행
    }

    // REQUIRES_NEW: 기존 트랜잭션 일시정지, 새 트랜잭션 시작
    if (definition.getPropagationBehavior() == PROPAGATION_REQUIRES_NEW) {
        Object suspendedResources = suspend(transaction);  // 기존 트랜잭션 일시정지
        try {
            return startTransaction(definition, transaction, debugEnabled, suspendedResources);
            // → doBegin() 호출 → 새 Connection 획득 → 새 트랜잭션 시작
        } catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }

    // NESTED: Savepoint 생성
    if (definition.getPropagationBehavior() == PROPAGATION_NESTED) {
        if (useSavepointForNestedTransaction()) {
            // 기존 트랜잭션에 Savepoint 생성
            DefaultTransactionStatus status = prepareTransactionStatus(
                definition, transaction, false, false, debugEnabled, null);
            status.createAndHoldSavepoint();  // JDBC Savepoint 생성
            return status;
        } else {
            // Savepoint 불가 환경 → 새 트랜잭션 시작 (REQUIRES_NEW처럼)
            return startTransaction(definition, transaction, debugEnabled, null);
        }
    }

    // SUPPORTS, REQUIRED, MANDATORY: 기존 트랜잭션 참여
    // → 논리 트랜잭션만 생성 (물리 트랜잭션 공유)
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false,
        newSynchronization, debugEnabled, null);
    // isNewTransaction() = false → 참여 중임을 표시
}
```

### 2. suspend() — 트랜잭션 일시정지 메커니즘

```java
// AbstractPlatformTransactionManager.suspend() — REQUIRES_NEW / NOT_SUPPORTED 시 호출
protected final SuspendedResourcesHolder suspend(@Nullable Object transaction) {

    // 현재 ThreadLocal에 등록된 Synchronization 목록 꺼내기
    List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();

    Object suspendedResources = null;
    if (transaction != null) {
        suspendedResources = doSuspend(transaction);
        // JpaTransactionManager.doSuspend():
        //   EntityManagerHolder를 ThreadLocal에서 제거 (저장은 해둠)
        //   ConnectionHolder를 ThreadLocal에서 제거 (저장은 해둠)
        //   → 이후 새 트랜잭션의 doBegin()이 새 Connection 획득
    }

    // 현재 트랜잭션 이름, readOnly 등 ThreadLocal 정리
    String name = TransactionSynchronizationManager.getCurrentTransactionName();
    TransactionSynchronizationManager.setCurrentTransactionName(null);
    boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
    TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
    Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
    TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
    boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
    TransactionSynchronizationManager.setActualTransactionActive(false);

    // 일시정지된 모든 정보를 SuspendedResourcesHolder에 보관
    return new SuspendedResourcesHolder(
        suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
    // → DefaultTransactionStatus.suspendedResources에 저장
    // → 내부 트랜잭션 완료 후 resume()으로 복원
}
```

### 3. 7가지 Propagation 동작 요약

#### REQUIRED (기본값)

```
기존 트랜잭션 있음 → 참여 (논리 트랜잭션)
기존 트랜잭션 없음 → 새 트랜잭션 시작

내부 롤백 시:
  inner.setRollbackOnly() → 논리 트랜잭션 롤백 마킹
  outer.commit() 시도 → isGlobalRollbackOnly() = true
  → outer도 롤백 → UnexpectedRollbackException 발생!

실무 사용:
  대부분의 @Transactional 메서드 (기본값)
  하나의 업무 단위로 묶어야 하는 경우
```

#### REQUIRES_NEW

```
기존 트랜잭션 있음 → 일시정지 + 새 물리 트랜잭션 시작 (새 Connection)
기존 트랜잭션 없음 → 새 트랜잭션 시작

내부 롤백 시:
  내부 트랜잭션만 롤백 (외부 영향 없음)
내부 커밋 후 외부 롤백:
  내부는 이미 커밋됨 → 외부 롤백으로 되돌릴 수 없음!

실무 사용:
  감사 로그, 알림 발송 (주 트랜잭션 결과와 무관하게 저장)
  외부 시스템 호출 결과 기록
  주의: Connection 2개 점유 → Pool 고갈 위험
```

#### NESTED

```
기존 트랜잭션 있음 → 동일 물리 트랜잭션에 Savepoint 생성
기존 트랜잭션 없음 → 새 트랜잭션 시작 (REQUIRED와 동일)

내부 롤백 시:
  Savepoint까지만 롤백 (외부 트랜잭션 유지)
외부 롤백 시:
  Savepoint 이후 작업 포함 전체 롤백

실무 사용:
  외부 트랜잭션이 롤백되면 같이 롤백되어야 하지만
  내부 실패는 외부에 영향 없어야 하는 경우
  (예: 선택적 부가 처리)
주의: JPA 환경에서 지원 제한적 (JDBC 드라이버가 Savepoint 지원 필요)
```

#### SUPPORTS

```
기존 트랜잭션 있음 → 참여
기존 트랜잭션 없음 → 트랜잭션 없이 실행

실무 사용:
  읽기 전용 메서드 (트랜잭션 있으면 참여, 없어도 무방)
  @Transactional(propagation=SUPPORTS, readOnly=true)
```

#### NOT_SUPPORTED

```
기존 트랜잭션 있음 → 일시정지 + 트랜잭션 없이 실행
기존 트랜잭션 없음 → 트랜잭션 없이 실행

실무 사용:
  트랜잭션이 있으면 오히려 문제가 되는 작업
  외부 API 호출 (오래 걸리면 DB Lock 점유 방지)
  트랜잭션 없는 배치 읽기
```

#### MANDATORY

```
기존 트랜잭션 있음 → 참여
기존 트랜잭션 없음 → IllegalTransactionStateException 발생

실무 사용:
  반드시 외부 트랜잭션 컨텍스트에서만 호출해야 하는 메서드
  (독립 호출을 컴파일 타임이 아닌 런타임에 방지)
```

#### NEVER

```
기존 트랜잭션 있음 → IllegalTransactionStateException 발생
기존 트랜잭션 없음 → 트랜잭션 없이 실행

실무 사용:
  절대 트랜잭션 안에서 실행되어서는 안 되는 메서드
  (예: 장시간 외부 API 호출, 트랜잭션 내 실행 금지 정책 강제)
```

### 4. REQUIRED 내부 트랜잭션 롤백 → UnexpectedRollbackException

```java
// 함정: 내부 트랜잭션(REQUIRED)이 롤백되면 외부 트랜잭션도 반드시 롤백

@Transactional
public void outer() {
    outerRepository.save(data);
    try {
        inner();  // REQUIRED → 같은 물리 트랜잭션 참여
    } catch (Exception e) {
        // 예외를 잡아도 의미 없음!
        log.warn("내부 실패 무시", e);
    }
    // outer commit 시도
    // → inner에서 setRollbackOnly() 됐으므로 → UnexpectedRollbackException 발생!
}

@Transactional  // REQUIRED
public void inner() {
    innerRepository.save(data);
    throw new RuntimeException("내부 실패");
    // TransactionInterceptor가 rollbackOnly 마킹
}

// 해결 방법:
// 1. inner()를 REQUIRES_NEW로 변경 (독립 트랜잭션)
// 2. inner()를 별도 서비스 빈으로 분리 (NESTED 사용)
// 3. 애초에 inner()의 예외를 막음 (Checked Exception + noRollbackFor)
```

### 5. REQUIRES_NEW 동작 다이어그램

```
outer() 시작 → 물리 트랜잭션 1 시작 (Connection A)
  │
  ├── outerRepository.save() → Connection A에서 INSERT
  │
  ├── inner() 호출 (REQUIRES_NEW)
  │     → suspend(): Connection A를 ThreadLocal에서 제거 (보관)
  │     → doBegin(): Connection B 획득, 트랜잭션 2 시작
  │     → innerRepository.save() → Connection B에서 INSERT
  │     → 정상 완료: Connection B commit() → 트랜잭션 2 종료
  │     → resume(): Connection A를 ThreadLocal에 복원
  │
  └── outer 계속 → Connection A에서 추가 작업
      → outer commit() → Connection A commit() → 트랜잭션 1 종료

동시 사용 Connection: 최대 2개 (A + B)
→ Pool size가 작으면 Connection B 대기 발생 → 데드락 위험
```

---

## 💻 실험으로 확인하기

### 실험 1: REQUIRED — 내부 롤백이 외부에 미치는 영향

```java
@Test
void requiredRollbackPropagation() {
    assertThrows(UnexpectedRollbackException.class, () -> {
        outerService.outerWithRequired();
        // inner에서 롤백 → outer commit 시 UnexpectedRollbackException
    });

    // outer도 롤백됨 — DB에 아무것도 저장 안 됨
    assertEquals(0, userRepository.count());
}
```

### 실험 2: REQUIRES_NEW — 독립 커밋 확인

```java
@Test
void requiresNewIndependentCommit() {
    try {
        outerService.outerWithRequiresNew();  // outer 롤백
    } catch (Exception e) {
        // outer 실패
    }

    // inner(REQUIRES_NEW)는 이미 커밋 완료 → DB에 남아있음
    assertEquals(1, auditLogRepository.count());  // inner가 저장한 감사 로그
    assertEquals(0, orderRepository.count());      // outer 롤백으로 주문은 없음
}
```

### 실험 3: NESTED — Savepoint 롤백 확인

```java
@Test
@Transactional
void nestedSavepointRollback() {
    User user = userRepository.save(new User("외부"));

    try {
        nestedService.saveWithNested();  // 내부 실패 → Savepoint까지 롤백
    } catch (Exception e) {
        // 내부 예외 처리
    }

    // 외부 트랜잭션 유지 — user는 저장됨
    // 내부에서 저장 시도한 Post는 롤백됨
    assertEquals(1, userRepository.count());  // 외부 데이터 유지
    assertEquals(0, postRepository.count());  // 내부 데이터 롤백
}
```

---

## ⚡ 성능 임팩트

```
Propagation별 Connection 사용 비교:

REQUIRED (기존 참여):
  추가 Connection 없음 → Pool 영향 없음
  ThreadLocal 조회만 수행 → ~수백 ns 오버헤드

REQUIRES_NEW:
  추가 Connection 1개 획득
  동시에 2개 Connection 점유 (외부 + 내부)
  Pool 크기 부족 시 → 대기 → 타임아웃 → 데드락 위험
  → Pool 크기 = 예상 최대 동시 REQUIRES_NEW 중첩 깊이 고려 필요

NOT_SUPPORTED:
  현재 Connection을 ThreadLocal에서 제거 (반환은 아님)
  단, 트랜잭션 없으면 요청마다 새 Connection 획득/반환

NESTED:
  추가 Connection 없음 (Savepoint는 같은 Connection에서 생성)
  JDBC Savepoint 생성 비용: ~수십 μs
  → REQUIRES_NEW보다 Connection 효율적

실무 권고:
  REQUIRES_NEW 남용 금지 → Connection Pool 고갈 위험
  감사 로그, 알림 등 비핵심 처리에 제한적 사용
  NESTED는 JPA 환경 지원 여부 확인 후 사용
```

---

## 🤔 트레이드오프

```
REQUIRED (기본):
  Connection 효율 최상 (1개)
  내부 롤백 → 전체 롤백 (UnexpectedRollbackException 주의)
  대부분의 경우 올바른 선택

REQUIRES_NEW:
  독립성 보장 (내/외부 롤백 격리)
  Connection 2개 동시 점유 (Pool 고려 필수)
  감사 로그 / 알림에 적합, 과용 금지

NESTED:
  Connection 1개 유지 (Savepoint)
  외부 롤백 시 내부도 롤백 (REQUIRES_NEW와 다름)
  JPA 환경 지원 제한적
  REQUIRES_NEW보다 리소스 효율적 (가능하면 NESTED 우선)

SUPPORTS:
  읽기 전용 조회에 적합
  트랜잭션 있으면 참여, 없으면 그냥 실행
  유연하지만 트랜잭션 보장 없음

MANDATORY / NEVER:
  계약(contract) 강제 수단
  잘못된 호출을 런타임에 즉시 발견
  API 설계 의도를 코드로 표현
```

---

## 📌 핵심 정리

```
물리 트랜잭션 vs 논리 트랜잭션
  물리: 실제 JDBC Connection 트랜잭션 (1개)
  논리: @Transactional 메서드 참여 단위 (여러 개)
  REQUIRED: 여러 논리 트랜잭션 → 1개 물리 트랜잭션
  REQUIRES_NEW: 추가 물리 트랜잭션 (Connection 추가 점유)

REQUIRED 함정
  내부 RuntimeException → rollbackOnly 마킹
  외부 catch로 잡아도 → commit 시 UnexpectedRollbackException
  → 내부 실패를 외부에서 무시하려면 REQUIRES_NEW 또는 NESTED

REQUIRES_NEW 핵심
  suspend() → ThreadLocal에서 기존 트랜잭션 제거 (보관)
  doBegin() → 새 Connection 획득 → 새 트랜잭션
  완료 후 resume() → 기존 트랜잭션 복원
  Connection 동시 2개 사용 → Pool 크기 고려 필수

NESTED 핵심
  동일 Connection에 Savepoint 생성
  내부 롤백 → Savepoint까지만 (외부 유지)
  외부 롤백 → 전체 롤백 (내부 포함)
  JPA에서 지원 여부 드라이버에 의존

각 Propagation 한 줄 요약
  REQUIRED:      기존 참여, 없으면 새로
  REQUIRES_NEW:  항상 새 물리 트랜잭션
  NESTED:        Savepoint 기반 중첩
  SUPPORTS:      있으면 참여, 없으면 그냥
  NOT_SUPPORTED: 있으면 중단, 없이 실행
  MANDATORY:     없으면 예외
  NEVER:         있으면 예외
```

---

## 🤔 생각해볼 문제

**Q1.** `REQUIRED` Propagation으로 내부 메서드가 `RuntimeException`을 던지고 외부에서 `try-catch`로 잡았을 때, `UnexpectedRollbackException`이 발생하는 정확한 시점과 이유는?

**Q2.** `REQUIRES_NEW`를 사용하는 메서드가 재귀적으로 자기 자신을 5번 호출한다면, Connection Pool에서 Connection을 몇 개 점유하는가?

**Q3.** `NESTED` Propagation은 JPA 환경에서 완전히 지원되지 않을 수 있다고 했다. `JpaTransactionManager`가 `NESTED`를 처리할 때 어떤 조건에서 Savepoint를 사용하고, 어떤 조건에서 REQUIRES_NEW처럼 동작하는가?

> 💡 **해설**
>
> **Q1.** `UnexpectedRollbackException`은 외부 메서드가 `commit()`을 시도하는 시점에 발생한다. 내부 메서드에서 `RuntimeException`이 발생하면 `TransactionInterceptor`의 `completeTransactionAfterThrowing()`이 호출된다. 이때 내부는 `REQUIRED`로 기존 트랜잭션에 참여 중이므로 `isNewTransaction() = false`다. 자기가 시작한 트랜잭션이 아니므로 직접 롤백하는 대신 `TransactionStatus.setRollbackOnly()`를 호출해 전역 롤백 마킹만 한다. 외부에서 `try-catch`로 예외를 잡아도 이 마킹은 해제되지 않는다. 이후 외부 메서드가 정상 완료되어 `commit()`을 시도하면 `AbstractPlatformTransactionManager.commit()`이 `isGlobalRollbackOnly() = true`를 감지하고 롤백 후 `UnexpectedRollbackException`을 던진다.
>
> **Q2.** 최대 5개(또는 6개)를 점유한다. 첫 번째 호출에서 Connection 1을 획득하고, `REQUIRES_NEW`로 자기 자신을 호출하면 Connection 1을 `suspend()`하고 Connection 2를 획득한다. 다시 자기 자신을 호출하면 Connection 2를 suspend하고 Connection 3을 획득한다. 이렇게 5번 중첩되면 동시에 5개의 Connection이 모두 점유된 상태가 된다. Pool 크기가 5 미만이면 6번째 시도에서 Connection 대기가 발생하고, `connectionTimeout` 초과 시 예외가 발생한다. 재귀 구조에서 `REQUIRES_NEW`는 Pool 고갈의 전형적인 원인이다.
>
> **Q3.** `AbstractPlatformTransactionManager.getTransaction()`에서 `NESTED` 분기로 진입한 후 `useSavepointForNestedTransaction()`을 호출한다. 이 메서드는 `nestedTransactionAllowed` 플래그를 확인한다. `JpaTransactionManager`는 기본적으로 `nestedTransactionAllowed = false`로 설정되어 있어 Savepoint 대신 REQUIRES_NEW처럼 새 트랜잭션을 시작한다. JDBC `DataSourceTransactionManager`는 기본적으로 `nestedTransactionAllowed = true`이며 `Connection.setSavepoint()`를 사용한다. JPA 환경에서 진짜 NESTED(Savepoint)를 사용하려면 `JpaTransactionManager.setNestedTransactionAllowed(true)`를 설정해야 하며, Hibernate Session이 JDBC Connection의 Savepoint를 지원하는지 확인이 필요하다.

---

<div align="center">

**[⬅️ 이전: @Transactional 프록시 생성 메커니즘](./02-transactional-proxy-mechanism.md)** | **[다음: Isolation Level과 Database Lock ➡️](./04-isolation-level-lock.md)**

</div>
