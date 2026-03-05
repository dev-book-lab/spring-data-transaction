# Rollback 규칙 — checked vs unchecked 예외의 트랜잭션 처리 차이

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Spring이 `RuntimeException`은 롤백하고 `CheckedException`은 커밋하는 기술적 근거는?
- `RuleBasedTransactionAttribute.rollbackOn()`이 예외 계층에서 가장 구체적인 규칙을 선택하는 방법은?
- `rollbackFor = Exception.class`를 쓰면 모든 예외에서 롤백되는가?
- Checked Exception으로 인해 데이터가 오염되는 실제 사례와 방지 패턴은?
- 비즈니스 예외를 설계할 때 Checked vs Unchecked 선택 기준은?

---

## 🔍 왜 이게 존재하는가

### 문제: 예외가 발생했는데 트랜잭션이 롤백되지 않아 데이터가 오염된다

```java
// 실제 사고 사례 패턴
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);           // 주문 저장
    paymentService.charge(order);          // 결제 처리
    inventoryService.decreaseStock(order); // 재고 차감
}

// inventoryService.decreaseStock()이 Checked Exception을 던진다
public void decreaseStock(Order order) throws StockException {
    // 재고 부족 → StockException (Checked Exception)
    throw new StockException("재고 부족");
}

// 결과:
// StockException은 Checked Exception → Spring 기본 정책: 롤백 안 함
// → orderRepository.save(order) 커밋됨
// → paymentService.charge(order) 커밋됨 (결제 완료)
// → decreaseStock 실패
// → 재고는 그대로인데 주문과 결제는 완료 상태 → 데이터 오염!
```

```
Spring의 기본 롤백 정책:
  RuntimeException (Unchecked) → 롤백
  Error                        → 롤백
  Exception (Checked)          → 커밋 (예외는 상위로 전파됨)

이 정책의 배경:
  Java 설계 철학에서 Checked Exception = 복구 가능한 예외
  → 호출자가 처리하도록 강제 (catch or throws 선언)
  → 복구 가능하다면 트랜잭션을 유지하는 것이 합리적

  RuntimeException = 복구 불가능한 예외 (프로그래밍 오류)
  → 롤백이 적절

실무 함정:
  비즈니스 예외를 Checked Exception으로 설계하면 롤백 안 됨
  → rollbackFor 명시 or Unchecked로 전환 필요
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 예외가 발생하면 항상 트랜잭션이 롤백된다

```java
// ❌ 잘못된 이해: 모든 예외 = 롤백

// 아래 코드는 롤백되지 않음
@Transactional
public void riskyOperation() throws IOException {
    repository.save(data);
    externalService.call(); // IOException 발생 → Checked → 롤백 안 됨!
}
// → data는 DB에 저장된 채로 남음

// ✅ Checked Exception도 롤백하려면 명시 필요
@Transactional(rollbackFor = IOException.class)
public void riskyOperation() throws IOException {
    repository.save(data);
    externalService.call(); // IOException → rollbackFor에 의해 롤백
}
```

### Before: rollbackFor = Exception.class 이면 모든 예외에서 롤백된다

```java
// ✅ 맞는 이해이나 주의 필요
@Transactional(rollbackFor = Exception.class)
public void process() throws Exception {
    repository.save(data);
    // RuntimeException, IOException, SQLException 모두 롤백
}

// 단, noRollbackFor와 조합 시 주의
@Transactional(
    rollbackFor = Exception.class,
    noRollbackFor = BusinessException.class  // BusinessException만 롤백 안 함
)
public void process() throws Exception {
    // BusinessException: 커밋 (noRollbackFor 우선)
    // 그 외: 롤백 (rollbackFor = Exception.class)
}
```

### Before: @Transactional 메서드가 예외를 catch하면 롤백 여부를 제어할 수 있다

```java
// ❌ 잘못된 이해:
@Transactional
public void process() {
    try {
        riskyWork();  // RuntimeException 발생
    } catch (RuntimeException e) {
        log.warn("무시", e);
        // 예외를 잡으면 롤백 안 될 것 같지만...
    }
    // ← 여기서는 실제로 롤백 안 됨 (예외가 TransactionInterceptor까지 전파 안 됨)
    // 단, 내부 트랜잭션(REQUIRED)이 rollbackOnly를 마킹했다면 UnexpectedRollbackException
}

