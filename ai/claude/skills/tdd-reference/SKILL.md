# TDD Reference Skill

Java/JUnit5 기반 TDD(Test-Driven Development) 가이드라인

## TDD 사이클

1. **Red**: 실패하는 테스트 작성
2. **Green**: 테스트를 통과하는 최소한의 코드 작성
3. **Refactor**: 코드 개선 (테스트는 계속 통과해야 함)

## 테스트 작성 원칙

### FIRST 원칙

- **Fast**: 테스트는 빠르게 실행되어야 함
- **Independent**: 테스트 간 의존성 없음
- **Repeatable**: 어떤 환경에서도 동일한 결과
- **Self-validating**: 성공/실패가 명확함
- **Timely**: 프로덕션 코드 전에 작성

### 테스트 구조 (Given-When-Then)

```java
@Test
@DisplayName("설명적인 테스트 이름")
void shouldDoSomething_whenCondition() {
    // Given (준비)
    var input = createTestInput();

    // When (실행)
    var result = sut.execute(input);

    // Then (검증)
    assertThat(result).isEqualTo(expected);
}
```

## JUnit5 주요 어노테이션

| 어노테이션 | 용도 |
|-----------|------|
| `@Test` | 테스트 메서드 |
| `@DisplayName` | 테스트 설명 |
| `@BeforeEach` | 각 테스트 전 실행 |
| `@AfterEach` | 각 테스트 후 실행 |
| `@BeforeAll` | 모든 테스트 전 1회 실행 |
| `@Nested` | 테스트 그룹화 |
| `@ParameterizedTest` | 파라미터화된 테스트 |
| `@Tag` | 테스트 분류 |

## 테스트 분류

### 단위 테스트 (Unit Test)

- 위치: `src/test/java`
- 네이밍: `*Test.java`
- 외부 의존성 모킹
- 빠른 실행 속도

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;
}
```

### 통합 테스트 (Integration Test)

- 위치: `src/test/java`
- 네이밍: `*IntegrationTest.java` 또는 `*IT.java`
- 실제 의존성 사용
- `@SpringBootTest` 활용

```java
@SpringBootTest
@Transactional
class UserServiceIntegrationTest {
    @Autowired
    private UserService userService;
}
```

## AssertJ 활용

```java
// 기본 검증
assertThat(actual).isEqualTo(expected);
assertThat(actual).isNotNull();

// 컬렉션 검증
assertThat(list).hasSize(3);
assertThat(list).contains(item);
assertThat(list).extracting("name").containsExactly("a", "b");

// 예외 검증
assertThatThrownBy(() -> service.execute())
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("expected message");
```

## 테스트 더블

| 유형 | 용도 |
|------|------|
| Stub | 미리 정의된 응답 반환 |
| Mock | 호출 검증 |
| Spy | 실제 객체 + 부분 모킹 |
| Fake | 간단한 구현체 |

## 테스트 커버리지 기준

- 라인 커버리지: 80% 이상
- 브랜치 커버리지: 70% 이상
- 핵심 비즈니스 로직: 90% 이상
