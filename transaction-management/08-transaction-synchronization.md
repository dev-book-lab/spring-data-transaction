# TransactionSynchronization 활용 — 트랜잭션 이벤트 훅과 ThreadLocal 자원 관리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `TransactionSynchronizationManager`는 어떤 정보를 ThreadLocal에 저장하는가?
- `afterCommit()` 훅이 필요한 전형적인 상황과 `afterCompletion()`과의 차이는?
- `@TransactionalEventListener`가 내부적으로 `TransactionSynchronization`을 어떻게 사용하는가?
- 트랜잭션 커밋 이후에 이벤트를 발행해야 하는 이유 — 커밋 전 발행의 문제점은?
- `TransactionSynchronization` 등록이 트랜잭션 없는 환경에서 호출되면 어떻게 되는가?

---

## 🔍 왜 이게 존재하는가

### 문제: 트랜잭션 커밋 이후에 실행해야 할 작업이 있다

```java
// 문제 상황: 주문 완료 후 이메일 발송
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);
    emailService.sendOrderConfirmation(order); // ← 여기서 발송하면 문제!
    // 이메일 발송 후 트랜잭션이 롤백되면?
    // 이메일은 이미 나갔는데 주문은 DB에 없음 → 데이터 불일치
}
```

```
이메일, 푸시 알림, 외부 API 호출, 캐시 무효화 등
트랜잭션 커밋이 확정된 후에 실행해야 하는 작업들:

  트랜잭션 내부:
    DB에 데이터가 기록됨 (아직 커밋 전)
    이 시점의 외부 호출은 롤백 시 불일치 발생

  트랜잭션 커밋 후:
    DB 데이터 영구 확정
    외부 시스템 호출이 DB 상태와 일치

해결:
  TransactionSynchronization.afterCommit() 훅
  → 커밋이 완료된 직후 콜백 실행
  → @TransactionalEventListener로 선언적 사용 가능
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: afterCommit()에서 트랜잭션이 필요한 작업을 수행한다

```java
// ❌ afterCommit() 내부에서 @Transactional 메서드 호출 — 트랜잭션 없음
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            // 이 시점에 트랜잭션 컨텍스트가 없음!
            // @Transactional 메서드를 호출해도 새 트랜잭션이 시작되지 않을 수 있음
            // (PROPAGATION_REQUIRED라면 새 트랜잭션 시작되지만, 이 훅은 트랜잭션 범위 밖)
            anotherService.doSomething(); // 별도 트랜잭션으로 실행 (REQUIRED → 새 트랜잭션)
        }
    }
);

// ✅ afterCommit()에서 트랜잭션이 필요하면 PROPAGATION_REQUIRES_NEW 명시
// 또는 @TransactionalEventListener(phase = AFTER_COMMIT) 사용
```

### Before: @TransactionalEventListener는 항상 비동기로 실행된다

```java
// ❌ 잘못된 이해: @TransactionalEventListener = 비동기 처리

// ✅ 실제:
// @TransactionalEventListener는 기본적으로 동기 실행
// → 같은 스레드, 트랜잭션 커밋 직후 실행
// 비동기 처리가 필요하면 @Async 추가 필요

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Async  // ← 비동기 처리하려면 명시적으로 추가
public void handleOrderPlaced(OrderPlacedEvent event) {
    emailService.sendConfirmation(event.getOrder());
}
```

### Before: 트랜잭션 없는 환경에서도 TransactionSynchronization이 동작한다

```java
// ❌ 잘못된 이해: 트랜잭션이 없어도 afterCommit()이 나중에 호출된다

// ✅ 실제:
// 활성 트랜잭션이 없으면 registerSynchronization() 자체가 예외 발생
// IllegalStateException: "Transaction synchronization is not active"

// @TransactionalEventListener의 경우:
// 트랜잭션 없이 이벤트를 발행하면 → 기본적으로 이벤트 무시 (실행 안 됨!)
// fallbackExecution = true 설정 시 → 트랜잭션 없어도 즉시 실행