// ✅ 올바른 이해:
// catch로 예외를 잡으면 TransactionInterceptor가 예외를 볼 수 없음 → 롤백 안 됨
// 단, 같은 트랜잭션 내 내부 메서드의 REQUIRED 예외 → rollbackOnly 마킹은 남음
// 명시적으로 롤백하려면:
@Transactional
public void process() {
    try {
        riskyWork();
    } catch (RuntimeException e) {
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly(); // 명시적 마킹
        log.warn("처리 후 롤백", e);
    }
}
```

---

## ✨ 올바른 이해와 패턴

### After: 롤백 규칙 설계 원칙

```
기본 정책 (대부분 적합):
  RuntimeException / Error → 롤백
  Checked Exception → 커밋 후 예외 전파

비즈니스 예외 설계 권장:
  Unchecked 예외로 설계 → 별도 rollbackFor 없이 자동 롤백
  Spring, JPA 계층 자체도 RuntimeException 기반

rollbackFor 사용:
  외부 라이브러리가 Checked Exception만 던질 때
  기존 코드 수정 없이 롤백 규칙 추가할 때

noRollbackFor 사용:
  특정 비즈니스 예외에서 부분 커밋이 의도적으로 필요할 때
  예: 감사 로그는 저장하고 주 작업은 실패 처리
```

---

## 🔬 내부 동작 원리 — Rollback 규칙 소스 추적

### 1. rollbackOn() — 롤백 여부 결정 핵심 로직

```java
// RuleBasedTransactionAttribute — rollbackFor / noRollbackFor 처리
public class RuleBasedTransactionAttribute extends DefaultTransactionAttribute {

    @Nullable
    private List<RollbackRuleAttribute> rollbackRules;

    @Override
    public boolean rollbackOn(Throwable ex) {

        RollbackRuleAttribute winner = null;
        int deepest = Integer.MAX_VALUE;

        // rollbackFor, noRollbackFor 규칙 목록 순회
        if (this.rollbackRules != null) {
            for (RollbackRuleAttribute rule : this.rollbackRules) {
                int depth = rule.getDepth(ex);
                // depth: 예외 계층에서 규칙과의 거리
                //   exact match: 0
                //   부모 클래스: 1, 2, 3, ...
                //   매칭 안 됨: -1

                if (depth >= 0 && depth < deepest) {
                    deepest = depth;
                    winner = rule;  // 가장 구체적인 규칙 선택
                }
            }
        }

        // 매칭 규칙이 없으면 기본 정책 적용
        if (winner == null) {
            return super.rollbackOn(ex);
            // DefaultTransactionAttribute.rollbackOn():
            // return (ex instanceof RuntimeException || ex instanceof Error)
        }

        // 가장 구체적인 규칙 적용
        return !(winner instanceof NoRollbackRuleAttribute);
        // RollbackRuleAttribute → true (롤백)
        // NoRollbackRuleAttribute → false (커밋)
    }
}

// RollbackRuleAttribute.getDepth() — 예외 계층 거리 계산
public int getDepth(Throwable exception) {
    return getDepth(exception.getClass(), 0);
}

