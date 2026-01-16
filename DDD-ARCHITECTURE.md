# DDD 아키텍처 설계 문서

## 프로젝트 개요

이 프로젝트는 **Domain-Driven Design (DDD)** 원칙과 **Hexagonal Architecture (Ports and Adapters)**를 결합한 Spring Boot 3.5 기반 애플리케이션입니다.

### 핵심 기술 스택
- **Java**: 21
- **Spring Boot**: 3.5.0
- **JPA/Hibernate**: ORM 매핑
- **Spring Data JPA**: Repository 구현
- **ArchUnit**: 아키텍처 규칙 검증

---

## 아키텍처 개요

### Hexagonal Architecture (3계층 구조)

```
┌─────────────────────────────────────────┐
│          Adapter Layer                  │
│  (webapi, security, integration)        │
│  - MemberApi (REST Controller)          │
│  - SecurePasswordEncoder                │
│  - DummyEmailSender                     │
└───────────────┬─────────────────────────┘
                │ depends on
┌───────────────▼─────────────────────────┐
│       Application Layer                 │
│  (provided interfaces + services)       │
│  - MemberRegister (provided port)       │
│  - MemberFinder (provided port)         │
│  - MemberRepository (required port)     │
│  - EmailSender (required port)          │
│  - MemberModifyService                  │
│  - MemberQueryService                   │
└───────────────┬─────────────────────────┘
                │ depends on
┌───────────────▼─────────────────────────┐
│          Domain Layer                   │
│  (pure business logic)                  │
│  - Member (Aggregate Root)              │
│  - MemberDetail (Entity)                │
│  - Profile, Email (Value Objects)       │
│  - PasswordEncoder (Domain Service)     │
└─────────────────────────────────────────┘
```

---

## 패키지 구조

### 1. Domain Layer (`tobyspring.splearn.domain`)

도메인 계층은 순수한 비즈니스 로직을 포함하며, 외부 프레임워크나 인프라에 대한 의존성이 없습니다.

```
domain/
├── member/                          # Member Bounded Context
│   ├── Member.java                  # Aggregate Root (집합 루트)
│   ├── MemberDetail.java            # Entity (엔티티)
│   ├── MemberStatus.java            # Enum (열거형)
│   ├── Profile.java                 # Value Object (값 객체)
│   ├── PasswordEncoder.java         # Domain Service Interface
│   ├── MemberRegisterRequest.java   # Command Object (커맨드)
│   ├── MemberInfoUpdateRequest.java # Command Object
│   ├── DuplicateEmailException.java # Domain Exception
│   └── DuplicateProfileException.java
├── shared/                          # Shared Kernel (공유 커널)
│   └── Email.java                   # Shared Value Object
└── AbstractEntity.java              # Base Entity Class
```

**특징:**
- 순수 Java 코드 (프레임워크 독립적)
- 비즈니스 규칙과 불변식 포함
- 최소한의 JPA 어노테이션만 사용 (`@Entity`, `@NaturalId`)

### 2. Application Layer (`tobyspring.splearn.application`)

애플리케이션 계층은 유스케이스를 정의하고 구현하며, 포트(인터페이스)를 정의합니다.

```
application/
└── member/
    ├── provided/                    # Driving Ports (Inbound)
    │   ├── MemberRegister.java      # Use Case Interface
    │   └── MemberFinder.java        # Query Interface
    ├── required/                    # Driven Ports (Outbound)
    │   ├── MemberRepository.java    # Persistence Port
    │   └── EmailSender.java         # Integration Port
    ├── MemberModifyService.java     # Use Case Implementation
    └── MemberQueryService.java      # Query Implementation
```

**포트 분류:**
- **Provided Ports (제공 포트)**: 외부에서 애플리케이션을 호출하는 인터페이스
- **Required Ports (요구 포트)**: 애플리케이션이 외부 인프라를 호출하는 인터페이스

### 3. Adapter Layer (`tobyspring.splearn.adapter`)

어댑터 계층은 외부 시스템과의 통합을 담당하며, 포트를 구현합니다.

```
adapter/
├── webapi/                          # Primary/Driving Adapters
│   ├── MemberApi.java               # REST Controller
│   ├── ApiControllerAdvice.java     # Exception Handler
│   └── dto/
│       └── MemberRegisterResponse.java
├── security/                        # Secondary/Driven Adapters
│   └── SecurePasswordEncoder.java   # Password Encoder Adapter
└── integration/                     # Secondary/Driven Adapters
    └── DummyEmailSender.java        # Email Service Adapter
```

**어댑터 분류:**
- **Primary Adapters (주도 어댑터)**: 애플리케이션을 호출 (예: REST API, CLI)
- **Secondary Adapters (피주도 어댑터)**: 애플리케이션이 호출 (예: DB, 외부 API)

---

## DDD 전술 패턴 (Tactical Patterns)

### 1. Aggregate (집합)

