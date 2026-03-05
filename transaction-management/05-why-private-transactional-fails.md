# private 메서드에 @Transactional이 안 되는 이유 — CGLIB과 Self-Invocation 함정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- CGLIB 프록시가 `private` 메서드를 오버라이딩하지 못하는 기술적 이유는?
- Self-Invocation(`this.method()`)이 프록시를 우회하는 정확한 메커니즘은?
- `@Transactional`이 무시되는 네 가지 상황과 각각의 근본 원인은?
- Self-Invocation 문제를 해결하는 방법들과 각각의 트레이드오프는?
- `AopContext.currentProxy()`가 왜 코드 냄새(Code Smell)인가?

---

## 🔍 왜 이게 존재하는가

### 문제: @Transactional을 붙였는데 트랜잭션이 적용 안 된다

```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(Order order) {
        validateOrder(order);      // 내부 호출 — 트랜잭션 적용 안 됨!
        saveOrder(order);          // 내부 호출 — 트랜잭션 적용 안 됨!
    }

    @Transactional(propagation = REQUIRES_NEW)  // ← 무시됨
    private void validateOrder(Order order) {   // private — 아예 무시됨
        // 별도 트랜잭션에서 검증하려 했지만 적용 안 됨
    }

    @Transactional  // ← 무시됨 (Self-Invocation)
    public void saveOrder(Order order) {
        orderRepository.save(order);
    }
}
```

```
@Transactional이 동작하는 전제 조건:
  1. Spring이 관리하는 빈(Bean)이어야 함
  2. 외부에서 프록시를 통해 호출되어야 함
  3. public 메서드여야 함 (CGLIB 오버라이딩 가능)
  4. final이 아니어야 함

위 조건 중 하나라도 위배되면 @Transactional은 무시됨
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: private 메서드에 @Transactional을 붙이면 경고라도 난다

```java
// ❌ 컴파일 오류도, 런타임 예외도, 경고도 없음
// 그냥 조용히 무시됨

@Service
public class UserService {

    @Transactional  // ← 완전히 무시, 아무 일도 일어나지 않음
    private void internalProcess() {
        userRepository.save(new User());
        // 트랜잭션 없음 → save()는 SimpleJpaRepository의 @Transactional로 처리됨
        // 하지만 여러 save()를 묶어서 원자적으로 처리하는 건 불가능
    }
}
// → 데이터 불일치가 발생해도 원인 파악이 매우 어려움
```

### Before: @Transactional 메서드에서 다른 @Transactional 메서드를 호출하면 당연히 적용된다

```java
// ❌ Self-Invocation — 같은 클래스 내부 호출
@Service
public class PaymentService {

    @Transactional
    public void processPayment(Payment payment) {
        chargeCard(payment);   // this.chargeCard() — 프록시 우회!
        notifyUser(payment);   // this.notifyUser() — 프록시 우회!
    }

    @Transactional(propagation = REQUIRES_NEW)  // ← 무시됨
    public void chargeCard(Payment payment) { ... }

    @Transactional(propagation = REQUIRES_NEW)  // ← 무시됨
    public void notifyUser(Payment payment) { ... }
    // chargeCard와 notifyUser는 REQUIRED로 processPayment 트랜잭션에 참여
    // REQUIRES_NEW 의도와 완전히 다른 동작
}
```

---

## ✨ 올바른 이해와 패턴

### After: 프록시 동작 원리를 알면 함정이 보인다

```
외부 호출 (정상):
  Client → PaymentService 프록시 → TransactionInterceptor → processPayment()
  ↳ 트랜잭션 시작 → processPayment 실행 → 커밋

Self-Invocation (함정):
  processPayment() 내부에서 this.chargeCard() 호출
  → this = 실제 PaymentService 객체 (프록시가 아님!)
  → TransactionInterceptor를 거치지 않음
  → @Transactional 무시
```

---

## 🔬 내부 동작 원리 — CGLIB과 Self-Invocation 메커니즘

### 1. CGLIB 프록시 생성 방식 — private/final 불가 이유

```java
// CGLIB은 원본 클래스를 상속해 서브클래스를 동적 생성
// 예시: PaymentService에 대해 생성되는 프록시 (개념적)
public class PaymentService$$SpringCGLIB$$0 extends PaymentService {