@TransactionalEventListener(
    phase = AFTER_COMMIT,
    fallbackExecution = true  // 트랜잭션 없어도 실행
)
public void handleEvent(MyEvent event) { ... }
```

---

## ✨ 올바른 이해와 패턴

### After: TransactionSynchronization 활용 시나리오

```
afterCommit() 사용:
  이메일/SMS/푸시 알림 발송 (커밋 확정 후)
  외부 API 호출 (Webhook, Slack 알림)
  캐시 무효화 (DB 데이터 확정 후 캐시 삭제)
  이벤트 발행 (메시지 큐, Kafka 프로듀서)

afterCompletion() 사용:
  커밋/롤백 무관 정리 작업
  임시 파일 삭제, 락 해제
  성능 측정 종료

beforeCommit() 사용:
  커밋 전 최종 유효성 검증
  추가 데이터 저장 (커밋과 같이 묶어야 할 때)
  flush 보장이 필요한 경우
```

---

## 🔬 내부 동작 원리 — TransactionSynchronization 소스 추적

### 1. TransactionSynchronizationManager — ThreadLocal 저장소 전체 구조

```java
// TransactionSynchronizationManager — 트랜잭션 컨텍스트의 모든 ThreadLocal
public abstract class TransactionSynchronizationManager {

    // [1] 트랜잭션 자원 (Connection, EntityManager 등)
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

    // [2] 커밋/롤백 후 콜백 목록
    private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
        new NamedThreadLocal<>("Transaction synchronizations");

    // [3] 현재 트랜잭션 활성 여부
    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");

    // [4] 현재 트랜잭션 이름
    private static final ThreadLocal<String> currentTransactionName =
        new NamedThreadLocal<>("Current transaction name");

    // [5] readOnly 여부
    private static final ThreadLocal<Boolean> currentTransactionReadOnly =
        new NamedThreadLocal<>("Current transaction read-only status");

    // [6] 현재 Isolation Level
    private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
        new NamedThreadLocal<>("Current transaction isolation level");

    // Synchronization 등록
    public static void registerSynchronization(TransactionSynchronization synchronization) {
        Set<TransactionSynchronization> synchs = synchronizations.get();
        if (synchs == null) {
            throw new IllegalStateException("Transaction synchronization is not active");
            // 활성 트랜잭션 없으면 예외!
        }
        synchs.add(synchronization);
    }
}
```

### 2. TransactionSynchronization 인터페이스 — 전체 훅 목록

```java
// TransactionSynchronization — 트랜잭션 이벤트 콜백 인터페이스
public interface TransactionSynchronization extends Ordered, Flushable {

    // 트랜잭션 완료 상태 상수
    int STATUS_COMMITTED  = 0;
    int STATUS_ROLLED_BACK = 1;
    int STATUS_UNKNOWN    = 2;

    // 커밋 직전 (flush 가능)
    default void beforeCommit(boolean readOnly) {}

    // 커밋/롤백 전 (트랜잭션 완료 직전)
    default void beforeCompletion() {}

    // 커밋 직후 (트랜잭션 완료 전!)
    // 주의: 예외 발생 시 데이터 불일치 가능
    default void afterCommit() {}

    // 트랜잭션 완전 종료 후 (커밋/롤백 모두 호출)
    // status: STATUS_COMMITTED, STATUS_ROLLED_BACK, STATUS_UNKNOWN
    default void afterCompletion(int status) {}

