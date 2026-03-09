# Repository 테스트 전략 — Query Method / JPQL / Custom Repository 격리 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Query Method / JPQL 통합 테스트 / Custom Repository 격리 테스트를 어떻게 구분하는가?
- `TestEntityManager`로 픽스처 데이터를 준비하는 올바른 패턴은?
- Custom Repository(`JPAQueryFactory`)를 `@DataJpaTest`에서 설정하는 방법은?
- 페이지네이션, 정렬, 동적 조건 쿼리를 검증하는 전략은?
- Repository 테스트가 과도해지는 징후와 리팩터링 기준은?

---

## 🔍 왜 이게 존재하는가

### 문제: Repository 계층이 다양한 방식으로 구현되어 테스트 전략이 달라진다

```java
// Repository 구현 방식별 테스트 접근 차이:

// [방식 1] Spring Data Query Method
// findByEmail, findAllByStatusOrderByCreatedAtDesc
// → 명명 규칙으로 자동 생성 → "SQL이 올바른가?" 검증
// → @DataJpaTest + 단순 픽스처로 충분

// [방식 2] @Query JPQL/네이티브
// @Query("SELECT u FROM User u WHERE ...") 또는 nativeQuery=true
// → 쿼리 직접 작성 → "쿼리 문법, 조인, 조건이 올바른가?" 검증
// → @DataJpaTest + flush/clear + 쿼리 결과 검증

// [방식 3] Custom Repository (QueryDSL)
// JPAQueryFactory + BooleanExpression 동적 조건
// → 조건 조합마다 다른 SQL → 조합별 검증 필요
// → @DataJpaTest + @Import(QuerydslConfig) + 동적 조건 케이스별 테스트

// 전략을 통일하면: 단순한 것은 과잉 검증, 복잡한 것은 부족한 검증
```

---

## 😱 흔한 오해 또는 잘못된 사용

### Before: 모든 Repository 메서드에 동일한 테스트 패턴을 적용한다

```java
// ❌ Query Method를 과잉 검증
@Test
void findById_builtin() {
    User saved = repo.save(new User("홍길동", "hong@test.com"));
    Optional<User> found = repo.findById(saved.getId());
    assertTrue(found.isPresent()); // Spring Data가 이미 검증함
    // → Spring Data JPA의 기본 동작을 재검증하는 낭비
}

// ❌ 복잡한 동적 쿼리를 하나의 케이스만 테스트
@Test
void search_onlyHappyPath() {
    // name="홍", status=ACTIVE, minAge=20 케이스만 테스트
    // null 조건 처리 버그는 발견 못함
}

// ✅ 올바른 구분
// Query Method: 네이밍이 조건을 완전히 표현하는지만 검증
// @Query JPQL: 조인, 서브쿼리, 조건의 정확성 검증
// Custom (동적): 조건 조합별 — null 처리, 경계값 집중
```

---

## 🔬 내부 동작 원리 — 방식별 테스트 패턴

### 1. Query Method 테스트 — 최소 검증

```java
@DataJpaTest
class UserQueryMethodTest {

    @Autowired UserRepository repo;
    @Autowired TestEntityManager em;

    @BeforeEach
    void setUp() {
        em.persist(new User("홍길동", "hong@test.com", ACTIVE, 30));
        em.persist(new User("이순신", "lee@test.com", INACTIVE, 40));
        em.persist(new User("박지원", "park@test.com", ACTIVE, 20));
        em.flush();
        em.clear(); // 1차 캐시 비움 → 이후 조회는 반드시 DB 왕복
    }

    // [검증 포인트] 명명 규칙이 의도한 조건을 표현하는가?
    @Test
    void findByEmail_returnsMatchingUser() {
        Optional<User> found = repo.findByEmail("hong@test.com");
        assertTrue(found.isPresent());
        assertEquals("홍길동", found.get().getName());
    }

    @Test
    void findAllByStatus_returnsOnlyMatching() {
        List<User> active = repo.findAllByStatus(ACTIVE);
        assertEquals(2, active.size());
        assertTrue(active.stream().allMatch(u -> u.getStatus() == ACTIVE));
    }

    @Test
    void findAllByStatusOrderByAge_sortedCorrectly() {
        List<User> sorted = repo.findAllByStatusOrderByAgeAsc(ACTIVE);
        assertEquals(2, sorted.size());
        assertEquals(20, sorted.get(0).getAge()); // 박지원(20) 먼저
        assertEquals(30, sorted.get(1).getAge()); // 홍길동(30) 다음
    }

    // findById, save, delete: 검증 불필요 (Spring Data 기본 동작)
}
```