    private TransactionInterceptor transactionInterceptor;

    // ✅ public 메서드는 오버라이딩 가능 → @Transactional 적용
    @Override
    public void processPayment(Payment payment) {
        // TransactionInterceptor 호출
        transactionInterceptor.invoke(() -> super.processPayment(payment));
    }

    // ✅ public chargeCard도 오버라이딩 가능
    @Override
    public void chargeCard(Payment payment) {
        transactionInterceptor.invoke(() -> super.chargeCard(payment));
    }

    // ❌ private 메서드는 Java 언어 규칙상 오버라이딩 불가
    // private void internalProcess() → CGLIB이 오버라이딩 못함 → @Transactional 무시

    // ❌ final 메서드도 오버라이딩 불가
    // public final void criticalProcess() → CGLIB 오버라이딩 불가 → @Transactional 무시
}
```

```
Java 상속 규칙:
  private 메서드: 서브클래스에서 보이지 않음 → 오버라이딩 불가
  final 메서드: 오버라이딩 금지
  → CGLIB 서브클래스도 이 규칙을 따름
  → 결과적으로 @Transactional 어노테이션이 있어도 프록시 로직 삽입 불가
```

### 2. Self-Invocation 메커니즘 — 왜 프록시를 우회하는가

```java
// 실제 객체 참조 구조
@SpringBootApplication
public class App {
    @Autowired
    PaymentService paymentService;
    // paymentService = PaymentService$$SpringCGLIB$$0 (프록시)
}

// 프록시 내부의 target 참조
class PaymentService$$SpringCGLIB$$0 extends PaymentService {
    private PaymentService target = new PaymentService(); // 실제 객체
    // (단순화된 표현, 실제로는 상속 구조)
}

// processPayment 실행 시 스택
// [1] 외부: paymentService.processPayment(order)
//     → 프록시의 processPayment() 호출
//     → TransactionInterceptor 실행 → 트랜잭션 시작
//     → super.processPayment(order) 호출 (실제 메서드)

// [2] 실제 PaymentService.processPayment() 내부:
//     this.chargeCard(payment)
//     → this = 실제 PaymentService 인스턴스 (프록시가 아님!)
//     → 직접 chargeCard() 호출 → TransactionInterceptor 없음
//     → @Transactional(REQUIRES_NEW) 무시
```

### 3. @Transactional이 무시되는 네 가지 상황

```java
// [상황 1] private 메서드
@Transactional          // 무시
private void privateMethod() { }
// 이유: CGLIB이 오버라이딩 불가

// [상황 2] final 메서드
@Transactional          // 무시
public final void finalMethod() { }
// 이유: CGLIB이 오버라이딩 불가

// [상황 3] Self-Invocation
@Transactional
public void outerMethod() {
    this.innerMethod(); // 무시 — 프록시 우회
}
@Transactional(propagation = REQUIRES_NEW)
public void innerMethod() { }

// [상황 4] Spring 빈이 아닌 객체
public class NotABean {
    @Transactional          // 무시 — Spring이 관리하지 않는 객체
    public void process() { }
}
NotABean obj = new NotABean(); // new로 직접 생성
obj.process(); // @Transactional 무시
```

### 4. Self-Invocation 해결 방법 3가지

#### 방법 1: 별도 빈으로 분리 (권장)

```java
// ✅ 가장 깔끔한 해결책 — 메서드를 별도 빈으로 추출
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final CardChargeService cardChargeService;    // 별도 빈
    private final NotificationService notificationService; // 별도 빈

    @Transactional
    public void processPayment(Payment payment) {
        cardChargeService.chargeCard(payment);   // 프록시 경유 → REQUIRES_NEW 적용
        notificationService.notifyUser(payment); // 프록시 경유 → REQUIRES_NEW 적용
    }
}

@Service
public class CardChargeService {
    @Transactional(propagation = REQUIRES_NEW)  // ✅ 실제로 적용됨
    public void chargeCard(Payment payment) { ... }
}
```

#### 방법 2: AopContext.currentProxy() — 안티패턴

```java
// ❌ 동작은 하지만 코드 냄새 (Code Smell)
@Service
public class PaymentService {