**Member Aggregate:**
```java
@Entity
public class Member extends AbstractEntity {  // Aggregate Root
    @NaturalId
    private Email email;              // Value Object
    private String nickname;
    private String passwordHash;
    private MemberStatus status;      // Enum
    private MemberDetail detail;      // Entity (집합 내부 엔티티)

    // Factory Method Pattern
    public static Member register(
        MemberRegisterRequest registerRequest,
        PasswordEncoder passwordEncoder
    ) {
        Member member = new Member();
        member.email = new Email(registerRequest.email());
        member.nickname = requireNonNull(registerRequest.nickname());
        member.passwordHash = requireNonNull(
            passwordEncoder.encode(registerRequest.password())
        );
        member.status = MemberStatus.PENDING;
        member.detail = MemberDetail.create();
        return member;
    }

    // Business Logic in Domain
    public void activate() {
        state(status == MemberStatus.PENDING, "PENDING 상태가 아닙니다");
        this.status = MemberStatus.ACTIVE;
        this.detail.activate();  // 자식 엔티티 전파
    }

    public void deactivate() {
        state(status == MemberStatus.ACTIVE, "ACTIVE 상태가 아닙니다");
        this.status = MemberStatus.INACTIVE;
        this.detail.deactivate();
    }

    public void updateInfo(MemberInfoUpdateRequest updateRequest) {
        state(getStatus() == MemberStatus.ACTIVE,
              "등록 완료 상태가 아니면 정보를 수정할 수 없습니다");
        this.nickname = Objects.requireNonNull(updateRequest.nickname());
        this.detail.updateInfo(updateRequest);
    }
}
```

**집합의 특징:**
- `Member`는 집합 루트 (Aggregate Root)
- `MemberDetail`은 집합 경계 내의 엔티티
- 모든 수정은 집합 루트를 통해서만 가능
- 트랜잭션 일관성 경계 (Transactional Consistency Boundary)
- 불변식 (Invariants) 보호

**집합 설계 원칙:**
1. 작은 집합 설계 (Small Aggregates)
2. ID로 다른 집합 참조 (Reference by ID)
3. 경계 밖은 결과적 일관성 (Eventual Consistency)
4. 하나의 트랜잭션에서 하나의 집합만 수정

### 2. Value Object (값 객체)

불변성과 자가 검증을 가진 값 객체를 Java Record로 구현합니다.

**Email Value Object (Shared Kernel):**
```java
public record Email(String address) {
    private static final Pattern EMAIL_PATTERN = Pattern.compile(
        "^[a-zA-Z0-9_!#$%&'*+/=?`{|}~^.-]+@[a-zA-Z0-9.-]+$"
    );

    public Email {
        if (address == null || address.isEmpty()) {
            throw new IllegalArgumentException("이메일을 입력해야 합니다");
        }
        if (!EMAIL_PATTERN.matcher(address).matches()) {
            throw new IllegalArgumentException("이메일 형식이 바르지 않습니다");
        }
    }
}
```

**Profile Value Object:**
```java
public record Profile(String address) {
    private static final Pattern PROFILE_ADDRESS_PATTERN = Pattern.compile(
        "^[a-zA-Z0-9._-]{3,}$"
    );

    public Profile {
        if (address == null ||
            (!address.isEmpty() && !PROFILE_ADDRESS_PATTERN.matcher(address).matches())) {
            throw new IllegalArgumentException("프로필 주소 형식이 바르지 않습니다");
        }
        if (address.length() > 15) {
            throw new IllegalArgumentException("프로필 주소는 최대 15자리를 넘을 수 없습니다");
        }
    }

    public String url() {
        return "@" + address;
    }
}
```

**값 객체의 특징:**
- **불변성 (Immutability)**: Java Record 사용
- **자가 검증 (Self-Validation)**: Compact Constructor에서 검증
- **값 동등성 (Value Equality)**: Record가 자동으로 제공
- **풍부한 행위 (Rich Behavior)**: `url()` 같은 도메인 메서드

**값 객체 사용 이점:**
1. 원시 타입 집착 (Primitive Obsession) 방지
2. 타입 안전성 향상
3. 검증 로직 중앙화
4. 도메인 개념 명시화

### 3. Entity (엔티티)

**MemberDetail Entity:**
```java
@Entity
public class MemberDetail extends AbstractEntity {
    private Profile profile;
    private LocalDate birthday;
    private MemberDetailStatus status;

    public static MemberDetail create() {
        MemberDetail detail = new MemberDetail();
        detail.profile = new Profile("");
        detail.status = MemberDetailStatus.PENDING;
        return detail;
    }

    public void activate() {
        this.status = MemberDetailStatus.ACTIVE;
    }

    public void updateInfo(MemberInfoUpdateRequest updateRequest) {
        this.profile = new Profile(updateRequest.profile());
        this.birthday = updateRequest.birthday();
    }
}
```

**엔티티의 특징:**
- 식별자 (Identity)로 구별
- 생명주기를 가짐
- 가변 (Mutable)
- 집합 루트가 아닌 경우 집합 경계 내에서만 존재

### 4. Domain Service (도메인 서비스)

**PasswordEncoder Domain Service:**
```java
public interface PasswordEncoder {
    String encode(String password);
    boolean matches(String password, String passwordHash);
}
```

**어댑터 구현:**
```java
@Component
public class SecurePasswordEncoder implements PasswordEncoder {
    private final BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();

    @Override
    public String encode(String password) {
        return bCryptPasswordEncoder.encode(password);
    }

    @Override
    public boolean matches(String password, String passwordHash) {
        return bCryptPasswordEncoder.matches(password, passwordHash);
    }
}
```

**도메인 서비스 사용 기준:**
- 특정 엔티티나 값 객체에 속하지 않는 비즈니스 로직
- 여러 집합을 조율하는 로직
- 외부 시스템 통합이 필요하지만 도메인 개념인 경우

### 5. Repository (리포지토리)

**Repository Port (Application Layer):**
```java
public interface MemberRepository extends Repository<Member, Long> {
    Member save(Member member);
    Optional<Member> findByEmail(Email email);
    Optional<Member> findById(Long memberId);