private int getDepth(Class<?> exceptionClass, int depth) {
    if (exceptionClass.getName().contains(this.exceptionName)) {
        return depth;  // 매칭됨 → 거리 반환
    }
    if (exceptionClass == Throwable.class) {
        return -1;  // 최상위까지 올라갔는데 매칭 없음
    }
    // 부모 클래스로 올라가며 재귀 탐색
    return getDepth(exceptionClass.getSuperclass(), depth + 1);
}
```

### 2. 규칙 우선순위 — 구체적인 예외가 우선

```java
// 예시: rollbackFor + noRollbackFor 조합
@Transactional(
    rollbackFor = BusinessException.class,
    noRollbackFor = SpecificBusinessException.class
    // SpecificBusinessException extends BusinessException
)
public void process() throws BusinessException { ... }

// SpecificBusinessException 발생 시:
//   rollbackFor = BusinessException: depth = 1 (부모)
//   noRollbackFor = SpecificBusinessException: depth = 0 (정확 매칭)
//   winner = noRollbackFor (더 구체적) → 롤백 안 함 (커밋)

// BusinessException 발생 시:
//   rollbackFor = BusinessException: depth = 0 (정확 매칭)
//   noRollbackFor = SpecificBusinessException: depth = -1 (매칭 안 됨)
//   winner = rollbackFor → 롤백
```

### 3. completeTransactionAfterThrowing() — 실제 적용 흐름

```java
// TransactionAspectSupport — 예외 처리 및 롤백/커밋 결정
protected void completeTransactionAfterThrowing(
        @Nullable TransactionInfo txInfo, Throwable ex) {

    if (txInfo != null && txInfo.getTransactionStatus() != null) {

        // rollbackOn() 호출 → 롤백 여부 결정
        if (txInfo.transactionAttribute != null
                && txInfo.transactionAttribute.rollbackOn(ex)) {

            // 롤백 실행
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());

        } else {
            // 커밋 실행 (Checked Exception 기본 동작)
            // 예외는 상위로 계속 전파됨
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
    }
}
```

### 4. Checked Exception 롤백 패턴 — 실무 설계

```java
// 패턴 1: rollbackFor 명시 (기존 Checked Exception 유지)
@Transactional(rollbackFor = {StockException.class, PaymentException.class})
public void processOrder(Order order) throws StockException, PaymentException {
    orderRepository.save(order);
    paymentService.charge(order);          // PaymentException (Checked)
    inventoryService.decreaseStock(order); // StockException (Checked)
    // 두 예외 모두 롤백됨
}

// 패턴 2: Unchecked 예외로 전환 (권장)
// Checked Exception을 RuntimeException으로 래핑
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    try {
        paymentService.charge(order);
    } catch (PaymentException e) {
        throw new PaymentFailedException("결제 실패", e); // RuntimeException 하위
        // → 자동 롤백
    }
    try {
        inventoryService.decreaseStock(order);
    } catch (StockException e) {
        throw new InsufficientStockException("재고 부족", e); // RuntimeException 하위
        // → 자동 롤백
    }
}

// 패턴 3: 비즈니스 예외를 처음부터 Unchecked로 설계
// Spring, JPA가 이 방식 사용 — RuntimeException 계층
public class InsufficientStockException extends RuntimeException {
    private final int requested;
    private final int available;

    public InsufficientStockException(int requested, int available) {
        super(String.format("재고 부족: 요청 %d개, 가용 %d개", requested, available));
        this.requested = requested;
        this.available = available;
    }
}