    @Transactional
    public void processPayment(Payment payment) {
        // 현재 프록시를 통해 자기 자신을 호출
        PaymentService proxy = (PaymentService) AopContext.currentProxy();
        proxy.chargeCard(payment);  // 프록시 경유 → @Transactional 적용
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void chargeCard(Payment payment) { ... }
}

// 필수 설정: exposeProxy = true 없으면 AopContext.currentProxy() 예외
@EnableTransactionManagement
// 또는:
@EnableAspectJAutoProxy(exposeProxy = true)

// 문제점:
// - 비즈니스 코드가 AOP 인프라에 직접 의존
// - 테스트 시 AopContext 설정 필요
// - 코드 의도가 불명확
```

#### 방법 3: Self 주입 (Self-Autowiring)

```java
// Self 주입 — 자기 자신의 프록시를 주입받아 사용
@Service
public class PaymentService {

    @Autowired
    @Lazy  // 순환 의존성 방지
    private PaymentService self;  // 프록시가 주입됨

    @Transactional
    public void processPayment(Payment payment) {
        self.chargeCard(payment);  // self = 프록시 → @Transactional 적용
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void chargeCard(Payment payment) { ... }
}

// 문제점:
// - 순환 의존성 (Spring이 경고)
// - @Lazy 없으면 빈 생성 실패
// - 의도가 명확하지 않음
// → 방법 1(별도 빈 분리)이 훨씬 낫다
```

#### 방법 4: AspectJ 위빙 — Self-Invocation 근본 해결

```java
// AspectJ는 프록시 패턴이 아닌 바이트코드 위빙 방식
// → Self-Invocation도 적용됨 (클래스 로드 시 바이트코드 수정)

@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)
// + spring-aspects 의존성
// + 로드 타임 위빙 또는 컴파일 타임 위빙 설정 필요

// 장점: private 메서드, Self-Invocation 모두 해결
// 단점: 빌드 설정 복잡, 디버깅 어려움, 학습 비용
// → 특수한 경우에만 사용
```

### 5. 올바른 설계로 Self-Invocation 애초에 피하기

```java
// ❌ Self-Invocation이 발생하는 설계
@Service
public class OrderService {
    @Transactional
    public void processOrder(Order order) {
        // 여러 단계를 하나의 클래스 안에서 내부 호출로 처리
        validateOrder(order);
        applyDiscount(order);
        saveOrder(order);
        sendNotification(order);
    }

    @Transactional(REQUIRES_NEW) public void sendNotification(...) { }
    private void validateOrder(...) { }
    private void applyDiscount(...) { }
    private void saveOrder(...) { }
}

// ✅ 책임 분리로 Self-Invocation 구조 자체를 제거
@Service @RequiredArgsConstructor
public class OrderService {

    private final OrderValidator orderValidator;
    private final DiscountService discountService;
    private final OrderRepository orderRepository;
    private final NotificationService notificationService; // REQUIRES_NEW

    @Transactional
    public void processOrder(Order order) {
        orderValidator.validate(order);          // 별도 빈
        discountService.apply(order);            // 별도 빈
        orderRepository.save(order);
        notificationService.notify(order);       // 별도 빈 → REQUIRES_NEW 적용
    }
}
// → @Transactional 함정도 없고, 단일 책임도 지킴
```

---

## 💻 실험으로 확인하기

### 실험 1: Self-Invocation 트랜잭션 미적용 확인

```java
@Service
public class TestService {

    @Transactional
    public void outer() {
        System.out.println("outer 트랜잭션: " +
            TransactionSynchronizationManager.isActualTransactionActive()); // true
        inner(); // Self-Invocation
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void inner() {
        System.out.println("inner 트랜잭션 이름: " +
            TransactionSynchronizationManager.getCurrentTransactionName());
        // outer와 동일한 트랜잭션 이름 출력 → REQUIRES_NEW 미적용 확인
    }
}
```

### 실험 2: private 메서드 @Transactional 무시 확인

```java
@Service
public class SaveService {