    @Query("select m from Member m where m.detail.profile = :profile")
    Optional<Member> findByProfile(Profile profile);
}
```

**리포지토리의 특징:**
- 컬렉션처럼 동작하는 인터페이스
- 도메인 객체 (Member)를 반환
- 도메인 값 객체 (Email, Profile)를 쿼리 파라미터로 사용
- Spring Data JPA가 자동으로 구현 제공
- 집합 단위로 저장/조회

**리포지토리 설계 원칙:**
1. 집합 루트마다 하나의 리포지토리
2. 컬렉션 지향 인터페이스
3. 도메인 언어 사용
4. 인프라 세부사항 숨김

### 6. Factory (팩토리)

**Static Factory Method Pattern:**
```java
public class Member {
    public static Member register(
        MemberRegisterRequest registerRequest,
        PasswordEncoder passwordEncoder
    ) {
        Member member = new Member();
        member.email = new Email(registerRequest.email());
        member.nickname = requireNonNull(registerRequest.nickname());
        member.passwordHash = requireNonNull(
            passwordEncoder.encode(registerRequest.password())
        );
        member.status = MemberStatus.PENDING;
        member.detail = MemberDetail.create();
        return member;
    }
}
```

**팩토리 사용 이점:**
1. 복잡한 생성 로직 캡슐화
2. 도메인 불변식 보장
3. 생성 의도 명확화 (`register`)
4. 생성자 오버로딩 문제 해결

---

## Hexagonal Architecture 상세

### Ports and Adapters Pattern

```
┌─────────────────────────────────────────────────────────┐
│                   External Systems                      │
│  (Web, CLI, DB, Message Queue, External APIs)          │
└────────────┬────────────────────────────┬───────────────┘
             │                            │
    ┌────────▼──────────┐        ┌───────▼────────────┐
    │ Primary Adapters  │        │ Secondary Adapters │
    │   (Driving)       │        │    (Driven)        │
    │  - MemberApi      │        │  - JPA Repository  │
    │  - CLI            │        │  - EmailSender     │
    └────────┬──────────┘        └───────▲────────────┘
             │                            │
    ┌────────▼────────────────────────────┴───────────┐
    │           Application Core                      │
    │  ┌──────────────────────────────────────┐      │
    │  │      Provided Ports (Inbound)        │      │
    │  │  - MemberRegister                    │      │
    │  │  - MemberFinder                      │      │
    │  └──────────────────────────────────────┘      │
    │  ┌──────────────────────────────────────┐      │
    │  │      Required Ports (Outbound)       │      │
    │  │  - MemberRepository                  │      │
    │  │  - EmailSender                       │      │
    │  └──────────────────────────────────────┘      │
    │  ┌──────────────────────────────────────┐      │
    │  │         Domain Model                 │      │
    │  │  - Member, MemberDetail              │      │
    │  │  - Email, Profile                    │      │
    │  └──────────────────────────────────────┘      │
    └─────────────────────────────────────────────────┘
```

### 1. Driving Ports (제공 포트 - Inbound)

애플리케이션이 외부에 제공하는 유스케이스 인터페이스입니다.

**MemberRegister Port:**
```java
public interface MemberRegister {
    Member register(@Valid MemberRegisterRequest registerRequest);
    Member activate(Long memberId);
    Member deactivate(Long memberId);
    Member updateInfo(Long memberId, @Valid MemberInfoUpdateRequest request);
}
```

**Implementation (Application Service):**
```java
@Service
@Transactional
@Validated
public class MemberModifyService implements MemberRegister {
    private final MemberFinder memberFinder;
    private final MemberRepository memberRepository;
    private final EmailSender emailSender;
    private final PasswordEncoder passwordEncoder;

    @Override
    public Member register(MemberRegisterRequest registerRequest) {
        // 1. Application Logic: 중복 체크
        checkDuplicateEmail(registerRequest);

        // 2. Domain Logic: 회원 생성
        Member member = Member.register(registerRequest, passwordEncoder);

        // 3. Infrastructure: 저장
        memberRepository.save(member);

        // 4. Integration: 이메일 발송
        sendWelcomeEmail(member);

        return member;
    }

    private void checkDuplicateEmail(MemberRegisterRequest registerRequest) {
        if (memberRepository.findByEmail(new Email(registerRequest.email())).isPresent()) {
            throw new DuplicateEmailException("이미 사용중인 이메일입니다");
        }
    }

    private void sendWelcomeEmail(Member member) {
        emailSender.send(
            member.getEmail(),
            "회원 가입을 환영합니다",
            "회원 가입이 완료되었습니다. 활성화를 진행해주세요."
        );
    }
}
```

### 2. Driven Ports (요구 포트 - Outbound)

애플리케이션이 외부 인프라에 요구하는 인터페이스입니다.

**MemberRepository Port:**
```java
public interface MemberRepository extends Repository<Member, Long> {
    Member save(Member member);
    Optional<Member> findByEmail(Email email);
    Optional<Member> findById(Long memberId);
    Optional<Member> findByProfile(Profile profile);
}
```

**EmailSender Port:**
```java
public interface EmailSender {
    void send(Email email, String subject, String body);
}
```

### 3. Primary Adapters (주도 어댑터)

외부에서 애플리케이션을 호출하는 어댑터입니다.

**REST API Adapter:**
```java
@RestController
public class MemberApi {
    private final MemberRegister memberRegister;
    private final MemberFinder memberFinder;