@Transactional  // rollbackFor 불필요 — RuntimeException 자동 롤백
public void decreaseStock(Long productId, int quantity) {
    Product product = productRepository.findById(productId).orElseThrow();
    if (product.getStock() < quantity) {
        throw new InsufficientStockException(quantity, product.getStock());
    }
    product.decreaseStock(quantity);
}
```

### 5. 명시적 롤백 마킹 — setRollbackOnly()

```java
// 예외를 catch로 삼키면서도 트랜잭션을 롤백해야 하는 경우
@Transactional
public OrderResult processOrder(Order order) {
    try {
        orderRepository.save(order);
        paymentService.charge(order);
        return OrderResult.success(order);

    } catch (PaymentException e) {
        // 예외를 삼키고 실패 결과 반환하면서 트랜잭션도 롤백
        log.error("결제 실패: {}", e.getMessage());

        // 명시적 롤백 마킹
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();

        return OrderResult.failure("결제 실패");
        // → 메서드는 정상 반환 (예외 없음)
        // → TransactionInterceptor는 결과 반환 후 commit() 시도
        // → isRollbackOnly() = true → 롤백 후 UnexpectedRollbackException?
        // 아님! setRollbackOnly()는 isRollbackOnly()를 true로 마킹
        // commit()에서 isLocalRollbackOnly() 확인 → processRollback() 실행
        // 단, 호출자는 UnexpectedRollbackException을 받지 않음
        // (isNewTransaction() = true면 정상 롤백 처리)
    }
}
```

### 6. 데이터 오염 사례와 방지

```java
// ❌ 실제 데이터 오염 사례
@Transactional
public void transferMoney(Long fromId, Long toId, int amount) throws InsufficientFundsException {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to = accountRepository.findById(toId).orElseThrow();

    from.withdraw(amount);  // 출금
    accountRepository.save(from);

    to.deposit(amount);     // 입금
    accountRepository.save(to);

    // 이후 감사 로그 → Checked Exception
    auditService.log(fromId, toId, amount); // throws AuditException (Checked)
    // AuditException → 기본 정책: 커밋!
    // → from 출금, to 입금은 DB에 반영됨
    // → 감사 로그만 없음 (데이터 불일치)
}

// ✅ 방지 패턴 1: rollbackFor 명시
@Transactional(rollbackFor = AuditException.class)
public void transferMoney(...) throws InsufficientFundsException, AuditException { ... }

// ✅ 방지 패턴 2: Unchecked로 래핑
@Transactional
public void transferMoney(...) {
    // ...
    try {
        auditService.log(fromId, toId, amount);
    } catch (AuditException e) {
        throw new AuditFailureException("감사 로그 실패", e); // RuntimeException
    }
}

// ✅ 방지 패턴 3: 감사 로그를 REQUIRES_NEW로 분리 (별도 트랜잭션)
// 주 트랜잭션 실패와 무관하게 감사 로그 저장
@Transactional(propagation = REQUIRES_NEW)
public void logAudit(...) { ... }
```

---

## 💻 실험으로 확인하기

### 실험 1: Checked Exception — 기본 동작 확인

```java
@Service
public class RollbackTestService {

    @Transactional
    public void checkedExceptionTest() throws IOException {
        userRepository.save(new User("테스트"));
        throw new IOException("체크예외");
        // 기본 정책: 커밋 → 데이터 저장됨
    }

    @Transactional
    public void uncheckedExceptionTest() {
        userRepository.save(new User("테스트"));
        throw new RuntimeException("언체크예외");
        // 기본 정책: 롤백 → 데이터 저장 안 됨
    }
}

@Test
void defaultRollbackPolicy() throws Exception {
    // Checked: 커밋
    assertThrows(IOException.class, () -> service.checkedExceptionTest());
    assertEquals(1, userRepository.count()); // 저장됨!

    // Unchecked: 롤백
    assertThrows(RuntimeException.class, () -> service.uncheckedExceptionTest());
    assertEquals(1, userRepository.count()); // 추가 저장 안 됨 (롤백)
}
```

### 실험 2: rollbackFor 계층 구조

```java
// 예외 계층
// Exception
//   └── BusinessException
//         ├── OrderException
//         └── PaymentException

@Transactional(rollbackFor = BusinessException.class)
public void test() throws BusinessException { ... }