    // 우선순위 (낮을수록 먼저 실행)
    @Override
    default int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

### 3. AbstractPlatformTransactionManager — 훅 호출 시점

```java
// AbstractPlatformTransactionManager.processCommit() — 커밋 처리 흐름
private void processCommit(DefaultTransactionStatus status) throws TransactionException {

    try {
        boolean beforeCompletionInvoked = false;
        try {
            boolean unexpectedRollback = false;

            // [1] beforeCommit() 호출 (커밋 직전)
            triggerBeforeCommit(status);

            // [2] beforeCompletion() 호출
            triggerBeforeCompletion(status);
            beforeCompletionInvoked = true;

            // [3] 실제 커밋 실행
            if (status.hasSavepoint()) {
                status.releaseHeldSavepoint();  // NESTED: Savepoint 해제
            } else if (status.isNewTransaction()) {
                doCommit(status);  // 물리 커밋
            }

            // [4] afterCommit() 호출 (커밋 직후, 트랜잭션 완전 종료 전)
            triggerAfterCommit(status);

        } finally {
            // [5] afterCompletion(STATUS_COMMITTED) 호출
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
        }

    } finally {
        // [6] 트랜잭션 자원 정리 (ThreadLocal unbind)
        cleanupAfterCompletion(status);
    }
}

// triggerAfterCommit() — afterCommit() 순차 호출
private void triggerAfterCommit(DefaultTransactionStatus status) {
    if (status.isNewSynchronization()) {
        TransactionSynchronizationUtils.triggerAfterCommit();
        // → 등록된 모든 TransactionSynchronization.afterCommit() 순차 호출
    }
}
```

### 4. afterCommit() vs afterCompletion() 차이

```
실행 시점 비교:

doCommit() ────────────────── 물리 커밋
     │
     ▼
afterCommit() ─────────────── 커밋 직후, 트랜잭션 자원 정리 전
     │                         → EntityManager, Connection 아직 열려있음
     │                         → 트랜잭션 컨텍스트(ThreadLocal) 아직 유효
     │
     ▼
afterCompletion(COMMITTED) ── 트랜잭션 완전 종료 후
     │                         → EntityManager, Connection 반환 완료
     │                         → ThreadLocal 정리 완료
     │
     ▼
cleanupAfterCompletion() ──── 자원 완전 해제

afterCommit() 사용:
  EntityManager / Connection이 아직 열려있는 상태에서 추가 DB 작업 가능
  (단, 새 트랜잭션 필요 — 현재 트랜잭션은 이미 커밋 완료)

afterCompletion() 사용:
  커밋/롤백 무관한 정리 작업
  status 파라미터로 성공/실패 구분 가능
```

### 5. @TransactionalEventListener — 선언적 afterCommit() 패턴

```java
// [1] 이벤트 발행 (트랜잭션 내부)
@Service
@RequiredArgsConstructor
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        // 트랜잭션 커밋 전에 이벤트 발행
        // 실제 리스너 실행은 커밋 후에 발생
        eventPublisher.publishEvent(new OrderPlacedEvent(this, order));
    }
}

// [2] 이벤트 클래스
public class OrderPlacedEvent extends ApplicationEvent {
    private final Order order;
    public OrderPlacedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}

// [3] 이벤트 리스너 — 커밋 후 실행
@Component
public class OrderEventListener {

    // AFTER_COMMIT: 트랜잭션 커밋 성공 후에만 실행
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.getOrder()); // 커밋 확정 후 이메일 발송
        // 롤백 시에는 실행 안 됨 → 이메일/주문 불일치 없음
    }

    // AFTER_ROLLBACK: 롤백 후에만 실행
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleOrderFailed(OrderPlacedEvent event) {
        notificationService.alertAdmin(event.getOrder()); // 실패 알림
    }

    // AFTER_COMPLETION: 커밋/롤백 모두 실행
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void handleOrderFinished(OrderPlacedEvent event) {
        metricsService.recordOrderAttempt(event.getOrder()); // 시도 횟수 기록
    }

    // BEFORE_COMMIT: 커밋 직전 실행 (트랜잭션 내부)
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void handleBeforeCommit(OrderPlacedEvent event) {
        auditRepository.save(new AuditLog(event)); // 감사 로그 (같은 트랜잭션)
    }
}
```

### 6. @TransactionalEventListener 내부 — TransactionSynchronization 연결

```java
// TransactionalApplicationListenerMethodAdapter
// @TransactionalEventListener을 TransactionSynchronization으로 연결
class TransactionalApplicationListenerMethodAdapter
        extends ApplicationListenerMethodAdapter
        implements TransactionalApplicationListener<ApplicationEvent> {

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // 활성 트랜잭션이 있는지 확인
        if (TransactionSynchronizationManager.isSynchronizationActive()
                && isListenerConditionMet(event)) {

            // TransactionSynchronization 등록
            TransactionSynchronizationManager.registerSynchronization(
                new TransactionSynchronization() {
                    @Override
                    public void afterCommit() {
                        // AFTER_COMMIT 리스너 실행
                        if (phase == TransactionPhase.AFTER_COMMIT) {
                            processEvent(event);
                        }
                    }

                    @Override
                    public void afterCompletion(int status) {
                        // AFTER_ROLLBACK: status == STATUS_ROLLED_BACK
                        // AFTER_COMPLETION: 항상
                        if (shouldProcess(phase, status)) {
                            processEvent(event);
                        }
                    }
                }
            );

        } else if (this.annotation.fallbackExecution()) {
            // 트랜잭션 없으면 즉시 실행 (fallbackExecution=true인 경우)
            processEvent(event);
        }
        // 트랜잭션 없고 fallbackExecution=false → 이벤트 무시
    }
}
```

### 7. 실용적 afterCommit() 패턴 — 직접 등록

```java
// 직접 TransactionSynchronization 등록 (세밀한 제어 필요 시)
@Service
public class ProductService {

    @Transactional
    public void updateProduct(Product product) {
        productRepository.save(product);

        // 커밋 후 캐시 무효화 등록
        Long productId = product.getId();
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronizationAdapter() {  // 인터페이스 default 메서드 구현체
                @Override
                public void afterCommit() {
                    // 커밋 확정 후 캐시 무효화
                    cacheManager.evict("products", productId);
                    // 트랜잭션 롤백 시 실행 안 됨 → 캐시 불일치 없음
                }

                @Override
                public void afterCompletion(int status) {
                    if (status == STATUS_ROLLED_BACK) {
                        log.warn("상품 업데이트 롤백: productId={}", productId);
                    }
                }

                @Override
                public int getOrder() {
                    return 0;  // 다른 Synchronization보다 먼저 실행
                }
            }
        );
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: afterCommit() 실행 시점 확인

```java
@Transactional
@Test
void afterCommitTiming() {
    List<String> events = new ArrayList<>();

    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronizationAdapter() {
            @Override
            public void beforeCommit(boolean readOnly) {
                events.add("beforeCommit");
            }
            @Override
            public void afterCommit() {
                events.add("afterCommit");
            }
            @Override
            public void afterCompletion(int status) {
                events.add("afterCompletion:" + status);
            }
        }
    );