    public MemberApi(MemberRegister memberRegister, MemberFinder memberFinder) {
        this.memberRegister = memberRegister;
        this.memberFinder = memberFinder;
    }

    @PostMapping("/api/members")
    public MemberRegisterResponse register(
        @RequestBody @Valid MemberRegisterRequest request
    ) {
        Member member = memberRegister.register(request);
        return MemberRegisterResponse.of(member);
    }

    @PostMapping("/api/members/{memberId}/activate")
    public void activate(@PathVariable Long memberId) {
        memberRegister.activate(memberId);
    }

    @GetMapping("/api/members/{memberId}")
    public MemberResponse getMember(@PathVariable Long memberId) {
        Member member = memberFinder.findById(memberId);
        return MemberResponse.of(member);
    }
}
```

### 4. Secondary Adapters (피주도 어댑터)

애플리케이션이 호출하는 인프라 어댑터입니다.

**Email Adapter:**
```java
@Component
@Fallback
public class DummyEmailSender implements EmailSender {
    @Override
    public void send(Email email, String subject, String body) {
        System.out.println("DummyEmailSender send email: " + email);
        System.out.println("Subject: " + subject);
        System.out.println("Body: " + body);
    }
}
```

**Password Encoder Adapter:**
```java
@Component
public class SecurePasswordEncoder implements PasswordEncoder {
    private final BCryptPasswordEncoder bCryptPasswordEncoder = new BCryptPasswordEncoder();

    @Override
    public String encode(String password) {
        return bCryptPasswordEncoder.encode(password);
    }

    @Override
    public boolean matches(String password, String passwordHash) {
        return bCryptPasswordEncoder.matches(password, passwordHash);
    }
}
```

---

## 의존성 규칙 (Dependency Rules)

### ArchUnit으로 강제되는 아키텍처 규칙

```java
@ArchTest
void haxagonalArchitecture(JavaClasses classes) {
    Architectures.layeredArchitecture()
        .consideringAllDependencies()
        .layer("domain").definedBy("tobyspring.splearn.domain..")
        .layer("application").definedBy("tobyspring.splearn.application..")
        .layer("adapter").definedBy("tobyspring.splearn.adapter..")

        .whereLayer("domain").mayOnlyBeAccessedByLayers("application", "adapter")
        .whereLayer("application").mayOnlyBeAccessedByLayers("adapter")
        .whereLayer("adapter").mayNotBeAccessedByAnyLayer()

        .check(classes);
}
```

### 의존성 방향

```
Adapter Layer (어댑터)
    ↓ (의존)
Application Layer (애플리케이션)
    ↓ (의존)
Domain Layer (도메인)
    ↓ (의존 없음)
```

### 계층별 의존성 규칙

| 계층 | 의존 가능한 계층 | 접근 가능한 계층 |
|------|-----------------|-----------------|
| Domain | Java 표준 라이브러리만 | Application, Adapter |
| Application | Domain | Adapter |
| Adapter | Application, Domain | 없음 (최상위) |

**핵심 원칙:**
1. **의존성 역전 원칙 (Dependency Inversion)**: 애플리케이션이 포트(인터페이스)를 정의하고, 어댑터가 구현
2. **안정된 추상화 (Stable Abstractions)**: 도메인은 가장 안정적이며 추상적
3. **변경의 방향 (Direction of Change)**: 변경은 외부에서 내부로 전파되지 않음

---

## Bounded Context (경계 컨텍스트)

### 현재 구현된 Bounded Context

#### Member Bounded Context
- **목적**: 회원 가입, 활성화, 비활성화, 프로필 관리
- **Aggregate Root**: Member
- **Entities**: Member, MemberDetail
- **Value Objects**: Email (공유), Profile, MemberStatus
- **Domain Services**: PasswordEncoder
- **Use Cases**:
  - MemberRegister (회원 등록, 활성화, 비활성화, 정보 수정)
  - MemberFinder (회원 조회)

### 향후 확장 가능한 Bounded Context

```
┌──────────────────────────────────────────────────────────┐
│                   Learning Platform                      │
└──────────────────────────────────────────────────────────┘
        │                   │                   │
┌───────▼────────┐  ┌──────▼──────┐  ┌────────▼─────────┐
│    Member      │  │   Course     │  │   Enrollment     │
│   Context      │  │   Context    │  │    Context       │
│                │  │              │  │                  │
│ - Member       │  │ - Course     │  │ - Enrollment     │
│ - Profile      │  │ - Lesson     │  │ - Progress       │
│ - Email        │  │ - Instructor │  │ - Certificate    │
└────────────────┘  └──────────────┘  └──────────────────┘
```

**Context Map:**
- Member → Enrollment: **Customer/Supplier** (고객-공급자)
- Course → Enrollment: **Customer/Supplier**
- Member ↔ Course: **Shared Kernel** (Email)

---

## 영속성 전략 (Persistence Strategy)

### XML 기반 ORM 매핑

도메인 모델을 프레임워크로부터 독립적으로 유지하기 위해 **XML 기반 ORM 매핑**을 사용합니다.

**orm.xml 예시:**
```xml
<entity-mappings>
    <entity class="tobyspring.splearn.domain.member.Member">
        <table name="member">
            <unique-constraint name="UK_MEMBER_EMAIL_ADDRESS">
                <column-name>email_address</column-name>
            </unique-constraint>
        </table>
        <attributes>
            <one-to-one name="detail" fetch="LAZY">
                <join-column name="detail_id"/>
                <cascade>
                    <cascade-all/>
                </cascade>
            </one-to-one>
            <embedded name="email">
                <attribute-override name="address">
                    <column name="email_address" length="100"/>
                </attribute-override>
            </embedded>
        </attributes>
    </entity>

    <embeddable class="tobyspring.splearn.domain.member.Profile">
        <attributes>
            <basic name="address">
                <column name="profile_address" length="20"/>
            </basic>
        </attributes>
    </embeddable>

    <embeddable class="tobyspring.splearn.domain.shared.Email">
        <attributes>
            <basic name="address">
                <column name="email_address" length="100"/>
            </basic>
        </attributes>
    </embeddable>
