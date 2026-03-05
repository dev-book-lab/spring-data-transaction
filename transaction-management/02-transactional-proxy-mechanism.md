# @Transactional 프록시 생성 메커니즘 — TransactionInterceptor와 AOP 체인

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Transactional`이 붙은 메서드 호출 시 내부적으로 어떤 순서로 무슨 일이 벌어지는가?
- `TransactionInterceptor`는 어떻게 AOP Advice 체인에 등록되는가?
- `TransactionAttributeSource`는 `@Transactional` 어노테이션을 어떻게 파싱하고 캐시하는가?
- 트랜잭션 시작 → 메서드 실행 → 커밋/롤백까지 정확한 코드 실행 경로는?
- `@EnableTransactionManagement`와 `@Transactional` 단독 사용의 차이는?

---

## 🔍 왜 이게 존재하는가

### 문제: 트랜잭션 경계 코드가 비즈니스 로직을 오염시킨다

```java
// 트랜잭션 수동 처리 — 비즈니스 로직과 인프라 코드가 뒤섞임
public void transferMoney(Long fromId, Long toId, int amount) {
    TransactionStatus status = txManager.getTransaction(new DefaultTransactionDefinition());
    try {
        Account from = accountRepository.findById(fromId).orElseThrow();
        Account to   = accountRepository.findById(toId).orElseThrow();
        from.withdraw(amount);   // 비즈니스 로직
        to.deposit(amount);      // 비즈니스 로직
        txManager.commit(status);
    } catch (Exception e) {
        txManager.rollback(status);
        throw e;
    }
}
```

```
AOP 기반 선언적 트랜잭션의 해결:
  @Transactional 어노테이션 하나로 트랜잭션 경계 선언
  프록시가 메서드 전후에 트랜잭션 시작/커밋/롤백을 자동 처리
  비즈니스 로직은 순수하게 유지

@Transactional
public void transferMoney(Long fromId, Long toId, int amount) {
    Account from = accountRepository.findById(fromId).orElseThrow();
    Account to   = accountRepository.findById(toId).orElseThrow();
    from.withdraw(amount);
    to.deposit(amount);
    // 트랜잭션 시작/커밋/롤백은 프록시가 담당
}
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Transactional을 붙이기만 하면 항상 트랜잭션이 적용된다

```java
// ❌ 트랜잭션이 적용되지 않는 경우들

// 1. @EnableTransactionManagement 누락
// → ProxyTransactionManagementConfiguration 미등록 → 프록시 미생성

// 2. 같은 클래스 내부 메서드 호출 (Self-Invocation)
@Service
public class OrderService {
    public void processOrder(Order order) {
        saveOrder(order);  // 프록시 우회! this.saveOrder() 호출
    }

    @Transactional  // ← 적용 안 됨
    public void saveOrder(Order order) {
        orderRepository.save(order);
    }
}

// 3. private 메서드
@Transactional  // ← CGLIB 오버라이딩 불가 → 무시됨
private void internalSave(Order order) { ... }

// 4. final 메서드
@Transactional  // ← CGLIB 서브클래스에서 오버라이딩 불가
public final void criticalSave(Order order) { ... }
```

### Before: @Transactional 어노테이션은 메서드 실행마다 매번 파싱된다

```java
// ❌ 잘못된 이해: 호출마다 리플렉션으로 @Transactional을 읽어옴

// ✅ 실제:
// TransactionAttributeSource가 파싱 결과를 캐시
// Map<MethodCacheKey, TransactionAttribute> 형태로 저장
// 두 번째 호출부터는 캐시에서 꺼냄 → 파싱 비용 없음
```

---

## ✨ 올바른 이해와 패턴

### After: @Transactional 처리 전체 흐름

```
[애플리케이션 시작 시]
@EnableTransactionManagement
  → ProxyTransactionManagementConfiguration 등록
  → BeanFactoryTransactionAttributeSourceAdvisor 등록
  → TransactionInterceptor (Advice) 등록
  → @Transactional이 붙은 빈 → CGLIB 프록시 생성

[런타임 메서드 호출 시]
클라이언트 → 프록시.method()
  → TransactionInterceptor.invoke()
  → TransactionAttributeSource에서 @Transactional 속성 조회 (캐시)
  → PlatformTransactionManager.getTransaction() → 트랜잭션 시작
  → 실제 메서드 실행
  → 예외 없음 → commit()
  → RuntimeException → rollback()
```

---

## 🔬 내부 동작 원리 — @Transactional 소스 추적