    userRepository.save(new User("테스트"));
    events.add("메서드 종료");

    // 실행 순서 확인
    // 출력: ["메서드 종료", "beforeCommit", "afterCommit", "afterCompletion:0"]
}
```

### 실험 2: @TransactionalEventListener — 롤백 시 이벤트 무시 확인

```java
@Test
void eventNotFiredOnRollback() {
    List<String> firedEvents = new ArrayList<>();

    // 이벤트 리스너 모의
    // (실제로는 @TransactionalEventListener 사용)

    assertThrows(RuntimeException.class, () -> {
        orderService.placeOrderWithFailure(); // 예외로 롤백
    });

    // AFTER_COMMIT 리스너는 실행 안 됨
    assertTrue(firedEvents.isEmpty());
    assertEquals(0, orderRepository.count()); // 롤백 확인
}
```

### 실험 3: 트랜잭션 없는 환경에서 fallbackExecution

```java
// 트랜잭션 없이 이벤트 발행
public void publishWithoutTransaction(OrderPlacedEvent event) {
    eventPublisher.publishEvent(event); // 트랜잭션 없음

    // fallbackExecution = false (기본): 이벤트 무시
    // fallbackExecution = true: 즉시 실행
}

@TransactionalEventListener(phase = AFTER_COMMIT, fallbackExecution = true)
public void handleWithFallback(OrderPlacedEvent event) {
    System.out.println("실행됨: " + event.getOrder().getId());
    // 트랜잭션 없으면 즉시 실행, 있으면 커밋 후 실행
}
```

---

## ⚡ 성능 임팩트

```
TransactionSynchronization 등록/실행 비용:

