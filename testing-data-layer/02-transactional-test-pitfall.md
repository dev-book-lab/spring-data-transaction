# @Transactional in Test의 함정 — 자동 롤백이 실제 동작을 은폐하는 방식

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 테스트 `@Transactional`의 자동 롤백이 어떤 메커니즘으로 동작하는가?
- `REQUIRES_NEW` 전파 동작이 테스트에서 다르게 보이는 이유는?
- `@Transactional` 테스트가 LazyInitializationException을 숨기는 원리는?
- `@TransactionalEventListener(AFTER_COMMIT)`이 테스트에서 발행되지 않는 이유는?
- `@Rollback(false)`를 사용해야 하는 상황과 주의점은?

---

## 🔍 왜 이게 존재하는가

### 문제: 테스트 격리와 실제 동작 검증이 상충한다

```java
// 테스트가 풀어야 하는 두 상충 요구:

// [격리] 테스트 간 데이터 오염 방지
// → 각 테스트 종료 시 DB 초기화
// → @Transactional 자동 롤백으로 해결 가능

// [검증] 실제 트랜잭션 동작 확인
// → "이 메서드가 예외 시 롤백되는가?"
// → "REQUIRES_NEW가 별도로 커밋되는가?"
// → @Transactional 테스트 안에서는 경계가 흐려짐

// 이 두 요구를 모두 충족하는 상황별 전략이 필요하다
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: @Transactional 테스트는 항상 안전하다

```java
// ❌ 세 가지 함정이 있다

// [함정 1] LazyInitializationException 은폐
@Service
public class UserService {
    @Transactional
    public User findUser(Long id) {
        User user = userRepository.findById(id).orElseThrow();
        return user; // Detached 상태로 반환
    }
}

@SpringBootTest
@Transactional // 테스트 트랜잭션이 서비스 트랜잭션을 감쌈
class UserServiceTest {
    @Test
    void findUser_lazyAccessAfterReturn() {
        User user = userService.findUser(1L);
        user.getOrders().size(); // ← 테스트 트랜잭션이 살아있어서 OK!
        // 실제 운영: 서비스 트랜잭션 종료 후 Detached → LazyInitializationException
        // 테스트에서 통과 → 운영 버그 발견 못함
    }
}

// [함정 2] REQUIRES_NEW 동작 왜곡
@Transactional // 테스트 트랜잭션
class AuditServiceTest {
    @Test
    void requiresNew_shouldCommitIndependently() {
        auditService.logAudit("action"); // 내부에서 REQUIRES_NEW 사용
        // REQUIRES_NEW는 새 트랜잭션 → 즉시 커밋
        // 테스트 종료 → 바깥 트랜잭션 롤백
        // auditLog는 이미 커밋됐으므로 남아있음
        // → REQUIRES_NEW가 "독립 커밋"하는지 제대로 검증하기 어려움
    }
}

// [함정 3] AFTER_COMMIT 이벤트 미발행
@Transactional
class EventTest {
    @Test
    void afterCommitEventShouldFire() {
        userService.createUser("홍길동");
        // 테스트 트랜잭션이 롤백 → AFTER_COMMIT 이벤트 발행 안 됨
        verify(emailService).sendWelcome("홍길동"); // ← 항상 실패
    }
}
```

---

## 🔬 내부 동작 원리 — 테스트 트랜잭션 관리 소스 추적

### 1. @Transactional 테스트 롤백 메커니즘

```java
// SpringExtension (JUnit 5) — 테스트 생명주기 관리
public class SpringExtension implements ... {

    // 테스트 메서드 실행 전
    @Override
    public void beforeTestMethod(ExtensionContext context) {
        // @Transactional 확인
        TransactionContext txContext = TransactionContextHolder.currentTransactionContext();

        if (isTransactional(testMethod)) {
            // 트랜잭션 시작
            txContext.startTransaction();
            // → PlatformTransactionManager.getTransaction()
            // → TransactionSynchronizationManager에 Connection 바인딩
        }
    }

    // 테스트 메서드 실행 후
    @Override
    public void afterTestMethod(ExtensionContext context) {
        // @Rollback 확인 (기본 true)
        boolean rollback = isRollback(testMethod);

        if (rollback) {
            txContext.endTransaction(); // → rollback()
        } else {
            txContext.endTransaction(); // → commit()
        }
        // ThreadLocal 정리
    }
}