### 1. @EnableTransactionManagement → 설정 체인 활성화

```java
// @EnableTransactionManagement
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
    boolean proxyTargetClass() default false;  // CGLIB 강제 여부
    AdviceMode mode() default AdviceMode.PROXY; // PROXY vs ASPECTJ
}

// TransactionManagementConfigurationSelector
class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<...> {
    @Override
    protected String[] selectImports(AdviceMode adviceMode) {
        return switch (adviceMode) {
            case PROXY -> new String[] {
                AutoProxyRegistrar.class.getName(),
                ProxyTransactionManagementConfiguration.class.getName()
            };
            case ASPECTJ -> new String[] {
                // AspectJ 위빙 방식 (바이트코드 직접 수정)
                TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME
            };
        };
    }
}

// ProxyTransactionManagementConfiguration — 핵심 빈 3개 등록
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

    // [1] Advisor = Pointcut + Advice
    @Bean
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(...) {
        BeanFactoryTransactionAttributeSourceAdvisor advisor =
            new BeanFactoryTransactionAttributeSourceAdvisor();
        advisor.setTransactionAttributeSource(transactionAttributeSource());
        advisor.setAdvice(transactionInterceptor());           // TransactionInterceptor 등록
        advisor.setOrder(this.enableTx.getNumber("order"));
        return advisor;
    }

    // [2] @Transactional 파싱 및 캐시
    @Bean
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource();
        // @Transactional → TransactionAttribute 변환 + 캐시
    }

    // [3] 실제 트랜잭션 처리 Advice
    @Bean
    public TransactionInterceptor transactionInterceptor(...) {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionAttributeSource(transactionAttributeSource());
        interceptor.setTransactionManager(txManager);
        return interceptor;
    }
}
```

### 2. 프록시 생성 — @Transactional 빈 감지

```java
// InfrastructureAdvisorAutoProxyCreator (AutoProxyRegistrar가 등록)
// → AbstractAdvisorAutoProxyCreator 상속
// → BeanPostProcessor로 동작

@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    // 빈이 생성된 후 Advisor가 적용되는지 확인
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

    if (specificInterceptors != DO_NOT_PROXY) {
        // @Transactional 메서드가 있으면 프록시 생성
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, ...);
        return proxy;  // 원본 빈 대신 프록시 반환
    }
    return bean;
}

// Advisor 매칭 — Pointcut이 @Transactional을 감지
// BeanFactoryTransactionAttributeSourceAdvisor.Pointcut:
//   클래스 또는 메서드에 @Transactional이 있으면 → 프록시 대상
```

### 3. TransactionInterceptor.invoke() — 트랜잭션 실행 핵심

```java
// TransactionInterceptor — MethodInterceptor 구현
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor {

    @Override
    @Nullable
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Class<?> targetClass = invocation.getThis() != null
            ? AopUtils.getTargetClass(invocation.getThis()) : null;

        // 핵심: invokeWithinTransaction() 위임
        return invokeWithinTransaction(
            invocation.getMethod(),
            targetClass,
            new CoroutinesInvocationCallback() {
                @Override
                public Object proceedWithInvocation() throws Throwable {
                    return invocation.proceed();  // 실제 메서드 실행
                }
            }
        );
    }
}

// TransactionAspectSupport.invokeWithinTransaction() — 실제 트랜잭션 처리
protected Object invokeWithinTransaction(Method method, Class<?> targetClass,
                                          InvocationCallback invocation) throws Throwable {

    // [1] @Transactional 속성 조회 (캐시)
    TransactionAttributeSource tas = getTransactionAttributeSource();
    TransactionAttribute txAttr = tas.getTransactionAttribute(method, targetClass);
    // txAttr: propagation, isolation, timeout, readOnly, rollbackFor 등

    // [2] TransactionManager 결정
    TransactionManager tm = determineTransactionManager(txAttr);
    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);

    String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    // [3] 트랜잭션 시작 (또는 기존 참여)
    TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
    // → ptm.getTransaction(txAttr) 호출
    // → AbstractPlatformTransactionManager.getTransaction() → Propagation 분기

    Object retVal;
    try {
        // [4] 실제 메서드 실행
        retVal = invocation.proceedWithInvocation();

    } catch (Throwable ex) {
        // [5] 예외 발생 → 롤백 규칙 적용
        completeTransactionAfterThrowing(txInfo, ex);
        // → txAttr.rollbackOn(ex) 판단
        // → true: ptm.rollback(status)
        // → false: ptm.commit(status) (예외는 재throw)
        throw ex;
    } finally {
        // [6] TransactionInfo 정리 (ThreadLocal 복원)
        cleanupTransactionInfo(txInfo);
    }

    // [7] 정상 완료 → 커밋
    commitTransactionAfterReturning(txInfo);
    // → ptm.commit(status)
    return retVal;
}
```