@Test
void rollbackForHierarchy() {
    // BusinessException → 롤백
    assertThrows(BusinessException.class, () -> service.throwBusiness());
    assertEquals(0, repository.count()); // 롤백

    // OrderException (BusinessException 하위) → 롤백
    assertThrows(OrderException.class, () -> service.throwOrder());
    assertEquals(0, repository.count()); // 롤백 (상속 적용)

    // IOException (BusinessException과 무관) → 커밋!
    assertThrows(IOException.class, () -> service.throwIO());
    assertEquals(1, repository.count()); // 커밋됨!
}
```

### 실험 3: noRollbackFor와 rollbackFor 조합

```java
@Transactional(
    rollbackFor = Exception.class,
    noRollbackFor = SpecificException.class  // SpecificException extends Exception
)
public void combinedRules() throws Exception { ... }

@Test
void combinedRollbackRules() {
    // SpecificException: noRollbackFor 정확 매칭 → 커밋
    assertThrows(SpecificException.class, () -> service.throwSpecific());
    assertEquals(1, repository.count()); // 커밋

    // Exception: rollbackFor 매칭 → 롤백
    assertThrows(Exception.class, () -> service.throwException());
    assertEquals(1, repository.count()); // 롤백 (추가 안 됨)
}
```

---

## ⚡ 성능 임팩트

```
rollbackOn() 실행 비용:
  규칙 목록 순회: O(n), n = rollbackFor + noRollbackFor 개수
  getDepth() 재귀 호출: 예외 계층 깊이에 비례 (통상 5~10 수준)
  합계: ~수 μs → 무시 가능

롤백 vs 커밋 비용 차이:
  커밋: flush() + Connection.commit() → ~수 ms
  롤백: Connection.rollback() → ~수 ms (flush 없음)
  → 롤백이 약간 빠를 수 있음 (flush 생략)
  → 실질적 차이 미미

데이터 오염 비용:
  오염 발생 시 데이터 복구 비용 >> 트랜잭션 비용
  → rollbackFor 누락 실수 하나가 서비스 장애로 이어질 수 있음
  → 비즈니스 예외를 Unchecked로 설계하면 이 위험 제거
```

---

## 🤔 트레이드오프

```
Checked Exception 유지 + rollbackFor 명시:
  ✅ 컴파일 타임에 예외 처리 강제 (호출자가 인지)
  ✅ API 계약이 명확 (메서드 시그니처에 throws 표시)
  ❌ 모든 @Transactional에 rollbackFor 추가 필요
  ❌ 누락 시 조용한 데이터 오염 (런타임 오류 없음)
  → 레거시 코드, 외부 라이브러리 연동 시

Unchecked Exception으로 전환:
  ✅ rollbackFor 불필요 → 자동 롤백
  ✅ Spring/JPA 계층 일관성 (DataAccessException 등 Unchecked)
  ✅ 예외 전파 보일러플레이트 제거
  ❌ 컴파일 타임 강제 없음 (IDE/문서로 보완)
  → 현대 Spring 애플리케이션 권장

비즈니스 예외 설계 원칙:
  "복구 가능한 비즈니스 예외" 라도 Spring 컨텍스트에서는 Unchecked 권장
  이유: Spring 자체 예외 계층(DataAccessException 등)이 모두 Unchecked
       Checked Exception 처리 비용 > Unchecked 명세 비용
  예외: 외부 인터페이스 (API, 라이브러리)와의 계약이 Checked를 요구하는 경우
```

---

## 📌 핵심 정리

```
Spring 기본 롤백 정책
  RuntimeException / Error → 롤백
  Checked Exception (Exception 하위) → 커밋 (예외는 전파)
  근거: Java 설계 철학 (Checked = 복구 가능)

rollbackOn() 규칙 선택 알고리즘
  예외와 각 규칙의 거리(depth) 계산
  가장 구체적인 규칙(depth 가장 작은) 적용
  동점 없음 (클래스 계층은 선형)

rollbackFor / noRollbackFor 적용 우선순위
  더 구체적인 예외 타입이 우선
  SpecificException (depth=0) > Exception (depth=1)

