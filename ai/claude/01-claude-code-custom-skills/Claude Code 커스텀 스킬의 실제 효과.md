# Claude Code 커스텀 스킬의 실제 효과

- 동일 요구사항, 다른 결과물

## 들어가며

Claude Code로 코드를 생성할 때 프롬프트만 잘 쓰면 된다고 생각했다. 스킬 설정은 토큰만 소모하고 별 차이가 없어 보였다. 그래서 직접 테스트해봤다. 같은 "태스크 관리 CLI 앱" 요구사항으로 두 가지 방식을 비교해보니 결과물 차이가 생각보다 컸다.

- **Project-1**: 커스텀 스킬(TDD 가이드, 아키텍처 레퍼런스)과 에이전트를 설정한 후 생성
- **Project-2**: 별도 설정 없이 프롬프트로만 생성

결론부터 말하면, 스킬 설정 유무에 따라 "동작하는 코드"와 "설계된 코드"의 차이가 났다.

---

## 실험 환경

- agent, skills, commands는 [Kent Beck의 글](https://tidyfirst.substack.com/p/augmented-coding-beyond-the-vibes/)을 참고하여 작성했다.
- 사용된 스킬은 본 문서와 같은 디렉토리의 `agents/`, `commands/`, `skills/` 폴더에서 확인할 수 있다.

### 사용한 스킬 구성

```
.claude/
├── agents/
│   └── tdd-developer.md      # TDD 전문가 에이전트
├── commands/
│   └── tdd-check.md          # TDD 체크리스트
└── skills/
    ├── tdd-reference/
    │   └── SKILL.md          # TDD 원칙, 테스트 전략
    └── architecture-reference/
        └── SKILL.md          # SOLID, 디자인 패턴
```

에이전트 설정에는 다음 지침이 포함되어 있다:

> "작업 시작 전에 TDD 원칙 및 품질 기준, 아키텍처 가이드를 읽어라"
> "항상 한 번에 하나의 테스트를 작성하고, 그 테스트를 실행 가능하게 만든 다음, 구조를 개선한다"

### 동일 요구사항

두 프로젝트 모두 동일한 프롬프트로 시작했다:

```
JSON 파일을 저장소로 사용하는 CLI 태스크 관리 도구를 만들어줘.

- 태스크 CRUD (생성, 조회, 수정, 삭제)
- 우선순위, 마감일, 태그 지원
- 태스크 필터링 및 검색
- 상태 변경 이력 추적
- 통계 리포트 생성
```

구현된 기능:

- 태스크 CRUD
- 상태/우선순위/태그 관리
- 검색 및 필터링
- 통계 리포트
- JSON 파일 저장

---

## 결과 비교

### 1. Repository 설계

**Project-1 (스킬 적용)**

```java
public interface TaskRepository {
    void save(Task task);

    Optional<Task> findById(String id);

    List<Task> findByStatus(TaskStatus status);

    List<Task> findByPriority(TaskPriority priority);
    // ...
}

public class JsonTaskRepository implements TaskRepository {
    // 구현
}
```

**Project-2 (프롬프트만)**

```java
public class TaskRepository {
    private final Path filePath;
    private List<Task> tasks;

    public Task create(Task task) { ...}

    public Optional<Task> findById(String id) { ...}
    // ...
}
```

스킬에 "구체 클래스가 아닌 추상화에 의존해야 한다(DIP)"라고 명시했더니 실제로 인터페이스 분리가 적용됐다. Project-2는 구체 클래스를 직접 사용해서 테스트할 때 Mock 처리가 어렵다.

### 2. 도메인 객체 설계

**Project-1**

```java
public class Task {
    private final String id;  // 불변
    private String title;

    private Task(Builder builder) {
        this.id = Objects.requireNonNull(builder.id);
        this.title = validateTitle(builder.title);
        // ...
    }

    public static Builder builder() {
        return new Builder();
    }
}
```

**Project-2**

```java
public class Task {
    private String id;

    public void setId(String id) {
        this.id = id;
    }

    public void setTitle(String title) { ...}
    // 모든 필드에 Setter
}
```

스킬의 빌더 패턴 예시와 "방어적 프로그래밍" 조항이 반영됐다. Project-2는 Setter가 전면 개방되어 있어서 객체 상태를 아무 데서나 바꿀 수 있다.

### 3. 테스트 전략

**Project-1**: Mock 기반 단위 테스트

```java

@ExtendWith(MockitoExtension.class)
class TaskServiceTest {
    @Mock
    private TaskRepository repository;

    @Test
    void shouldCreateTask() {
        // Given - When - Then
        verify(repository, times(1)).save(task);
    }
}
```

**Project-2**: 실제 구현체 기반 테스트

```java
class TaskServiceTest {
    @BeforeEach
    void setUp() {
        repository = new TaskRepository(testFile);  // 실제 구현체
        service = new TaskService(repository);
    }
}
```

스킬에 "의존성 주입으로 테스트 용이성 확보"라고 적어뒀는데, Project-1만 Mock을 활용한 순수 단위 테스트를 작성했다.

다만 Project-2 방식도 장점이 있다. 실제 구현체를 사용하면 통합 수준의 신뢰성을 얻을 수 있고, Mock 설정 없이 테스트가 간결해진다. 상황에 따라 적절한 전략을 선택해야 한다.

### 4. 빌드 설정

**Project-1의 build.gradle**

```gradle
plugins {
    id 'jacoco'
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = 0.8
            }
        }
    }
}
```

**Project-2의 build.gradle**

```gradle
// JaCoCo 설정 없음
```

스킬에 "커버리지 80% 이상"을 명시했더니 실제로 JaCoCo 설정이 포함됐다.

---

## 작업 과정 비교

코드 결과물뿐 아니라 AI의 작업 로그를 분석한 결과, 개발 과정에서도 명확한 차이가 나타났다.

### 작업 순서 비교

**Project-1 (TDD 스킬 적용)**

1. 테스트 작성 (RED)
2. 테스트 실행 - 실패 확인
3. 최소 구현 (GREEN)
4. 테스트 실행 - 통과 확인
5. 리팩토링 (REFACTOR)
6. 다음 기능으로 이동

**Project-2 (프롬프트만)**

1. 프로젝트 구조 생성
2. 도메인 모델 구현
3. 저장소 구현
4. 서비스 구현
5. CLI 구현
6. 테스트 작성 (마지막)

### 실제 작업 로그에서의 차이

**Project-1의 TDD 사이클**

```
[Tool: Write] TaskTest.java           # 1. 테스트 먼저 작성
[Tool: Bash] ./gradlew test           # 2. RED 상태 확인
[Tool: Write] Task.java               # 3. 구현
[Tool: Bash] ./gradlew test           # 4. GREEN 상태 확인
```

스킬에 "테스트를 먼저 작성하라"고 명시했더니 실제로 모든 기능에서 테스트-구현-검증 순서를 따랐다.

**Project-2의 순차적 구현**

```
[Tool: Write] Priority.java           # 1. 구현 먼저
[Tool: Write] Status.java             # 2. 구현 계속
[Tool: Write] Task.java               # 3. 구현 계속
...
[Tool: Write] TaskTest.java           # N. 테스트는 마지막
```

테스트가 마지막에 작성되어 "구현을 검증하는 테스트"가 되었다.

### Todo 관리 방식 차이

**Project-1의 Todo 진행**

```json
{
  "content": "Task 1-1: Task 엔티티 테스트 작성 (RED)",
  "status": "in_progress"
}
```

- 각 Todo 항목이 TDD 단계(RED/GREEN/REFACTOR)를 포함
- 테스트와 구현이 분리된 작업 단위

**Project-2의 Todo 진행**

```json
{
  "content": "도메인 모델 구현 (Task, Priority, Status, Tag, History)",
  "status": "in_progress"
}
```

- 기능 단위로 Todo 구성
- 테스트는 별도 Todo로 분리 ("단위 테스트 작성")

### 커밋 메시지 비교

**Project-1**

```
feat(taskmanager): TDD 방식으로 JSON 기반 태스크 관리 시스템 구현

- TDD Red-Green-Refactor 사이클 준수
- 테스트: 단위(70%), 통합(20%), E2E(10%) 구조
- 커버리지: 80% 이상 달성

아키텍처:
- 계층 분리: Domain - Repository - Service - CLI
- SOLID 원칙 준수
- 의존성 주입 적용
```

커밋 메시지에 TDD 과정과 아키텍처 원칙이 명시되어 있다.

**Project-2**

```
feat: 파일 기반 태스크 관리 CLI 도구 구현

- 태스크 CRUD (생성, 조회, 수정, 삭제)
- 우선순위, 마감일, 태그 지원
- 태스크 필터링 및 검색

테스트:
- 단위 테스트 39개
- 통합 테스트 14개
```

기능 중심의 커밋 메시지로, 개발 방법론에 대한 언급이 없다.

### 테스트 실행 빈도

| 항목        | Project-1                       | Project-2  |
|-----------|---------------------------------|------------|
| 테스트 실행 횟수 | 기능당 2-3회 (RED, GREEN, REFACTOR) | 구현 완료 후 1회 |
| 실패 테스트 확인 | 매번 의도적으로 확인                     | 확인하지 않음    |
| 점진적 빌드    | 자주 실행                           | 마지막에 실행    |

### AI가 참조한 문서

**Project-1**

- tdd-reference/SKILL.md (TDD 원칙)
- architecture-reference/SKILL.md (SOLID, 패턴)
- tdd-check.md (체크리스트)

작업 시작 전에 스킬 문서를 읽고 원칙을 확인했다.

**Project-2**

- 참조 문서 없음
- 프롬프트의 요구사항만 참고

### 코드 품질 검증 방식

**Project-1**

```bash
./gradlew test                    # 테스트 실행
./gradlew jacocoTestReport        # 커버리지 확인
```

커버리지 80% 기준을 만족하는지 명시적으로 확인했다.

**Project-2**

```bash
./gradlew test                    # 테스트 실행만
```

커버리지 확인 과정이 없었다.

### 작업 과정에서 발견된 핵심 차이

| 항목         | Project-1    | Project-2 |
|------------|--------------|-----------|
| 개발 방법론     | TDD (테스트 주도) | 구현 주도     |
| 테스트 작성 시점  | 구현 전         | 구현 후      |
| 실패 테스트 확인  | 있음 (RED 단계)  | 없음        |
| 리팩토링 단계    | 명시적 수행       | 암묵적/생략    |
| 품질 기준 검증   | JaCoCo로 측정   | 미측정       |
| 아키텍처 원칙 참조 | SOLID 스킬 문서  | 없음        |
| 커밋 메시지     | 방법론 포함       | 기능 중심     |

스킬 설정은 단순히 결과물의 코드 품질뿐 아니라 **개발 과정 자체**에도 영향을 미쳤다. 스킬은 AI의 **작업 방식(How)**을 제어하고, 프롬프트는 **작업 목표(What)**를 제어한다.

---

## 스킬이 영향을 준 부분

| 스킬 조항         | Project-1 반영           | Project-2    | 트레이드오프                                |
|---------------|------------------------|--------------|---------------------------------------|
| DIP (추상화에 의존) | Repository 인터페이스 분리    | 구체 클래스 직접 사용 | 인터페이스 분리는 확장성이 높지만, 단순 앱에서는 과설계일 수 있음 |
| Builder 패턴    | Task에 적용               | Setter 방식    | Builder는 불변성을 보장하지만 코드량이 늘어남          |
| 방어적 프로그래밍     | Objects.requireNonNull | 별도 검증 없음     | 런타임 안정성 vs 코드 간결성                     |
| 테스트 용이성       | Mock 활용                | 실제 구현체 사용    | 격리된 단위 테스트 vs 통합 수준 신뢰성               |
| 커버리지 80%      | JaCoCo 설정 포함           | 설정 없음        | 정량적 품질 관리 vs 설정 단순화                   |

스킬 문서에 적힌 내용이 거의 그대로 코드에 반영됐다. 단순히 "좋은 코드 작성해줘"라고 하는 것보다 구체적인 기준을 제시하는 게 효과적이었다.

다만 모든 상황에서 Project-1 방식이 정답은 아니다. 프로젝트 규모, 팀 상황, 유지보수 기간에 따라 적절한 수준을 선택해야 한다.

---

## 스킬 설정이 효과적이었던 이유

### 1. 명시적 참조 지시

에이전트 설정에 "작업 시작 전에 다음 파일들을 읽어라"라고 적어둔 게 핵심이다. Claude가 매 작업마다 스킬 문서를 컨텍스트에 로드하면서 일관된 기준을 유지했다.

### 2. 구체적인 예시 코드

스킬 문서에 추상적인 원칙만 적은 게 아니라 실제 코드 예시를 포함했다:

```java
// Good: 추상화에 의존
public class Calculator {
    private final Logger logger;

    public Calculator(Logger logger) {
        this.logger = logger;
    }
}
```

이런 예시가 있으니 Claude가 비슷한 패턴을 적용할 수 있었다.

### 3. 체크리스트 형식

`tdd-check.md`에 단계별 체크리스트를 만들어둔 것도 효과가 있었다:

```markdown
## 2단계: 코드 구현 (그린)

- [ ] 테스트를 통과시키는 최소한만 구현했는가?
- [ ] null/빈값 검증을 추가했는가?
- [ ] 방어적 프로그래밍을 적용했는가?
```

---

## 한계와 주의점

스킬을 설정한다고 무조건 좋은 코드가 나오는 건 아니다.

### 스킬이 해결하지 못한 부분

- **도메인 지식**: 태스크 관리 앱의 비즈니스 로직은 스킬로 정의할 수 없다
- **트레이드오프 판단**: 상황에 따라 다른 설계가 필요한 경우 스킬이 오히려 방해될 수 있다
- **과도한 추상화**: 스킬을 너무 엄격하게 적용하면 간단한 코드도 복잡해질 수 있다

### Project-2가 나은 부분도 있었다

- Picocli 라이브러리 활용 (CLI 프레임워크)
- 변경 이력 추적 기능
- 테스트 가독성 (`@DisplayName`, AssertJ)

실무 도구 활용이나 사용자 관점 기능은 스킬보다 프롬프트에서 직접 요청하는 게 나을 수 있다.

---

## 정리

동일한 요구사항이라도 스킬 설정 여부에 따라 코드 품질이 달라졌다. 특히 아키텍처 원칙이나 테스트 전략처럼 "어떻게 만들 것인가"에 대한 기준은 스킬로 정의해두는 게 효과적이다.

다만 스킬은 "기준"을 제시할 뿐이고, "무엇을 만들 것인가"는 여전히 프롬프트로 명확하게 전달해야 한다. 둘을 적절히 조합하는 게 Claude Code를 잘 쓰는 방법인 것 같다.

### 스킬 vs 프롬프트: 언제 무엇을 쓸까

| 상황                    | 권장 방식 | 이유                   |
|-----------------------|-------|----------------------|
| 아키텍처 원칙 (SOLID, DI 등) | 스킬    | 모든 코드에 일관되게 적용해야 함   |
| 테스트 전략 (TDD, 커버리지)    | 스킬    | 개발 프로세스 자체를 제어함      |
| 코딩 컨벤션 (네이밍, 포맷)      | 스킬    | 반복적으로 적용되는 규칙        |
| 특정 기능 요구사항            | 프롬프트  | 일회성 지시사항             |
| 라이브러리/도구 선택           | 프롬프트  | 상황별 판단이 필요함          |
| 비즈니스 로직               | 프롬프트  | 도메인 지식은 스킬로 정의하기 어려움 |

### 적용 권장 시나리오

**스킬 설정 권장**

- 팀 프로젝트에서 일관된 코드 스타일 유지
- 장기 유지보수가 예상되는 프로젝트
- 테스트 커버리지 등 정량적 품질 기준이 있는 경우

**프롬프트만으로 충분**

- 빠른 프로토타이핑
- 일회성 스크립트나 도구
- 간단한 개인 프로젝트

---

## 실전 적용 가이드

이번 실험에서 효과가 있었던 스킬 작성 방식:

1. **원칙 + 예시 코드**를 함께 제공
2. **체크리스트** 형식으로 검증 가능하게 작성
3. **정량적 기준** 명시 (커버리지 80%, 메서드 20줄 이하 등)
4. 에이전트에 **"읽어라"** 지시를 명시적으로 포함

---

## 부록: 상세 비교

### 정량적 비교

| 항목       | Project-1 (스킬) | Project-2 (프롬프트) |
|----------|----------------|------------------|
| 테스트 코드량  | 1,252 LOC      | 753 LOC          |
| 테스트 파일 수 | 6개             | 6개               |
| E2E 테스트  | 있음 (CLI 전체 흐름) | 없음               |
| Mocking  | Mockito 사용     | 미사용              |
| 커버리지 측정  | JaCoCo 설정      | 없음               |

Project-1의 E2E 테스트는 CLI 명령어를 실행하고 JSON 파일 출력까지 검증하는 전체 흐름 테스트다. 스킬에 "테스트 피라미드: 단위(70%), 통합(20%), E2E(10%)"를 명시했기 때문에 E2E 테스트가 포함됐다.

### 의존성 차이

| 라이브러리   | Project-1 | Project-2 |
|---------|-----------|-----------|
| Mockito | 5.7.0     | 없음        |
| JaCoCo  | 0.8.11    | 없음        |
| Picocli | 없음        | 4.7.5     |
| AssertJ | 없음        | 3.24.2    |

### 종합 평가

| 평가 항목       | Project-1 (스킬) | Project-2 (프롬프트) |
|-------------|----------------|------------------|
| 아키텍처 설계     | 우수             | 보통               |
| 테스트 격리성     | 우수             | 보통               |
| CLI 사용성     | 보통             | 우수               |
| 테스트 가독성     | 보통             | 우수               |
| 외부 라이브러리 활용 | 보통             | 우수               |
| 방어적 프로그래밍   | 우수             | 보통               |

각 프로젝트가 강점을 보이는 영역이 다르다. 스킬은 아키텍처와 테스트 설계에, 프롬프트는 실용적 도구 활용에 강점이 있다.