### 4. TransactionAttributeSource — @Transactional 파싱 및 캐시

```java
// AnnotationTransactionAttributeSource — @Transactional 어노테이션 처리
public class AnnotationTransactionAttributeSource extends AbstractFallbackTransactionAttributeSource {

    // 파싱 결과 캐시: 메서드 → TransactionAttribute
    // AbstractFallbackTransactionAttributeSource에 정의됨
    private final Map<Object, TransactionAttribute> attributeCache = new ConcurrentHashMap<>(1024);

    @Override
    @Nullable
    public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
        // 1. 캐시 조회
        Object cacheKey = getCacheKey(method, targetClass);
        TransactionAttribute cached = attributeCache.get(cacheKey);

        if (cached != null) {
            return cached == NULL_TRANSACTION_ATTRIBUTE ? null : cached;
        }

        // 2. 캐시 미스 → 파싱
        TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
        // → 메서드 레벨 @Transactional 탐색
        // → 없으면 클래스 레벨 탐색
        // → 없으면 인터페이스 메서드 탐색

        // 3. 캐시 저장
        attributeCache.put(cacheKey, txAttr != null ? txAttr : NULL_TRANSACTION_ATTRIBUTE);
        return txAttr;
    }

    // @Transactional 탐색 우선순위 (AbstractFallbackTransactionAttributeSource)
    // 1. 구체 클래스의 메서드 레벨 @Transactional
    // 2. 구체 클래스의 클래스 레벨 @Transactional
    // 3. 선언된 인터페이스의 메서드 레벨 @Transactional
    // 4. 선언된 인터페이스의 클래스 레벨 @Transactional
}

// SpringTransactionAnnotationParser — @Transactional → RuleBasedTransactionAttribute 변환
class SpringTransactionAnnotationParser implements TransactionAnnotationParser {

    @Override
    public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
        AnnotationAttributes attributes =
            AnnotatedElementUtils.findMergedAnnotationAttributes(element, Transactional.class, ...);

        if (attributes == null) return null;

        RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();
        rbta.setPropagationBehavior(attributes.getEnum("propagation").value());
        rbta.setIsolationLevel(attributes.getEnum("isolation").value());
        rbta.setTimeout(attributes.getNumber("timeout").intValue());
        rbta.setReadOnly(attributes.getBoolean("readOnly"));

        // rollbackFor, noRollbackFor → RollbackRuleAttribute 목록
        List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
        for (Class<?> clazz : attributes.getClassArray("rollbackFor")) {
            rollbackRules.add(new RollbackRuleAttribute(clazz));
        }
        for (Class<?> clazz : attributes.getClassArray("noRollbackFor")) {
            rollbackRules.add(new NoRollbackRuleAttribute(clazz));
        }
        rbta.setRollbackRules(rollbackRules);

        return rbta;
    }
}
```

### 5. completeTransactionAfterThrowing — 롤백 결정 로직

```java
// 예외 발생 시 롤백 여부 결정
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {

    if (txInfo != null && txInfo.getTransactionStatus() != null) {

        // rollbackOn() — 이 예외에서 롤백해야 하는가?
        if (txInfo.transactionAttribute != null
                && txInfo.transactionAttribute.rollbackOn(ex)) {
            // 롤백
            txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
        } else {
            // 롤백 안 함 — commit (예외는 상위로 전파)
            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }
    }
}

// RuleBasedTransactionAttribute.rollbackOn() — 롤백 규칙 평가
@Override
public boolean rollbackOn(Throwable ex) {
    RollbackRuleAttribute winner = null;
    int deepest = Integer.MAX_VALUE;

    // rollbackFor / noRollbackFor 규칙 중 가장 구체적인 것 선택
    for (RollbackRuleAttribute rule : this.rollbackRules) {
        int depth = rule.getDepth(ex);  // 예외 계층에서의 깊이
        if (depth >= 0 && depth < deepest) {
            deepest = depth;
            winner = rule;
        }
    }

    // 규칙이 없으면 기본값: RuntimeException / Error면 롤백
    if (winner == null) {
        return super.rollbackOn(ex);
        // DefaultTransactionAttribute.rollbackOn():
        // return (ex instanceof RuntimeException || ex instanceof Error)
    }

    return !(winner instanceof NoRollbackRuleAttribute);
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 프록시 타입 확인

```java
@Autowired
OrderService orderService;  // 실제로는 프록시