</entity-mappings>
```

**XML 매핑의 장점:**
1. 도메인 모델이 JPA 어노테이션으로 오염되지 않음
2. 영속성 로직과 도메인 로직의 분리
3. 도메인 모델의 순수성 유지
4. 테스트 용이성 향상

### JPA Repository 자동 구현

```java
public interface MemberRepository extends Repository<Member, Long> {
    Member save(Member member);
    Optional<Member> findByEmail(Email email);
    Optional<Member> findById(Long memberId);

    @Query("select m from Member m where m.detail.profile = :profile")
    Optional<Member> findByProfile(Profile profile);
}
```

Spring Data JPA가 인터페이스만으로 자동 구현을 제공합니다.

---

## 테스팅 전략 (Testing Strategy)

### 1. Domain Layer Tests (순수 단위 테스트)

```java
class MemberTest {
    private PasswordEncoder passwordEncoder = new TestPasswordEncoder();

    @Test
    void register() {
        Member member = Member.register(
            MemberFixture.createMemberRegisterRequest(),
            passwordEncoder
        );

        assertThat(member.getNickname()).isEqualTo("nickname");
        assertThat(member.getStatus()).isEqualTo(MemberStatus.PENDING);
    }

    @Test
    void activate() {
        Member member = Member.register(
            MemberFixture.createMemberRegisterRequest(),
            passwordEncoder
        );

        member.activate();

        assertThat(member.getStatus()).isEqualTo(MemberStatus.ACTIVE);
        assertThat(member.getDetail().getStatus()).isEqualTo(MemberDetailStatus.ACTIVE);
    }

    @Test
    void updateInfo_whenNotActive_throwsException() {
        Member member = Member.register(
            MemberFixture.createMemberRegisterRequest(),
            passwordEncoder
        );

        assertThatThrownBy(() ->
            member.updateInfo(MemberFixture.createMemberInfoUpdateRequest())
        ).isInstanceOf(IllegalStateException.class)
         .hasMessage("등록 완료 상태가 아니면 정보를 수정할 수 없습니다");
    }
}
```

**특징:**
- 프레임워크 의존성 없음
- 빠른 실행 속도
- 비즈니스 로직에 집중
- Given-When-Then 구조

### 2. Application Layer Tests (통합 테스트)

```java
@SpringBootTest
@Transactional
record MemberRegisterTest(
    MemberRegister memberRegister,
    MemberFinder memberFinder,
    EntityManager entityManager
) {
    @Test
    void register() {
        // Given
        MemberRegisterRequest request = MemberFixture.createMemberRegisterRequest();

        // When
        Member member = memberRegister.register(request);

        // Then
        assertThat(member.getId()).isNotNull();
        assertThat(member.getStatus()).isEqualTo(MemberStatus.PENDING);

        // Verify persistence
        entityManager.flush();
        entityManager.clear();

        Member found = memberFinder.findById(member.getId());
        assertThat(found.getEmail()).isEqualTo(member.getEmail());
    }

    @Test
    void register_withDuplicateEmail_throwsException() {
        // Given
        MemberRegisterRequest request = MemberFixture.createMemberRegisterRequest();
        memberRegister.register(request);

        // When & Then
        assertThatThrownBy(() -> memberRegister.register(request))
            .isInstanceOf(DuplicateEmailException.class)
            .hasMessage("이미 사용중인 이메일입니다");
    }
}
```

**특징:**
- Spring Context 로딩
- 실제 데이터베이스 사용 (H2)
- 트랜잭션 롤백 (@Transactional)
- 유스케이스 전체 흐름 검증

### 3. Adapter Layer Tests (웹 MVC 테스트)

```java
@SpringBootTest
@AutoConfigureMockMvc
class MemberApiTest {
    @Autowired
    private MockMvcTester mvcTester;

    @Test
    void register() throws Exception {
        // Given
        String requestJson = """
            {
                "email": "test@example.com",
                "nickname": "tester",
                "password": "password123"
            }
            """;

        // When
        MvcTestResult result = mvcTester.post().uri("/api/members")
            .contentType(MediaType.APPLICATION_JSON)
            .content(requestJson)
            .exchange();

        // Then
        assertThat(result).hasStatusOk();
        assertThat(result).bodyJson()
            .extractingPath("$.email").asString()
            .isEqualTo("test@example.com");
    }