### 2. @Query JPQL 테스트 — 쿼리 정확성 집중

```java
@DataJpaTest
class UserJpqlTest {

    @Autowired UserRepository repo;
    @Autowired TestEntityManager em;

    @BeforeEach
    void setUp() {
        Team devTeam = em.persist(new Team("개발팀"));
        Team opsTeam = em.persist(new Team("운영팀"));

        em.persist(new User("홍길동", "hong@test.com", ACTIVE, devTeam));
        em.persist(new User("이순신", "lee@test.com", ACTIVE, opsTeam));
        em.persist(new User("김무열", "kim@test.com", INACTIVE, devTeam));
        em.flush();
        em.clear();
    }

    // [검증 포인트 1] JOIN FETCH — N+1 없이 연관관계 함께 로딩
    @Test
    void findAllWithTeam_noNPlusOne() {
        // SQL 로그 확인: SELECT u.*, t.* FROM users u JOIN teams t ... (1번만)
        List<User> users = repo.findAllWithTeam();
        assertEquals(3, users.size());

        // 팀 접근 시 추가 SQL 없어야 함 (이미 fetch join)
        users.forEach(u -> assertNotNull(u.getTeam().getName()));
    }

    // [검증 포인트 2] 조건 필터링 정확성
    @Test
    void findActiveUsersInTeam_correctFilter() {
        List<User> result = repo.findActiveUsersByTeamName("개발팀");
        assertEquals(1, result.size()); // 홍길동만 (김무열은 INACTIVE)
        assertEquals("홍길동", result.get(0).getName());
    }

    // [검증 포인트 3] Projection — DTO로 필요한 필드만
    @Test
    void findUserSummaries_projectionCorrect() {
        List<UserSummaryDto> summaries = repo.findAllSummaries();
        assertEquals(3, summaries.size());
        // DTO에 teamName이 올바르게 매핑되는지
        assertTrue(summaries.stream()
            .anyMatch(s -> "개발팀".equals(s.getTeamName())));
    }

    // [검증 포인트 4] 결과 없는 경우
    @Test
    void findActiveUsersByTeamName_noMatch_returnsEmpty() {
        List<User> result = repo.findActiveUsersByTeamName("없는팀");
        assertNotNull(result); // null이 아닌 빈 리스트
        assertTrue(result.isEmpty());
    }

    // [검증 포인트 5] 페이지네이션
    @Test
    void findActiveUsers_paged() {
        Page<User> page = repo.findAllByStatus(ACTIVE, PageRequest.of(0, 2));
        assertEquals(2, page.getContent().size());
        assertEquals(2, page.getTotalElements()); // 전체 ACTIVE 수
        assertTrue(page.isFirst());
    }
}
```

### 3. Custom Repository (QueryDSL) 테스트 — 동적 조건 조합