@Test
void transactionalProxyType() {
    System.out.println(orderService.getClass().getName());
    // com.example.OrderService$$SpringCGLIB$$0 (CGLIB 프록시)

    System.out.println(AopUtils.isCglibProxy(orderService));  // true
    System.out.println(AopUtils.isAopProxy(orderService));    // true

    // 실제 대상 객체 꺼내기
    OrderService target = (OrderService) ((Advised) orderService).getTargetSource().getTarget();
    System.out.println(target.getClass().getName());  // com.example.OrderService
}
```

### 실험 2: 트랜잭션 실행 로그 확인

```yaml
logging:
  level:
    org.springframework.transaction: TRACE  # 트랜잭션 상세 로그
    org.springframework.orm.jpa: DEBUG
```

```
# 메서드 호출 시 출력되는 로그:
DEBUG TransactionInterceptor - Getting transaction for [com.example.OrderService.placeOrder]
DEBUG JpaTransactionManager  - Creating new transaction with name [...]: PROPAGATION_REQUIRED,...
DEBUG JpaTransactionManager  - Opened new EntityManager [...] for JPA transaction
DEBUG JpaTransactionManager  - Committing JPA transaction on EntityManager [...]
DEBUG TransactionInterceptor - Completing transaction for [com.example.OrderService.placeOrder]
```

### 실험 3: TransactionAttributeSource 캐시 확인

```java
@Autowired
TransactionAttributeSource transactionAttributeSource;

@Test
void attributeCaching() throws NoSuchMethodException {
    Method method = OrderService.class.getMethod("placeOrder", Order.class);

    // 첫 번째 조회 — 파싱 발생
    long start = System.nanoTime();
    TransactionAttribute attr1 = transactionAttributeSource.getTransactionAttribute(method, OrderService.class);
    long first = System.nanoTime() - start;

    // 두 번째 조회 — 캐시 히트
    start = System.nanoTime();
    TransactionAttribute attr2 = transactionAttributeSource.getTransactionAttribute(method, OrderService.class);
    long second = System.nanoTime() - start;

    System.out.printf("첫 번째: %dns, 두 번째: %dns%n", first, second);
    // 첫 번째: ~수천 ns, 두 번째: ~수백 ns (캐시 히트)
    assertSame(attr1, attr2);  // 동일 객체 참조
}
```

---

## ⚡ 성능 임팩트

```
@Transactional 프록시 오버헤드 (메서드 호출 당):

TransactionInterceptor.invoke():      ~수 μs
  - TransactionAttributeSource 캐시 조회: ~수백 ns
  - determineTransactionManager():    ~수백 ns
  - createTransactionIfNecessary():   ~수십~수백 μs (실제 트랜잭션 시작 포함)
  - CGLIB 프록시 메서드 디스패치:     ~수십 ns
  합계: 실제 트랜잭션 시작/종료 비용 포함 시 수백 μs~수 ms

트랜잭션 비용의 대부분:
  doBegin(): Connection 획득, setAutoCommit(false) → ~수십~수백 μs
  doCommit(): flush() + Connection.commit() → ~수 ms
  프록시 자체 오버헤드: ~수 μs (전체의 1% 미만)

캐시 효과:
  TransactionAttribute 캐시 → 파싱 비용 제거
  methodCache (AbstractAdvisorAutoProxyCreator) → Advisor 매칭 비용 제거
  → 두 번째 호출부터 오버헤드 최소화
```

---

## 🤔 트레이드오프

```
선언적 @Transactional (프록시 기반):
  장점: 코드 간결, 비즈니스 로직과 트랜잭션 분리
  단점: Self-Invocation 함정, private/final 불가, 프록시 오버헤드

프로그래밍 방식 (TransactionTemplate):
  장점: 조건부 트랜잭션, 루프 내 제어, Self-Invocation 회피
  단점: 코드 장황, 트랜잭션 로직이 비즈니스 코드에 섞임

AspectJ 위빙 방식 (@EnableTransactionManagement(mode=ASPECTJ)):
  장점: Self-Invocation 문제 없음, private/final 가능
  단점: 빌드 설정 복잡, 디버깅 어려움

@Transactional 위치:
  인터페이스에 선언 → 구현체에 프록시 적용 (CGLIB 환경에서 주의)
  구현 클래스에 선언 → 명확하고 안전 (권장)
  클래스 레벨 선언 → 모든 public 메서드에 적용 (SimpleJpaRepository 패턴)