    @Test
    void register_withInvalidEmail_returnsBadRequest() throws Exception {
        // Given
        String requestJson = """
            {
                "email": "invalid-email",
                "nickname": "tester",
                "password": "password123"
            }
            """;

        // When
        MvcTestResult result = mvcTester.post().uri("/api/members")
            .contentType(MediaType.APPLICATION_JSON)
            .content(requestJson)
            .exchange();

        // Then
        assertThat(result).hasStatus(HttpStatus.BAD_REQUEST);
    }
}
```

**특징:**
- HTTP 요청/응답 검증
- JSON 직렬화/역직렬화 검증
- 예외 처리 검증
- End-to-End 시나리오 테스트

### 테스트 피라미드

```
        ┌─────────┐
        │   E2E   │  ← 소수 (Adapter Layer)
        ├─────────┤
        │Integration│ ← 중간 (Application Layer)
        ├─────────┤
        │   Unit   │  ← 다수 (Domain Layer)
        └─────────┘
```

---

## 예외 처리 전략

### 1. Domain Exceptions

```java
public class DuplicateEmailException extends RuntimeException {
    public DuplicateEmailException(String message) {
        super(message);
    }
}

public class DuplicateProfileException extends RuntimeException {
    public DuplicateProfileException(String message) {
        super(message);
    }
}

public class MemberNotFoundException extends RuntimeException {
    public MemberNotFoundException(String message) {
        super(message);
    }
}
```

### 2. Global Exception Handler

```java
@RestControllerAdvice
public class ApiControllerAdvice {

    @ExceptionHandler(DuplicateEmailException.class)
    public ProblemDetail handleDuplicateEmail(DuplicateEmailException e) {
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
            HttpStatus.CONFLICT,
            e.getMessage()
        );
        problemDetail.setType(URI.create("https://example.com/problems/duplicate-email"));
        problemDetail.setTitle("Duplicate Email");
        return problemDetail;
    }

    @ExceptionHandler(MemberNotFoundException.class)
    public ProblemDetail handleMemberNotFound(MemberNotFoundException e) {
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND,
            e.getMessage()
        );
        problemDetail.setType(URI.create("https://example.com/problems/member-not-found"));
        problemDetail.setTitle("Member Not Found");
        return problemDetail;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationErrors(MethodArgumentNotValidException e) {
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST,
            "Validation failed"
        );
        problemDetail.setType(URI.create("https://example.com/problems/validation-error"));
        problemDetail.setTitle("Validation Error");

        List<String> errors = e.getBindingResult().getAllErrors().stream()
            .map(ObjectError::getDefaultMessage)
            .toList();
        problemDetail.setProperty("errors", errors);

        return problemDetail;
    }
}
```

**ProblemDetail (RFC 7807) 응답 예시:**
```json
{
  "type": "https://example.com/problems/duplicate-email",
  "title": "Duplicate Email",
  "status": 409,
  "detail": "이미 사용중인 이메일입니다"
}
```

---

## 아키텍처 의사결정 기록 (ADR)

### ADR-001: Hexagonal Architecture 채택

**상황 (Context):**
- 비즈니스 로직과 기술 세부사항을 분리해야 함
- 테스트 가능성과 유지보수성이 중요
- 외부 시스템 변경에 유연하게 대응 필요

**결정 (Decision):**
Hexagonal Architecture (Ports and Adapters)를 채택

**근거 (Rationale):**
- 비즈니스 로직이 프레임워크로부터 독립적
- 포트를 통한 명확한 경계 정의
- 어댑터 교체 용이 (테스트용 Mock, 실제 구현 등)
- 의존성 역전을 통한 유연성 확보

**결과 (Consequences):**
- ✅ 단위 테스트 작성 용이
- ✅ 외부 시스템 변경 시 영향 최소화
- ❌ 초기 구조 설계 복잡도 증가
- ❌ 인터페이스 증가로 파일 수 증가

### ADR-002: Value Objects as Records

**상황:**
- 불변 객체 필요
- 값 동등성 비교 필요
- 자가 검증 로직 필요

**결정:**
Java 17+ Record를 Value Object 구현에 사용

**근거:**
- 불변성 자동 보장
- 값 동등성 자동 구현 (equals, hashCode)
- Compact Constructor로 검증 로직 추가 가능
- 간결한 문법

**결과:**
- ✅ 보일러플레이트 코드 감소
- ✅ 불변성 보장
- ✅ 검증 로직 중앙화
- ❌ Record는 상속 불가

### ADR-003: XML-based ORM Mapping

**상황:**
- 도메인 모델을 순수하게 유지하고 싶음
- JPA 어노테이션이 도메인 코드를 오염시킴

**결정:**
XML 기반 ORM 매핑 사용 (orm.xml)

**근거:**
- 도메인 모델과 영속성 로직 분리
- 도메인 계층의 프레임워크 독립성 유지
- 테스트 용이성 향상

**결과:**
- ✅ 도메인 모델 순수성 유지
- ✅ 영속성 로직 중앙 관리
- ❌ XML 파일 관리 필요
- ❌ IDE 지원 부족

### ADR-004: Application Layer에 Port 정의

**상황:**
- 애플리케이션과 외부 시스템 간 계약 필요
- 의존성 역전 원칙 적용

**결정:**
Application Layer에 Port (인터페이스) 정의

**근거:**
- 애플리케이션이 필요한 것을 정의 (주도권)
- 어댑터는 애플리케이션의 요구사항을 구현
- 의존성 역전 원칙 준수

**결과:**
- ✅ 애플리케이션 중심 설계
- ✅ 테스트 시 Mock 구현 용이
- ✅ 어댑터 교체 가능
- ❌ 인터페이스와 구현 분리로 복잡도 증가

---

## 설계 원칙 및 Best Practices

### SOLID 원칙 적용

#### 1. Single Responsibility Principle (SRP)
```java
// ✅ Good: 책임 분리
public class Member {  // 도메인 로직만
    public void activate() { ... }
}