데이터 오염 방지 패턴
  비즈니스 예외 → Unchecked (RuntimeException 하위) 권장
  부득이하게 Checked → rollbackFor 반드시 명시
  예외를 catch로 삼키면서 롤백 → setRollbackOnly() 명시

명시적 롤백 마킹
  TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()
  → 예외 없이 메서드를 반환하면서 트랜잭션 롤백
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional(rollbackFor = Exception.class)`인 메서드에서 `StackOverflowError`가 발생하면 롤백되는가? `Error`는 `Exception`의 하위 클래스가 아닌데 어떻게 처리되는가?

**Q2.** 아래 코드에서 실제로 어떤 일이 벌어지는가?
```java
@Transactional
public void outer() {
    repository.save(data1);
    try { inner(); } catch (RuntimeException e) { /* 무시 */ }
    repository.save(data2);
}

@Transactional  // REQUIRED
public void inner() {
    repository.save(data3);
    throw new RuntimeException("실패");
}
```

**Q3.** `@Transactional(noRollbackFor = RuntimeException.class)`를 설정하면 모든 RuntimeException에서 롤백이 안 되는가? 이 설정이 실무에서 의미있게 사용되는 경우는?

> 💡 **해설**
>
> **Q1.** 롤백된다. `rollbackFor = Exception.class`는 `Exception` 계층에 대한 규칙이지만, `Error`는 `Exception`의 형제(`Throwable`의 하위)이므로 이 규칙에 매칭되지 않는다(`getDepth()` = -1). 하지만 `winner == null`일 때 기본 정책인 `DefaultTransactionAttribute.rollbackOn()`이 적용된다: `return (ex instanceof RuntimeException || ex instanceof Error)`. `StackOverflowError`는 `Error`의 하위이므로 기본 정책에 의해 롤백된다. 즉 `rollbackFor = Exception.class`가 `Error`를 커버하지 않아도, 기본 정책이 `Error`를 항상 롤백 처리한다.
>
> **Q2.** `outer()`가 `UnexpectedRollbackException`을 던지며 실패한다. `inner()`는 `REQUIRED`로 `outer()`의 트랜잭션에 참여한다. `RuntimeException` 발생 시 `TransactionInterceptor`가 `inner()`의 `TransactionStatus.setRollbackOnly()`를 호출해 전역 롤백 마킹을 한다. `outer()`가 `try-catch`로 예외를 잡아도 이 마킹은 해제되지 않는다. `outer()`가 `data2`를 저장하고 정상 반환하면 `commit()`을 시도하지만, `isGlobalRollbackOnly() = true`를 감지하고 롤백 후 `UnexpectedRollbackException`을 던진다. 결과적으로 `data1`, `data2`, `data3` 모두 저장되지 않는다.
>
> **Q3.** 롤백되지 않는다. 단, `noRollbackFor = RuntimeException.class`는 `RuntimeException`과 그 모든 하위 클래스에 대해 롤백하지 않도록 한다. 기본 정책보다 명시적 규칙이 우선하므로 `winner = noRollbackFor` → 커밋된다. 실무에서 이 설정이 의미 있는 경우: 특정 예외를 "예상된 비즈니스 상황"으로 취급하면서 트랜잭션 데이터(예: 실패 기록, 감사 로그)를 보존해야 할 때다. 예를 들어 배치 처리에서 일부 항목 처리 실패 이력을 DB에 저장하면서 전체 배치는 계속 진행하는 경우, 실패 이력 저장 메서드에 `noRollbackFor = BatchItemException.class`를 설정하고 실패를 기록 후 계속 진행할 수 있다. 단 이런 설계는 데이터 오염 위험이 있으므로 신중하게 사용해야 한다.

---

<div align="center">

**[⬅️ 이전: readOnly=true의 실제 효과](./06-readonly-true-effects.md)** | **[다음: TransactionSynchronization 활용 ➡️](./08-transaction-synchronization.md)**

</div>