등록:
  ThreadLocal Set에 추가: ~수백 ns
  → 무시 가능

afterCommit() 실행:
  등록된 Synchronization 수에 비례: O(n)
  통상 1~5개 → ~수 μs
  → 무시 가능 (콜백 내용이 병목)

afterCommit() 내 외부 API 호출 주의:
  이메일 발송: ~수백 ms ~ 수 초
  → 커밋 완료 후 실행이므로 DB 트랜잭션에는 영향 없음
  → 단, HTTP 요청 스레드가 블로킹될 수 있음
  → 비동기 처리 권장 (@Async 또는 메시지 큐)

@TransactionalEventListener vs 직접 등록:
  성능 차이 없음 (내부적으로 동일한 TransactionSynchronization 사용)
  @TransactionalEventListener: 선언적, 코드 분리 용이
  직접 등록: 람다로 간결, 세밀한 제어 가능
```

---

## 🤔 트레이드오프

```
afterCommit() 직접 사용:
  ✅ 트랜잭션 커밋 확정 후 실행 → 데이터 일관성 보장
  ✅ 롤백 시 미실행 → 외부 시스템 불일치 방지
  ❌ afterCommit() 내 예외가 호출자에게 전파됨 (커밋은 이미 완료)
  ❌ 재시도 메커니즘 없음 (이메일 발송 실패 시 손실)
  → 신뢰성이 필요하면 Outbox 패턴 또는 메시지 큐 사용

@TransactionalEventListener:
  ✅ 코드 분리 (DIP, 이벤트 기반 설계)
  ✅ 여러 리스너를 독립적으로 추가 가능
  ✅ phase 설정으로 세밀한 시점 제어
  ❌ fallbackExecution 기본값 false → 트랜잭션 없으면 이벤트 누락 위험
  ❌ 비동기 실행 시 @Async + SecurityContext 전파 필요

Outbox 패턴 (신뢰성 보장 필요 시):
  트랜잭션 내 "발송 예정" 레코드를 같은 DB에 저장
  커밋 후 폴러(Poller)가 레코드 읽어 실제 발송
  → 트랜잭션 보장 + 재시도 가능 + 메시지 손실 없음
  ❌ 구현 복잡도 증가

TransactionSynchronization 등록 위치:
  Service 레이어에서 직접: 인프라 의존 (코드 오염)
  Domain Event + @TransactionalEventListener: 관심사 분리 (권장)
  Spring ApplicationEvent 패턴: Domain에서 이벤트 발행, Infrastructure에서 처리
```

---

## 📌 핵심 정리

```
TransactionSynchronizationManager ThreadLocal 구성
  resources:        트랜잭션 자원 (Connection, EntityManager)
  synchronizations: 커밋/롤백 콜백 목록
  actualTransactionActive: 활성 트랜잭션 여부
  currentTransactionReadOnly: readOnly 여부
  currentTransactionName: 트랜잭션 이름

TransactionSynchronization 훅 실행 순서
  beforeCommit(readOnly)     → 커밋 직전 (flush 가능)
  beforeCompletion()         → 커밋/롤백 직전
  doCommit() / doRollback()  → 실제 커밋/롤백
  afterCommit()              → 커밋 직후 (자원 정리 전)
  afterCompletion(status)    → 트랜잭션 완전 종료 후

@TransactionalEventListener 동작
  이벤트 발행 → TransactionSynchronization 등록 (즉시 실행 안 함)
  커밋 후 → afterCommit() 또는 afterCompletion()에서 리스너 실행
  트랜잭션 없음 → 이벤트 무시 (fallbackExecution=false 기본)

afterCommit() 사용 시 주의사항
  예외 발생 → 커밋은 완료됐으므로 롤백 불가
  트랜잭션 필요 → PROPAGATION_REQUIRES_NEW로 새 트랜잭션 시작
  외부 API 호출 → @Async 또는 메시지 큐로 비동기 처리 권장
  신뢰성 필요 → Outbox 패턴
