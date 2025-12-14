# Architecture Reference Skill

Java/Spring Boot 프로젝트 아키텍처 가이드라인

## 레이어드 아키텍처

```
+-------------------------------------+
|           Presentation              |  Controller, DTO
+-------------------------------------+
|            Application              |  Service, UseCase
+-------------------------------------+
|              Domain                 |  Entity, Repository Interface
+-------------------------------------+
|           Infrastructure            |  Repository Impl, External API
+-------------------------------------+
```

## 패키지 구조

### 레이어 기반 구조

```
src/main/java/com/example/
├── controller/
├── service/
├── repository/
├── domain/
├── dto/
└── config/
```

### 도메인 기반 구조 (권장)

```
src/main/java/com/example/
├── user/
│   ├── application/
│   ├── domain/
│   ├── infrastructure/
│   └── presentation/
├── order/
│   ├── application/
│   ├── domain/
│   ├── infrastructure/
│   └── presentation/
└── common/
```

## 레이어별 책임

### Presentation Layer

- HTTP 요청/응답 처리
- 입력 검증 (`@Valid`)
- DTO 변환
- 예외 처리 (`@ControllerAdvice`)

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {
    private final UserService userService;

    @PostMapping
    public ResponseEntity<UserResponse> create(@Valid @RequestBody UserRequest request) {
        return ResponseEntity.ok(userService.create(request));
    }
}
```

### Application Layer

- 비즈니스 유스케이스 조합
- 트랜잭션 관리
- 도메인 서비스 호출

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;

    @Transactional
    public UserResponse create(UserRequest request) {
        User user = User.create(request.name(), request.email());
        return UserResponse.from(userRepository.save(user));
    }
}
```

### Domain Layer

- 핵심 비즈니스 로직
- 엔티티, 값 객체
- 도메인 이벤트
- Repository 인터페이스

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private String email;

    public static User create(String name, String email) {
        // 도메인 규칙 검증
        return new User(name, email);
    }
}
```

### Infrastructure Layer

- 외부 시스템 연동
- Repository 구현
- 기술적 세부사항

## 설계 원칙

### SOLID

- **S**ingle Responsibility: 하나의 책임만 가짐
- **O**pen/Closed: 확장에 열림, 수정에 닫힘
- **L**iskov Substitution: 하위 타입 대체 가능
- **I**nterface Segregation: 인터페이스 분리
- **D**ependency Inversion: 추상화에 의존

### 의존성 규칙

- 상위 레이어 -> 하위 레이어 의존
- Domain은 다른 레이어에 의존하지 않음
- Infrastructure -> Domain (인터페이스 구현)

## DTO 패턴

```java
// Request DTO
public record UserRequest(
    @NotBlank String name,
    @Email String email
) {}

// Response DTO
public record UserResponse(
    Long id,
    String name,
    String email
) {
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getName(), user.getEmail());
    }
}
```

## 예외 처리

```java
// 도메인 예외
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found: " + id);
    }
}

// 전역 예외 처리
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(UserNotFoundException e) {
        return ResponseEntity.status(NOT_FOUND)
            .body(new ErrorResponse(e.getMessage()));
    }
}
```

## 테스트 전략

| 레이어 | 테스트 유형 | 도구 |
|--------|-------------|------|
| Presentation | 슬라이스 테스트 | `@WebMvcTest` |
| Application | 단위/통합 테스트 | Mockito, `@SpringBootTest` |
| Domain | 단위 테스트 | JUnit5, AssertJ |
| Infrastructure | 통합 테스트 | `@DataJpaTest`, Testcontainers |