// 결과:
// 테스트 메서드 전체 → 하나의 트랜잭션
// 내부에서 호출하는 @Transactional 서비스 → REQUIRED → 같은 트랜잭션 합류
// 테스트 종료 → rollback → 모든 변경 취소
```

### 2. REQUIRES_NEW가 테스트 트랜잭션과 상호작용하는 방식

```java
// 정상 동작 (운영):
// 외부 트랜잭션(A) → REQUIRES_NEW 메서드 → 별도 트랜잭션(B) 커밋 → A 계속

// 테스트 환경:
// 테스트 트랜잭션(T) → REQUIRES_NEW 메서드 → 별도 트랜잭션(B) 커밋 → T 계속
//                                               ↑ 이 커밋은 실제로 일어남!
// 테스트 종료 → T 롤백 → T에서 한 작업만 롤백
// B에서 커밋된 데이터는 남음

@SpringBootTest
@Transactional
class RequiresNewTest {

    @Autowired AuditService auditService; // REQUIRES_NEW 내부 사용
    @Autowired AuditLogRepository auditLogRepo;

    @Test
    void requiresNewCommitsImmediately() {
        auditService.log("action"); // REQUIRES_NEW → 즉시 커밋

        // 이 시점에 auditLog는 DB에 커밋됨
        assertEquals(1, auditLogRepo.count()); // ✅ 통과

        // 테스트 종료 → T 롤백
        // but auditLog는 이미 커밋 → 남아있음 → 다음 테스트에 영향!
    }

    // ✅ 올바른 접근: @Transactional 제거 + @AfterEach 정리
}
```

### 3. LazyInitializationException을 숨기는 원리

```java
// 정상 동작 (운영):
// [서비스 트랜잭션] findUser() → User 반환 → 트랜잭션 종료 → User Detached
// [Controller] user.getOrders() → LazyInitializationException!

// 테스트 @Transactional 환경:
// [테스트 트랜잭션] 시작
//   [서비스 트랜잭션 (REQUIRED)] findUser() → User 반환 → 서비스 메서드 종료
//   (REQUIRED → 테스트 트랜잭션에 합류 → 실제로 종료 안 됨)
//   user.getOrders() → 테스트 트랜잭션이 살아있음 → Lazy Loading OK!
// [테스트 트랜잭션] 종료

// @Transactional 테스트가 LazyInit 버그를 숨기는 이유:
// 서비스 트랜잭션이 "실제로 종료"되지 않고 테스트 트랜잭션에 흡수되기 때문

// ✅ 올바른 테스트: @Transactional 제거
@SpringBootTest // @Transactional 없음
class UserServiceRealTest {

    @Autowired UserService userService;

    @Test
    void findUser_detachedAfterReturn() {
        // 서비스 트랜잭션 실제 종료 → User Detached
        User user = userService.findUser(savedId);

        // LazyInitializationException 발생!
        assertThrows(LazyInitializationException.class,
            () -> user.getOrders().size()); // ← 운영 버그 조기 발견 ✅
    }
}
```

### 4. @TransactionalEventListener와 테스트 트랜잭션

```java
// AFTER_COMMIT: 트랜잭션이 커밋된 후 발행
// 테스트 @Transactional: 롤백으로 종료 → AFTER_COMMIT 미발행

@Component
public class UserEventListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onUserCreated(UserCreatedEvent event) {
        emailService.sendWelcome(event.getName());
    }
}

// ❌ @Transactional 테스트 — 이벤트 미발행
@SpringBootTest
@Transactional
class EventListenerWrongTest {
    @Test
    void welcomeEmailSentAfterUserCreated() {
        userService.createUser("홍길동"); // 이벤트 발행 → AFTER_COMMIT 대기
        // 테스트 종료 → 롤백 (커밋 없음) → AFTER_COMMIT 리스너 실행 안 됨
        verify(emailService).sendWelcome("홍길동"); // ← 항상 실패
    }
}

// ✅ @Transactional 제거 → 실제 커밋 → AFTER_COMMIT 발행
@SpringBootTest // @Transactional 없음
class EventListenerCorrectTest {

    @MockBean EmailService emailService;
    @Autowired UserService userService;
    @Autowired UserRepository userRepository;

    @AfterEach
    void cleanup() { userRepository.deleteAll(); }

    @Test
    void welcomeEmailSentAfterCommit() {
        userService.createUser("홍길동"); // 서비스 트랜잭션 → 실제 커밋
        // AFTER_COMMIT → emailService.sendWelcome("홍길동") 실행

        verify(emailService, timeout(1000)).sendWelcome("홍길동"); // ✅
    }
}
```

### 5. @Rollback(false) — 사용 기준

```java
// @Rollback(false): 테스트 후 커밋 (롤백하지 않음)