@Service
public class MemberModifyService {  // 유스케이스 조율만
    public Member activate(Long memberId) {
        Member member = memberFinder.findById(memberId);
        member.activate();
        return memberRepository.save(member);
    }
}
```

#### 2. Open/Closed Principle (OCP)
```java
// ✅ Good: 확장에 열려있고 수정에 닫혀있음
public interface EmailSender {
    void send(Email email, String subject, String body);
}

// 새로운 구현 추가 시 기존 코드 변경 없음
@Component
public class SmtpEmailSender implements EmailSender { ... }

@Component
public class SendGridEmailSender implements EmailSender { ... }
```

#### 3. Liskov Substitution Principle (LSP)
```java
// ✅ Good: 인터페이스 계약 준수
public interface MemberRegister {
    Member register(@Valid MemberRegisterRequest request);
}

// 어떤 구현체를 사용하더라도 동일하게 동작
@Service
public class MemberModifyService implements MemberRegister { ... }
```

#### 4. Interface Segregation Principle (ISP)
```java
// ✅ Good: 역할별 인터페이스 분리
public interface MemberRegister {  // 쓰기 작업
    Member register(MemberRegisterRequest request);
    Member activate(Long memberId);
}

public interface MemberFinder {  // 읽기 작업
    Member findById(Long memberId);
    Member findByEmail(Email email);
}
```

#### 5. Dependency Inversion Principle (DIP)
```java
// ✅ Good: 추상화에 의존
@Service
public class MemberModifyService implements MemberRegister {
    private final MemberRepository memberRepository;  // 인터페이스에 의존
    private final EmailSender emailSender;            // 인터페이스에 의존

    // 구체적인 구현은 Spring이 주입
}
```

### DDD 설계 원칙

#### 1. Ubiquitous Language (보편 언어)
- 도메인 전문가와 개발자가 동일한 용어 사용
- 코드에 도메인 개념을 그대로 표현

```java
// ✅ Good: 도메인 언어 사용
public class Member {
    public void activate() { ... }      // "활성화하다"
    public void deactivate() { ... }    // "비활성화하다"
    public void updateInfo(...) { ... } // "정보를 수정하다"
}

// ❌ Bad: 기술 용어 사용
public class Member {
    public void setStatusToActive() { ... }
    public void changeStatusInactive() { ... }
}
```

#### 2. Model-Driven Design
- 비즈니스 로직은 도메인 모델에 위치
- 서비스는 얇게, 도메인 모델은 풍부하게

```java
// ✅ Good: Rich Domain Model
public class Member {
    public void activate() {
        state(status == MemberStatus.PENDING, "PENDING 상태가 아닙니다");
        this.status = MemberStatus.ACTIVE;
        this.detail.activate();
    }
}

// ❌ Bad: Anemic Domain Model
public class Member {
    private MemberStatus status;
    public void setStatus(MemberStatus status) {
        this.status = status;
    }
}

@Service
public class MemberService {
    public void activate(Member member) {
        if (member.getStatus() != MemberStatus.PENDING) {
            throw new IllegalStateException(...);
        }
        member.setStatus(MemberStatus.ACTIVE);
    }
}
```

#### 3. Aggregates 설계 원칙

**작은 집합 (Small Aggregates):**
```java
// ✅ Good: 작은 집합
public class Member {  // Aggregate Root
    private MemberDetail detail;  // 내부 엔티티
}

// ❌ Bad: 큰 집합
public class Member {
    private List<Order> orders;           // 다른 집합!
    private List<Payment> payments;       // 다른 집합!
    private List<Address> addresses;      // 다른 집합!
}
```

**ID로 참조 (Reference by ID):**
```java
// ✅ Good: ID로 참조
public class Order {
    private Long memberId;  // Member 집합을 ID로 참조
}

// ❌ Bad: 직접 참조
public class Order {
    private Member member;  // 집합 경계 위반!
}
```

#### 4. 불변식 보호 (Invariant Protection)

```java
public class Member {
    // 불변식: PENDING 상태에서만 활성화 가능
    public void activate() {
        state(status == MemberStatus.PENDING, "PENDING 상태가 아닙니다");
        this.status = MemberStatus.ACTIVE;
        this.detail.activate();
    }

    // 불변식: ACTIVE 상태에서만 정보 수정 가능
    public void updateInfo(MemberInfoUpdateRequest request) {
        state(status == MemberStatus.ACTIVE,
              "등록 완료 상태가 아니면 정보를 수정할 수 없습니다");
        this.nickname = Objects.requireNonNull(request.nickname());
        this.detail.updateInfo(request);
    }

    private void state(boolean condition, String message) {
        if (!condition) {
            throw new IllegalStateException(message);
        }
    }
}
```

---

## 향후 개선 방향

### 1. Domain Events (도메인 이벤트)

**현재 문제:**
- 회원 등록 후 이메일 발송 로직이 애플리케이션 서비스에 직접 구현됨
- 집합 간 통신이 동기적이고 강결합

**개선 방안:**
```java
// Member Aggregate에서 이벤트 발행
public class Member {
    public static Member register(...) {
        Member member = new Member();
        // ... 초기화
        member.registerEvent(new MemberRegisteredEvent(member));
        return member;
    }
}