    @Transactional
    public void publicSave(Data data) {
        dataRepository.save(data);
        throw new RuntimeException("롤백");
    }

    @Transactional  // 무시됨
    private void privateSave(Data data) {
        dataRepository.save(data);
        throw new RuntimeException("롤백 안 됨?");
    }

    public void callPrivate(Data data) {
        privateSave(data); // @Transactional 없이 실행
        // → 예외 발생해도 save()는 이미 커밋됨 (SimpleJpaRepository의 @Transactional)
    }
}

@Test
void privateTransactionalIsIgnored() {
    assertThrows(RuntimeException.class, () -> saveService.callPrivate(new Data()));
    // data는 DB에 저장됨 — privateSave의 @Transactional이 무시됐기 때문
    assertEquals(1, dataRepository.count()); // 롤백 안 됨!
}
```

### 실험 3: 별도 빈 분리 후 REQUIRES_NEW 적용 확인

```java
@Test
void requiresNewAppliedWithSeparateBean() {
    assertThrows(RuntimeException.class, () -> outerService.process());
    // outerService 롤백됨
    // innerService(REQUIRES_NEW)는 이미 커밋됨
    assertEquals(0, outerRepository.count()); // outer 롤백
    assertEquals(1, innerRepository.count()); // inner 커밋 유지
}
```

---

## ⚡ 성능 임팩트

```
Self-Invocation 문제의 성능 영향:

직접적 성능 영향: 없음
  (Self-Invocation이 트랜잭션을 무시하므로 오히려 오버헤드 없음)

간접적 성능 영향: 심각
  여러 save()가 별도 트랜잭션으로 실행됨
  → Connection 획득/해제 반복
  → Dirty Checking 실패 → 데이터 불일치
  → 버그 추적 비용 (성능보다 더 큰 문제)

해결 방법별 오버헤드:
  별도 빈 분리:      추가 빈 생성 비용 (~수 ms 시작 시) → 런타임 오버헤드 없음
  AopContext:        ThreadLocal 조회 (~수백 ns) → 무시 가능
  Self-Autowiring:   순환 의존성 처리 (~시작 시) → 런타임 오버헤드 없음
  AspectJ:           바이트코드 위빙 (빌드 시) → 런타임 오버헤드 최소
```

---

## 🤔 트레이드오프

```
Self-Invocation 해결 방법 비교:

별도 빈 분리 (권장):
  ✅ 깔끔, 단일 책임 원칙 준수
  ✅ 테스트 용이 (각 빈을 독립적으로 테스트)
  ✅ AOP 인프라에 코드 미의존
  ❌ 파일/클래스 수 증가
  → 대부분의 경우 정답

AopContext.currentProxy():
  ✅ 별도 클래스 불필요 (빠른 수정)
  ❌ 비즈니스 코드가 AOP 인프라에 직접 의존
  ❌ exposeProxy=true 전역 설정 필요
  ❌ 테스트 시 추가 설정 필요
  → 레거시 수정 시 임시 방편으로만

Self-Autowiring (@Lazy):
  ✅ 별도 클래스 불필요
  ❌ 순환 의존성 (Spring 경고)
  ❌ @Lazy로 인한 빈 초기화 지연
  → 피하는 게 좋음

AspectJ 위빙:
  ✅ private, final, Self-Invocation 모두 해결
  ❌ 빌드 설정 복잡 (컴파일/로드 타임 위빙)
  ❌ IDE 지원 제한적
  ❌ 학습 비용
  → 특수 요구사항이 있을 때만
```

---

## 📌 핵심 정리

```
@Transactional이 무시되는 4가지 상황
  private 메서드: CGLIB 오버라이딩 불가 (Java 상속 규칙)
  final 메서드:   CGLIB 오버라이딩 불가
  Self-Invocation: this.method() → 프록시 우회 (실제 객체 직접 호출)
  비Spring 빈:    프록시 생성 안 됨

Self-Invocation 원인
  CGLIB 프록시 = PaymentService를 상속한 서브클래스
  super.processPayment() 내부에서 this = 실제 PaymentService 인스턴스
  → TransactionInterceptor를 거치지 않음

해결 방법 우선순위
  1. 별도 빈 분리 (단일 책임, 테스트 용이)
  2. AopContext.currentProxy() (임시 방편, 인프라 의존)
  3. AspectJ 위빙 (근본 해결, 복잡한 설정)

올바른 설계 방향
  Self-Invocation이 필요한 설계 → 책임 분리 신호
  @Transactional 함정 = 설계 개선 기회
  private 메서드로 비즈니스 로직 → public 메서드로 변경 or 별도 빈
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional`이 붙은 `public` 메서드가 같은 클래스의 `private` 메서드를 호출하고, 그 `private` 메서드 안에서 RuntimeException이 발생한다. 이때 `public` 메서드의 트랜잭션은 롤백되는가?

**Q2.** `@Transactional` 클래스를 `final`로 선언하면 어떤 일이 발생하는가? Spring Boot에서 Kotlin을 사용할 때 자주 만나는 이 문제의 해결책은?

**Q3.** Self-Invocation으로 인해 `REQUIRES_NEW`가 무시되어 내부 메서드가 외부 트랜잭션에 참여하게 됐다. 이때 내부 메서드에서 예외가 발생하고 외부 메서드에서 `try-catch`로 잡는다면 실제로 어떤 일이 벌어지는가?

> 💡 **해설**
>
> **Q1.** 롤백된다. `private` 메서드는 `@Transactional`이 무시되지만, 해당 메서드에서 발생한 `RuntimeException`은 호출 스택을 타고 `public` 메서드로 전파된다. `public` 메서드의 `@Transactional`은 정상 적용되어 있으므로 `TransactionInterceptor`가 예외를 감지하고 트랜잭션을 롤백한다. 중요한 것은 `@Transactional` 어노테이션의 무시가 "트랜잭션 경계 선언"의 무시이지, "예외 전파"의 차단이 아니라는 점이다. `private` 메서드 자체는 이미 실행 중인 트랜잭션 컨텍스트 안에서 동작한다.
>
> **Q2.** `final` 클래스는 CGLIB이 상속할 수 없으므로 프록시 생성에 실패한다. Spring Boot 애플리케이션 시작 시 `BeanCreationException`이 발생한다. Kotlin에서는 모든 클래스가 기본적으로 `final`이므로 Spring의 CGLIB 프록시 생성이 실패하는 문제가 자주 발생한다. 해결책은 두 가지다: (1) `open` 키워드를 클래스와 메서드에 명시적으로 추가한다 — 하지만 모든 `@Service`, `@Component` 클래스에 추가해야 하므로 번거롭다. (2) `kotlin-spring` Gradle 플러그인(`org.jetbrains.kotlin.plugin.spring`)을 사용하면 `@Component`, `@Transactional`, `@Async` 등 Spring 어노테이션이 붙은 클래스와 메서드에 자동으로 `open`을 추가해준다 — Spring Initializr에서 Kotlin을 선택하면 기본으로 포함된다.
>
> **Q3.** `REQUIRES_NEW`가 무시됐으므로 내부 메서드는 외부 트랜잭션에 `REQUIRED`로 참여하고 있다. 내부에서 `RuntimeException`이 발생하면 `TransactionInterceptor`가 이를 감지한다. 내부 메서드는 `isNewTransaction() = false`(기존 참여)이므로 직접 롤백하지 않고 `TransactionStatus.setRollbackOnly()`를 호출해 전역 롤백 마킹만 한다. 외부 메서드에서 `try-catch`로 예외를 잡아도 이 마킹은 취소되지 않는다. 외부 메서드가 정상 완료되어 `commit()`을 시도하는 순간 `isGlobalRollbackOnly() = true`를 감지하고 롤백 후 `UnexpectedRollbackException`을 던진다. 결국 전체 트랜잭션이 롤백되며, 호출자는 의도치 않은 `UnexpectedRollbackException`을 받게 된다.

---

<div align="center">

**[⬅️ 이전: Isolation Level과 Database Lock](./04-isolation-level-lock.md)** | **[다음: readOnly=true의 실제 효과 ➡️](./06-readonly-true-effects.md)**

</div>