// 적합한 사용 사례:
// 1. AFTER_COMMIT 이벤트 발행 테스트
// 2. DB 트리거/뷰 동작 확인 (커밋 후에만 발동하는 경우)
// 3. 실제 데이터 상태를 시각적으로 확인하고 싶을 때 (임시 디버깅)

@DataJpaTest
class DataInspectionTest {

    @Autowired UserRepository repo;

    @Test
    @Rollback(false) // 커밋 → DB에 실제 저장됨
    @Disabled("디버깅용: 실행 후 DB 확인, 커밋 후 반드시 제거")
    void inspectData() {
        repo.save(new User("디버그유저", "debug@test.com"));
        // 테스트 종료 후 DB에 데이터 남아있음 → DBeaver 등으로 확인 가능
    }
}

// 주의: @Rollback(false) 사용 후 반드시 데이터 정리
// 다음 테스트에 영향 줄 수 있음
// → 임시 디버깅용으로만 사용, @Disabled 함께 붙이는 것 권장
```

### 6. 실제 트랜잭션 동작 검증 — 올바른 패턴

```java
// 트랜잭션 전파, 롤백 동작을 검증하는 올바른 방법
// @Transactional 제거 + @AfterEach 정리

@SpringBootTest
// @Transactional 없음
class TransactionBehaviorTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepo;
    @Autowired AuditLogRepository auditRepo;

    @AfterEach
    void cleanup() {
        auditRepo.deleteAll();
        orderRepo.deleteAll();
    }

    // [검증 1] 예외 시 롤백
    @Test
    void rollbackOnException() {
        assertThrows(PaymentException.class,
            () -> orderService.placeOrder(failingRequest));

        assertEquals(0, orderRepo.count()); // 롤백 확인
    }

    // [검증 2] REQUIRES_NEW 독립 커밋
    @Test
    void requiresNewSurvivesOuterRollback() {
        assertThrows(OrderException.class,
            () -> orderService.placeOrderWithAudit(failingRequest));

        assertEquals(0, orderRepo.count());    // 주문 롤백
        assertEquals(1, auditRepo.count());   // 감사 로그 독립 커밋 ✅
    }

    // [검증 3] readOnly에서 쓰기 시도
    @Test
    void readOnlyPreventsWrite() {
        assertThrows(Exception.class,
            () -> readOnlyUserService.tryToModify(userId));
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: 자동 롤백으로 격리 확인

```java
@DataJpaTest
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class RollbackIsolationTest {

    @Autowired UserRepository repo;

    @Test @Order(1)
    void insert() {
        repo.save(new User("홍길동", "hong@test.com"));
        assertEquals(1, repo.count());
    } // → 롤백

    @Test @Order(2)
    void previousDataGone() {
        assertEquals(0, repo.count()); // 이전 테스트 롤백 확인 ✅
    }
}
```

### 실험 2: BEFORE_COMMIT vs AFTER_COMMIT 발행 비교

```java
@SpringBootTest
@Transactional
class EventPhaseTest {

    @MockBean EventRecorder recorder; // 이벤트 수신 Mock

    @Test
    void beforeCommitFires_afterCommitDoesNot() {
        userService.createUserWithEvents("홍길동");
        // BEFORE_COMMIT: 커밋 직전 → 같은 트랜잭션 → 롤백 전 실행됨
        verify(recorder).recordBeforeCommit("홍길동"); // ✅ 발행됨

        // AFTER_COMMIT: 커밋 후 → 테스트는 롤백 → 발행 안 됨
        verify(recorder, never()).recordAfterCommit("홍길동"); // ✅ 미발행
    }
}
```

---

## ⚡ 성능 임팩트

```
롤백 성능:
  DB 언두 로그 적용 → 커밋보다 빠름
  대량 데이터 삽입 후 롤백: 삭제 DML보다 훨씬 빠름
  → 격리를 위한 rollback은 @AfterEach deleteAll()보다 성능 우수

@Transactional 없는 테스트:
  @AfterEach deleteAll(): 행 수에 비례
  TRUNCATE: 행 수 무관, 빠름 (DDL)
  @Transactional 롤백: 가장 빠름

결론:
  단순 격리 목적: @Transactional 롤백 (가장 빠름)
  트랜잭션 동작 검증: @Transactional 제거 + TRUNCATE/deleteAll
```

---

## 🤔 트레이드오프

```
@Transactional 테스트 (자동 롤백):
  ✅ 격리 자동, 빠른 롤백, 설정 단순
  ❌ 트랜잭션 전파 검증 불가
  ❌ Lazy Loading 버그 은폐
  ❌ AFTER_COMMIT 이벤트 미발행
  → Repository CRUD, JPQL 정확성

@Transactional 없는 테스트:
  ✅ 실제 트랜잭션 동작 검증
  ✅ LazyInit 버그 발견
  ✅ AFTER_COMMIT 이벤트 발행
  ❌ @AfterEach 정리 코드 필요
  → 서비스 통합, 트랜잭션 전파, 이벤트 리스너

@Rollback(false):
  ✅ AFTER_COMMIT 발행, DB 직접 확인
  ❌ 다음 테스트 데이터 오염
  → 임시 디버깅만, 항상 @Disabled 병행
```

---

## 📌 핵심 정리

```
@Transactional 테스트 세 가지 함정
  1. LazyInitializationException 은폐
     → 서비스 트랜잭션이 테스트 트랜잭션에 흡수 → Lazy OK처럼 보임
  2. REQUIRES_NEW 동작 왜곡
     → 별도 커밋은 실제 일어남, 하지만 검증 구조 복잡
  3. AFTER_COMMIT 이벤트 미발행
     → 롤백으로 종료 → AFTER_COMMIT 트리거 없음

트랜잭션 동작 검증 패턴
  @Transactional 제거
  @AfterEach: 테이블 정리 (deleteAll 또는 TRUNCATE)
  실제 커밋/롤백 결과를 Repository로 검증

@Rollback(false)
  AFTER_COMMIT 테스트, 임시 디버깅용
  항상 @AfterEach 정리 또는 @Disabled 병행
```

---

## 🤔 생각해볼 문제

**Q1.** 테스트 클래스에 `@Transactional`이 있고, 내부 서비스 메서드에 `@Transactional(readOnly=true)`가 있으면 어떤 트랜잭션 특성이 적용되는가?

**Q2.** `REQUIRES_NEW` 메서드를 테스트할 때 `@Transactional` 테스트와 비트랜잭션 테스트에서 `auditLogRepository.count()` 결과가 다른 이유는?

**Q3.** `@DataJpaTest`에서 `@Transactional`이 기본으로 적용되는데, `@Transactional(propagation=NEVER)`를 가진 Repository 메서드를 호출하면 어떻게 되는가?

> 💡 **해설**
>
> **Q1.** `REQUIRED` 전파(기본)이므로 테스트 트랜잭션에 합류한다. 합류 시 기존 트랜잭션의 특성이 유지되므로, 이미 읽기-쓰기 트랜잭션으로 시작된 테스트 트랜잭션에 `readOnly=true` 메서드가 합류하면 `readOnly` 속성은 무시된다. Hibernate는 첫 트랜잭션 시작 시 `FlushMode`를 결정하는데, 이미 읽기-쓰기 모드로 시작됐으므로 `readOnly=true`의 `FlushMode.MANUAL` 최적화가 적용되지 않는다. 즉 `readOnly=true` 효과(FlushMode 최적화, 읽기 전용 힌트)가 없어진다.
>
> **Q2.** `@Transactional` 테스트에서: `REQUIRES_NEW` 메서드가 별도 트랜잭션을 열고 커밋한 후, 테스트 트랜잭션이 계속된다. `count()`는 테스트 트랜잭션 내에서 실행되는데, REQUIRES_NEW로 커밋된 데이터는 이미 DB에 저장되어 있으므로 `count() = 1`이다. 테스트 종료 → 테스트 트랜잭션 롤백 → auditLog는 이미 커밋됐으므로 남는다. 비트랜잭션 테스트에서: 서비스 메서드 호출 시 REQUIRES_NEW가 커밋, 외부 트랜잭션도 정상 커밋/롤백된다. `@AfterEach`로 정리하지 않으면 다음 테스트에 데이터가 남는다.
>
> **Q3.** `TransactionException`이 발생한다. `NEVER` 전파는 "현재 트랜잭션이 있으면 예외를 던진다"는 의미다. `@DataJpaTest`가 테스트 트랜잭션을 시작한 상태에서 `NEVER` 메서드를 호출하면 `IllegalTransactionStateException: Existing transaction found for transaction marked with propagation 'never'`가 발생한다. 이 동작 자체를 검증하는 테스트라면 의도된 결과지만, 단순 Repository 테스트를 목적으로 하는 경우라면 `@DataJpaTest`의 기본 `@Transactional`이 방해가 된다. 해결: 해당 테스트 메서드에 `@Transactional(propagation=NOT_SUPPORTED)`를 붙여 테스트 트랜잭션을 일시 중단시킨다.

---

<div align="center">

**[⬅️ 이전: @DataJpaTest 범위와 제약](./01-datajpatest-scope.md)** | **[다음: Testcontainers vs H2 선택 기준 ➡️](./03-testcontainers-vs-h2.md)**

</div>