```

---

## 📌 핵심 정리

```
프록시 생성 경로
  @EnableTransactionManagement
    → ProxyTransactionManagementConfiguration
    → BeanFactoryTransactionAttributeSourceAdvisor (Pointcut + TransactionInterceptor)
    → @Transactional 빈 감지 → CGLIB 프록시 생성

메서드 호출 경로
  프록시.method()
    → TransactionInterceptor.invoke()
    → invokeWithinTransaction()
      → TransactionAttributeSource.getTransactionAttribute() [캐시]
      → PlatformTransactionManager.getTransaction()
      → 실제 메서드 실행
      → commit() 또는 rollback()

@Transactional 탐색 우선순위
  1. 구체 클래스 메서드 레벨
  2. 구체 클래스 클래스 레벨
  3. 인터페이스 메서드 레벨
  4. 인터페이스 클래스 레벨

롤백 결정
  RuntimeException / Error → 기본 롤백
  Checked Exception → 기본 커밋 (rollbackFor로 변경 가능)
  RuleBasedTransactionAttribute.rollbackOn() → 가장 구체적인 규칙 적용
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional`이 인터페이스에 선언되어 있고 Spring Boot의 기본 설정(CGLIB)을 사용할 때, 트랜잭션이 정상 적용되는가? JDK Proxy와 CGLIB에서 차이가 있는가?

**Q2.** `@Transactional` 메서드 내부에서 `new Thread(() -> anotherTransactionalMethod()).start()`를 호출하면 어떻게 되는가? 트랜잭션이 새 스레드에 전파되는가?

**Q3.** `cleanupTransactionInfo(txInfo)`는 finally 블록에서 실행된다. 이것이 `completeTransactionAfterThrowing()`에서 commit이 호출되고 예외가 발생했을 때 어떤 순서로 실행되며, TransactionInfo 정리와 커밋/롤백 실패가 겹치면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** CGLIB은 클래스를 상속해 서브클래스를 생성한다. 인터페이스에 선언된 `@Transactional`은 인터페이스 메서드에 붙어있으므로, CGLIB이 생성한 서브클래스가 이 메서드를 오버라이딩할 때 `AnnotationTransactionAttributeSource`가 인터페이스 메서드의 `@Transactional`까지 탐색한다(탐색 우선순위 3, 4번). 따라서 CGLIB 환경에서도 인터페이스의 `@Transactional`은 적용된다. 단 Spring 공식 가이드는 구현 클래스에 선언하는 것을 권장한다 — 인터페이스의 `@Transactional`은 JDK Proxy 환경에서만 보장되며, AOP 프록시 유형이 바뀌면 동작이 달라질 수 있기 때문이다.
>
> **Q2.** 트랜잭션은 새 스레드에 전파되지 않는다. `PlatformTransactionManager`는 `TransactionSynchronizationManager`의 `ThreadLocal`에 트랜잭션 컨텍스트를 저장한다. 새 스레드는 별도의 ThreadLocal을 갖기 때문에 트랜잭션 컨텍스트가 없다. 새 스레드에서 `@Transactional` 메서드를 호출하면 Propagation이 `REQUIRED`라면 새 트랜잭션을 시작한다. 원래 스레드의 트랜잭션과는 완전히 별개이므로 두 트랜잭션은 서로 독립적으로 커밋/롤백된다.
>
> **Q3.** 실행 순서: `completeTransactionAfterThrowing()` 내에서 커밋이 실패하면 `TransactionException`이 발생한다. 이 예외는 catch 블록을 벗어나 finally 블록으로 진입하고, `cleanupTransactionInfo(txInfo)`가 실행된다. `cleanupTransactionInfo()`는 ThreadLocal에 저장된 `TransactionInfo`를 이전 상태로 복원하는 역할만 하며 커밋/롤백을 다시 시도하지 않는다. 결국 커밋 실패 예외가 원래 비즈니스 예외보다 상위로 전파된다. Spring은 `TransactionSystemException`으로 래핑하여 "could not commit JPA transaction" 메시지와 함께 던진다. 원래 비즈니스 예외는 `initCause`로 연결된다.

---

<div align="center">

**[⬅️ 이전: PlatformTransactionManager 구조](./01-platform-transaction-manager.md)** | **[다음: Propagation 7가지 완전 분석 ➡️](./03-propagation-seven-types.md)**

</div>