```java
// QuerydslConfig — JPAQueryFactory 빈 등록
@Configuration
public class QuerydslConfig {
    @Bean
    public JPAQueryFactory jpaQueryFactory(EntityManager em) {
        return new JPAQueryFactory(em);
    }
}

// UserQueryRepository — 테스트 대상
@Repository
public class UserQueryRepository {
    private final JPAQueryFactory queryFactory;
    private final QUser user = QUser.user;

    public List<UserDto> search(UserSearchCond cond) {
        return queryFactory
            .select(Projections.constructor(UserDto.class,
                user.id, user.name, user.email, user.status))
            .from(user)
            .where(
                nameContains(cond.getName()),
                statusEq(cond.getStatus()),
                minAgeGoe(cond.getMinAge())
            )
            .orderBy(user.name.asc())
            .fetch();
    }

    private BooleanExpression nameContains(String name) {
        return StringUtils.hasText(name) ? user.name.contains(name) : null;
    }
    private BooleanExpression statusEq(Status status) {
        return status != null ? user.status.eq(status) : null;
    }
    private BooleanExpression minAgeGoe(Integer minAge) {
        return minAge != null ? user.age.goe(minAge) : null;
    }
}

// 테스트
@DataJpaTest
@Import(QuerydslConfig.class)
class UserQueryRepositoryTest {

    @Autowired JPAQueryFactory queryFactory;
    @Autowired TestEntityManager em;
    UserQueryRepository queryRepo;

    @BeforeEach
    void setUp() {
        queryRepo = new UserQueryRepository(queryFactory);
        em.persist(new User("홍길동", "hong@test.com", ACTIVE, 30));
        em.persist(new User("홍길순", "soon@test.com", ACTIVE, 25));
        em.persist(new User("이순신", "lee@test.com", INACTIVE, 40));
        em.persist(new User("박지원", "park@test.com", ACTIVE, 20));
        em.flush(); em.clear();
    }

    // [케이스 1] 모든 조건 null → 전체 조회
    @Test
    void search_noCondition_returnsAll() {
        List<UserDto> result = queryRepo.search(new UserSearchCond());
        assertEquals(4, result.size());
    }

    // [케이스 2] 각 조건 단독 — null 처리 검증
    @Test
    void search_nameOnly_filtersCorrectly() {
        UserSearchCond cond = UserSearchCond.builder().name("홍").build();
        List<UserDto> result = queryRepo.search(cond);
        assertEquals(2, result.size());
        assertTrue(result.stream().allMatch(u -> u.getName().contains("홍")));
    }

    @Test
    void search_statusOnly_filtersCorrectly() {
        UserSearchCond cond = UserSearchCond.builder().status(ACTIVE).build();
        assertEquals(3, queryRepo.search(cond).size());
    }

    @Test
    void search_minAgeOnly_filtersCorrectly() {
        UserSearchCond cond = UserSearchCond.builder().minAge(30).build();
        List<UserDto> result = queryRepo.search(cond);
        assertEquals(2, result.size()); // 홍길동(30), 이순신(40)
    }

    // [케이스 3] 조건 조합
    @Test
    void search_nameAndStatus_combined() {
        UserSearchCond cond = UserSearchCond.builder()
            .name("홍").status(ACTIVE).build();
        assertEquals(2, queryRepo.search(cond).size());
    }

    // [케이스 4] 경계값
    @Test
    void search_noMatch_returnsEmptyList() {
        UserSearchCond cond = UserSearchCond.builder().name("없는이름").build();
        List<UserDto> result = queryRepo.search(cond);
        assertNotNull(result);
        assertTrue(result.isEmpty()); // null 아닌 빈 리스트
    }

    // [케이스 5] 정렬 검증
    @Test
    void search_sortedByNameAsc() {
        List<UserDto> result = queryRepo.search(new UserSearchCond());
        List<String> names = result.stream().map(UserDto::getName).toList();
        assertEquals(List.of("박지원", "이순신", "홍길동", "홍길순"), names);
    }
}
```

### 4. TestEntityManager 픽스처 패턴

```java
@DataJpaTest
class FixturePatternTest {

    @Autowired TestEntityManager em;
    @Autowired OrderRepository orderRepo;

    // [패턴 1] persist + flush + clear — DB 왕복 보장
    @Test
    void dbRoundTripVerification() {
        User user = em.persist(new User("홍길동", "hong@test.com"));
        Order order = em.persist(new Order(user, PENDING));
        em.flush();  // SQL 전송
        em.clear();  // 1차 캐시 초기화

        // 이후 조회는 반드시 DB에서 가져옴
        Order found = orderRepo.findById(order.getId()).orElseThrow();
        assertEquals(PENDING, found.getStatus());
        assertEquals("홍길동", found.getUser().getName()); // EAGER or Fetch Join
    }

    // [패턴 2] persistFlushFind — 저장 후 즉시 재조회
    @Test
    void persistFlushFind_getsDbVersion() {
        User user = em.persistFlushFind(new User("홍길동", "hong@test.com"));
        // persist → flush → clear → find 한 번에
        // DB에서 다시 가져온 인스턴스 → @CreatedDate 등 DB 기본값 포함
        assertNotNull(user.getCreatedAt()); // DB DEFAULT 반영 확인
    }

    // [패턴 3] 연관관계 포함 픽스처
    @Test
    void relationshipFixture() {
        Team team = em.persist(Team.builder().name("개발팀").build());
        User user1 = em.persist(User.builder()
            .name("홍길동").email("hong@test.com").team(team).build());
        User user2 = em.persist(User.builder()
            .name("이순신").email("lee@test.com").team(team).build());
        em.flush(); em.clear();

        // 팀 조회 후 멤버 접근 (LAZY)
        Team found = em.find(Team.class, team.getId());
        // @Transactional 내 → Lazy Loading 가능
        assertEquals(2, found.getMembers().size());
    }
}
```