// Event Handler에서 처리
@EventListener
public class MemberEventHandler {
    @Async
    @TransactionalEventListener
    public void handleMemberRegistered(MemberRegisteredEvent event) {
        emailSender.sendWelcomeEmail(event.getMember().getEmail());
    }
}
```

**기대 효과:**
- 집합 간 느슨한 결합
- 비동기 처리 가능
- 새로운 부가 기능 추가 용이 (알림, 로그 등)

### 2. CQRS (Command Query Responsibility Segregation)

**현재 상황:**
- 읽기와 쓰기가 동일한 모델 사용
- 복잡한 조회 쿼리 시 성능 이슈 가능

**개선 방안:**
```java
// Command Model (Write)
@Service
public class MemberCommandService {
    public void register(RegisterMemberCommand command) { ... }
}

// Query Model (Read)
@Service
public class MemberQueryService {
    public MemberDetailView getMemberDetail(Long memberId) {
        // 조회 전용 DTO 반환
        // 복잡한 조인 쿼리 가능
    }
}
```

**기대 효과:**
- 읽기 성능 최적화
- 쓰기 모델 단순화
- 각 모델 독립적 확장 가능

### 3. Specification Pattern (명세 패턴)

**현재 문제:**
- 복잡한 비즈니스 규칙이 메서드에 흩어져 있음

**개선 방안:**
```java
public interface MemberSpecification {
    boolean isSatisfiedBy(Member member);
}

public class ActiveMemberSpecification implements MemberSpecification {
    @Override
    public boolean isSatisfiedBy(Member member) {
        return member.getStatus() == MemberStatus.ACTIVE;
    }
}

// 사용
if (new ActiveMemberSpecification().isSatisfiedBy(member)) {
    // ...
}
```

**기대 효과:**
- 비즈니스 규칙 재사용
- 명확한 비즈니스 의도 표현
- 조합 가능한 규칙

### 4. Outbox Pattern (신뢰성 있는 이벤트 발행)

**현재 문제:**
- 데이터 저장과 이벤트 발행의 원자성 미보장

**개선 방안:**
```java
@Entity
public class OutboxEvent {
    private String eventType;
    private String payload;
    private LocalDateTime createdAt;
    private EventStatus status;
}

// 트랜잭션 내에서 Outbox에 이벤트 저장
@Transactional
public void register(MemberRegisterRequest request) {
    Member member = memberRepository.save(...);
    outboxRepository.save(new OutboxEvent("MemberRegistered", ...));
}

// 별도 스케줄러가 Outbox를 폴링하여 이벤트 발행
@Scheduled
public void publishEvents() {
    List<OutboxEvent> events = outboxRepository.findPending();
    events.forEach(event -> {
        eventPublisher.publish(event);
        outboxRepository.markPublished(event);
    });
}
```

**기대 효과:**
- 데이터 일관성 보장
- 이벤트 발행 신뢰성 향상
- 재시도 메커니즘 구현 가능

### 5. 모듈 분리 (Multi-Module Gradle Project)

**향후 확장 시:**
```
di-related-services/
├── member-module/
│   ├── member-domain/
│   ├── member-application/
│   └── member-adapter/
├── course-module/
│   ├── course-domain/
│   ├── course-application/
│   └── course-adapter/
├── enrollment-module/
│   ├── enrollment-domain/
│   ├── enrollment-application/
│   └── enrollment-adapter/
└── shared-kernel/
    └── common-types/
```

**기대 효과:**
- Bounded Context별 독립 배포
- 명확한 모듈 경계
- 팀 단위 개발 용이

---

## 참고 자료

### DDD 핵심 개념
- **Aggregate**: 트랜잭션 일관성 경계를 가진 도메인 객체의 클러스터
- **Value Object**: 식별자 없이 속성으로만 구별되는 불변 객체
- **Entity**: 식별자로 구별되는 생명주기를 가진 객체
- **Domain Service**: 특정 엔티티나 값 객체에 속하지 않는 도메인 로직
- **Repository**: 집합을 저장/조회하는 컬렉션 지향 인터페이스
- **Bounded Context**: 특정 도메인 모델이 정의되고 적용되는 경계

### Hexagonal Architecture 핵심 개념
- **Port**: 애플리케이션과 외부 세계 간의 인터페이스
- **Adapter**: 포트를 구현하는 구체적인 기술
- **Primary/Driving**: 애플리케이션을 호출하는 쪽
- **Secondary/Driven**: 애플리케이션이 호출하는 쪽

### 추천 도서
- **"Domain-Driven Design"** by Eric Evans
- **"Implementing Domain-Driven Design"** by Vaughn Vernon
- **"Get Your Hands Dirty on Clean Architecture"** by Tom Hombergs

---

## 마치며

이 프로젝트는 **DDD와 Hexagonal Architecture를 실천적으로 적용한 예시**입니다. 핵심은 다음과 같습니다:

1. **도메인 중심 설계**: 비즈니스 로직을 도메인 모델에 집중
2. **명확한 경계**: 계층 간 의존성 규칙 엄격히 준수
3. **테스트 가능성**: 프레임워크로부터 독립적인 도메인 로직
4. **유연성**: 포트와 어댑터를 통한 기술 스택 교체 용이성

이러한 원칙들을 지키면서 프로젝트를 확장해 나가면, 유지보수가 쉽고 비즈니스 변화에 빠르게 대응할 수 있는 시스템을 구축할 수 있습니다.