```

---

## 🤔 생각해볼 문제

**Q1.** `@TransactionalEventListener(phase = AFTER_COMMIT)`으로 이벤트를 처리하는 중에 예외가 발생하면 어떻게 되는가? 원래 트랜잭션(이미 커밋된)에 영향을 주는가?

**Q2.** 다음 코드에서 `afterCommit()` 내부에서 새 트랜잭션으로 데이터를 저장하려 한다. 이것이 올바르게 동작하려면 무엇이 필요한가?
```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronizationAdapter() {
        @Override
        public void afterCommit() {
            auditService.saveLog(data); // @Transactional(REQUIRED)
        }
    }
);
```

**Q3.** `TransactionSynchronizationManager.registerSynchronization()`을 `@Transactional(readOnly = true)` 메서드 내에서 호출하고, `afterCommit()` 훅에서 DB 쓰기를 시도하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** 원래 트랜잭션에는 영향을 주지 않는다. `afterCommit()`은 커밋이 완료된 이후에 실행되므로, 이 시점의 예외는 이미 커밋된 트랜잭션을 롤백할 수 없다. 예외는 `triggerAfterCommit()`을 호출한 코드로 전파된다. Spring의 `AbstractPlatformTransactionManager.processCommit()` 내에서 `afterCommit()` 예외는 `afterCompletion()` 호출까지는 진행되지만, `cleanupAfterCompletion()`은 finally 블록으로 보장된다. `@TransactionalEventListener`에서 예외가 발생하면 `TransactionalApplicationListenerMethodAdapter`가 예외를 잡아 로그를 남기거나 상위로 전파한다. 결론적으로 사용자가 받는 HTTP 응답에는 영향을 줄 수 있지만(예외 전파 시) 이미 커밋된 DB 데이터는 변경되지 않는다.
>
> **Q2.** `auditService.saveLog()`에 `@Transactional(propagation = REQUIRES_NEW)` 또는 최소한 `@Transactional`이 있어야 한다. `afterCommit()` 실행 시점에는 원래 트랜잭션이 커밋 완료되어 `TransactionSynchronizationManager`의 자원이 정리되기 시작한다. `REQUIRED` Propagation이면 기존 트랜잭션이 없으므로 새 트랜잭션을 시작한다(`afterCommit()` 시점에는 기존 트랜잭션 자원이 ThreadLocal에서 해제됨). 실제로는 `afterCommit()`이 `afterCompletion()` 이전에 호출되므로 아직 완전히 정리되지 않은 상태일 수 있어, `REQUIRES_NEW`를 명시적으로 사용하는 것이 더 안전하다. 또한 `auditService`가 Spring 빈이어야 하고 `@Transactional` AOP 프록시가 적용되어 있어야 한다.
>
> **Q3.** `afterCommit()` 훅 자체는 `readOnly` 트랜잭션의 커밋 후에 실행된다. 이 시점에 원래 `readOnly` 트랜잭션은 완료됐으므로 ThreadLocal의 readOnly 설정도 해제된다. `afterCommit()` 내에서 새 트랜잭션(`REQUIRED` → 기존 없음 → 새 트랜잭션 시작)으로 DB 쓰기를 시도하면, 새 트랜잭션은 readOnly가 아니므로 정상적으로 쓰기가 가능하다. 단, `afterCommit()`이 실행되는 시점은 `afterCompletion()` 이전이고 `cleanupAfterCompletion()` 이전이므로, 아직 이전 트랜잭션의 일부 자원이 ThreadLocal에 남아있을 수 있다. 따라서 `afterCommit()` 내 새 트랜잭션은 `PROPAGATION_REQUIRES_NEW`를 명시하여 완전히 새로운 컨텍스트를 보장하는 것이 권장된다.

---

<div align="center">

**[⬅️ 이전: Rollback 규칙](./07-rollback-rules.md)** | **[홈으로 🏠](../README.md)**

</div>