### 5. @Modifying 쿼리 테스트

```java
@DataJpaTest
class ModifyingQueryTest {

    @Autowired UserRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void bulkStatusUpdate_affectsCorrectRows() {
        em.persist(new User("홍길동", "hong@test.com", ACTIVE));
        em.persist(new User("이순신", "lee@test.com", ACTIVE));
        em.persist(new User("박지원", "park@test.com", INACTIVE));
        em.flush(); em.clear();

        // 벌크 업데이트 실행
        int updated = repo.updateStatusByEmail("hong@test.com", INACTIVE);
        assertEquals(1, updated); // 영향받은 행 수 검증

        em.clear(); // 1차 캐시 클리어 필수!
        // (벌크 업데이트는 영속성 컨텍스트 갱신 안 함)

        User user = repo.findByEmail("hong@test.com").orElseThrow();
        assertEquals(INACTIVE, user.getStatus()); // DB 반영 확인
    }

    @Test
    void softDelete_updatesDeletedAt() {
        User user = em.persistFlushFind(new User("홍길동", "hong@test.com"));
        em.clear();

        repo.softDelete(user.getId());
        em.clear();

        // 소프트 삭제 확인
        Optional<User> found = repo.findById(user.getId());
        assertFalse(found.isPresent()); // @Where(clause="deleted_at IS NULL")
        // 실제 DB에서 raw 조회
        User deleted = em.find(User.class, user.getId());
        assertNotNull(deleted.getDeletedAt()); // 삭제 시각 설정됨
    }
}
```

---

## 💻 실험으로 확인하기

### 실험 1: em.clear() 없이 @Modifying 결과 확인

```java
@Test
void withoutClear_staleDataFromCache() {
    User user = em.persist(new User("홍길동", ACTIVE));
    em.flush();
    // clear() 없이 벌크 업데이트

    repo.updateStatusByEmail(user.getEmail(), INACTIVE);
    // 1차 캐시에 ACTIVE 상태 남아있음

    User cached = repo.findByEmail(user.getEmail()).orElseThrow();
    // 1차 캐시에서 반환 → 여전히 ACTIVE!
    assertEquals(ACTIVE, cached.getStatus()); // DB는 INACTIVE지만 캐시는 ACTIVE

    em.clear(); // clear 후
    User fresh = repo.findByEmail(user.getEmail()).orElseThrow();
    assertEquals(INACTIVE, fresh.getStatus()); // DB 값 정확히 반영
}
```

---

## ⚡ 성능 임팩트

```
테스트 방식별 실행 시간:

Query Method 테스트 (5개 메서드):
  픽스처: ~5ms (3~5개 INSERT)
  쿼리: ~1ms 각
  롤백: ~2ms
  총: ~15ms

@Query JPQL 테스트 (5개 메서드):
  픽스처 + flush + clear: ~10ms
  쿼리: ~2ms 각
  총: ~25ms

Custom Repository 동적 조건 (8개 케이스):
  픽스처: ~10ms (1회)
  각 케이스: ~2ms
  총: ~30ms

→ 모두 빠름 — H2 인메모리 + @Transactional 롤백 덕분
→ @SpringBootTest 대비 10배 이상 빠름
```

---

## 🤔 트레이드오프

```
테스트 과도화 징후:
  findById, save, delete 검증 (Spring Data 기본)
  동일한 픽스처 + 조건 반복 테스트
  의미 없는 assertTrue(true) 수준의 단언

테스트 부족 징후:
  동적 쿼리의 null 조건 미검증
  결과 없는 경우 미검증 (null vs 빈 리스트)
  @Modifying 후 em.clear() 없이 검증
  페이지네이션 totalElements 미검증

Query Method:
  → 명명 규칙이 의도를 표현하는지만 검증
  → findById, save, delete: 검증 불필요

@Query JPQL:
  → 조인, 조건, Projection 정확성
  → 결과 없는 케이스, 페이지네이션

Custom (QueryDSL):
  → 조건 조합별 (null, 단독, 복합, 경계값)
  → 정렬 순서, 빈 리스트 반환
```

---

## 📌 핵심 정리

```
방식별 테스트 전략
  Query Method: 최소 검증 (명명 조건 표현 여부)
  @Query JPQL: 조인/조건/Projection 정확성, 경계값
  Custom (동적): 조건 조합별 — null, 단독, 복합, 경계값

TestEntityManager 패턴
  persist + flush + clear: DB 왕복 보장
  persistFlushFind: 저장 후 즉시 재조회 (DB 기본값 포함)
  연관관계: 부모 먼저 persist → 자식 persist → flush + clear

Custom Repository @DataJpaTest 설정
  @Import(QuerydslConfig.class)로 JPAQueryFactory 등록
  Custom Repository는 직접 new로 생성
  @Autowired JPAQueryFactory 주입 후 주입

@Modifying 주의점
  벌크 업데이트 후 em.clear() 필수
  1차 캐시 갱신 안 됨 → 클리어 없이 조회 시 stale 데이터
```

---

## 🤔 생각해볼 문제

**Q1.** `@DataJpaTest`에서 `@Transactional`이 기본 적용되는데, `repo.findAll()`이 지연 로딩 연관관계를 접근해도 `LazyInitializationException`이 발생하지 않는 이유는?

**Q2.** Custom Repository를 `@DataJpaTest`에서 `@Autowired`로 직접 주입하려면 어떻게 해야 하는가?

**Q3.** `em.persistFlushFind()`와 `repo.save()` 후 `repo.findById()` 조회의 차이는?

> 💡 **해설**
>
> **Q1.** `@DataJpaTest`에는 `@Transactional`이 기본으로 붙어있어 테스트 메서드 전체가 하나의 트랜잭션 내에서 실행된다. Lazy 연관관계를 접근하는 시점에도 트랜잭션이 활성화되어 있으므로 Hibernate가 추가 SQL을 실행해 Lazy 로딩을 완료할 수 있다. 운영 환경에서는 서비스 트랜잭션이 종료된 후 Controller에서 Detached 엔티티에 접근하면 `LazyInitializationException`이 발생하지만, 테스트에서는 트랜잭션이 테스트 메서드 전체를 감싸므로 이 예외가 숨겨진다.
>
> **Q2.** `@Repository`가 붙은 인터페이스의 구현체가 아닌, `JPAQueryFactory`를 직접 사용하는 커스텀 클래스는 `@DataJpaTest`에서 자동으로 로딩되지 않는다. `@Autowired`로 주입하려면 `@Import(UserQueryRepository.class)`를 테스트 클래스에 추가해야 한다. 단, `UserQueryRepository`가 `JPAQueryFactory`를 생성자로 받는다면 `@Import`만으로는 부족하고 `QuerydslConfig`도 함께 `@Import`해야 한다. 또는 `@BeforeEach`에서 `new UserQueryRepository(queryFactory)`로 직접 생성하는 방식도 많이 사용된다.
>
> **Q3.** `em.persistFlushFind()`는 세 단계를 순서대로 실행한다: `persist()` → `flush()`(SQL DB 전송) → `clear()`(1차 캐시 초기화) → `find()`(DB에서 다시 SELECT). 반환된 인스턴스는 DB에서 새로 가져온 것이므로 `@CreatedDate`, DB 기본값, 트리거 결과 등이 반영되어 있다. 반면 `repo.save()` 후 `repo.findById()`는 flush 없이 1차 캐시에 저장된 동일 인스턴스를 반환할 수 있다. `findById()`가 같은 트랜잭션 내에서 같은 ID를 조회하면 DB 왕복 없이 1차 캐시의 인스턴스를 즉시 반환하기 때문이다. 따라서 DB에 실제로 어떤 값이 저장됐는지 검증하려면 `persistFlushFind()` 또는 명시적 `flush()` + `clear()` 후 조회를 사용해야 한다.

---

<div align="center">

**[⬅️ 이전: Testcontainers vs H2 선택 기준](./03-testcontainers-vs-h2.md)** | **[다음: @Sql 스크립트 관리 ➡️](./05-sql-script-management.md)**

</div>
