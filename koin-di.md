# Koin 의존성 주입 학습 가이드

이 문서는 프로젝트에서 사용된 Koin 의존성 주입 프레임워크를 정리합니다.

## 1. Koin 개요

### 개념
Koin은 Kotlin 전용 경량 의존성 주입 프레임워크입니다. 리플렉션을 사용하지 않고 Kotlin DSL로 의존성을 정의합니다.

### 특징
- **경량**: 리플렉션 없음, 빠른 시작 시간
- **Kotlin DSL**: 선언적이고 읽기 쉬운 코드
- **간단함**: 어노테이션 프로세싱 불필요
- **Ktor 통합**: Ktor와 완벽하게 통합

### 다른 DI 프레임워크와 비교

| 특징 | Koin | Dagger | Spring |
|------|------|--------|--------|
| 리플렉션 | ❌ | ❌ | ✅ |
| 컴파일 타임 검증 | ❌ | ✅ | ❌ |
| 학습 곡선 | 낮음 | 높음 | 중간 |
| 성능 | 빠름 | 매우 빠름 | 느림 |
| Kotlin 친화성 | ✅ | ⚠️ | ⚠️ |

## 2. 기본 문법

### module { } 블록

```kotlin
// AppModule.kt
val appModule = module {
    // 의존성 정의
}
```

### single (싱글톤)

```kotlin
val appModule = module {
    // 앱 전체에서 하나의 인스턴스만 생성
    single<BoardRepository> { BoardRepositoryAdapter() }
    single<TimeProvider> { SystemTimeProvider() }
}
```

**특징:**
- 앱 전체에서 하나의 인스턴스
- 처음 요청 시 생성
- 이후 요청 시 동일한 인스턴스 반환

### factory (팩토리)

```kotlin
val appModule = module {
    // 요청할 때마다 새 인스턴스 생성
    factory { BoardDtoMapper() }
}
```

**특징:**
- 요청할 때마다 새 인스턴스 생성
- 상태를 가지지 않는 객체에 적합

### scoped (스코프)

```kotlin
val appModule = module {
    // 특정 스코프 내에서만 유지
    scope<BoardController> {
        scoped { BoardService(get(), get()) }
    }
}
```

**특징:**
- 특정 스코프 내에서만 인스턴스 유지
- 스코프 종료 시 인스턴스 제거

## 3. 프로젝트 적용 사례

### 전체 모듈 정의

```kotlin
// common/di/AppModule.kt
val appModule = module {
    // Secondary Adapters (출력 어댑터)
    single<BoardRepository> { BoardRepositoryAdapter() }
    single<TimeProvider> { SystemTimeProvider() }
    
    // Domain Services (유스케이스 구현)
    single { BoardService(get(), get()) }
    single<CreateBoardUseCase> { get<BoardService>() }
    single<GetBoardUseCase> { get<BoardService>() }
    single<ListBoardsUseCase> { get<BoardService>() }
    single<UpdateBoardUseCase> { get<BoardService>() }
    single<DeleteBoardUseCase> { get<BoardService>() }
    
    // Mappers
    single { BoardDtoMapper() }
    
    // Primary Adapters (입력 어댑터)
    single {
        BoardController(
            createBoardUseCase = get(),
            getBoardUseCase = get(),
            listBoardsUseCase = get(),
            updateBoardUseCase = get(),
            deleteBoardUseCase = get(),
            mapper = get()
        )
    }
}
```

### 의존성 해결

#### get() 함수
```kotlin
single { BoardService(get(), get()) }
//                    ↑     ↑
//                    │     └─ TimeProvider 자동 주입
//                    └─ BoardRepository 자동 주입
```

Koin이 타입을 보고 자동으로 의존성을 찾아 주입합니다.

#### 명시적 타입 지정
```kotlin
single<CreateBoardUseCase> { get<BoardService>() }
//     ↑                      ↑
//     │                      └─ BoardService 인스턴스 가져오기
//     └─ CreateBoardUseCase 타입으로 등록
```

## 4. Ktor와 통합

### Koin 초기화

```kotlin
// Application.kt
fun Application.module() {
    // Koin 플러그인 설치
    install(Koin) {
        slf4jLogger()  // SLF4J 로거 사용
        modules(appModule)  // 모듈 등록
    }
    
    // ...
}
```

### 의존성 주입

#### by inject() 위임 프로퍼티
```kotlin
routing {
    val boardController by inject<BoardController>()
    boardController.apply { boardRoutes() }
}
```

**특징:**
- 지연 주입 (lazy injection)
- 처음 접근 시 인스턴스 생성
- 위임 프로퍼티 사용

#### get() 함수
```kotlin
routing {
    val boardController = get<BoardController>()
    boardController.apply { boardRoutes() }
}
```

**특징:**
- 즉시 주입
- 호출 시점에 인스턴스 가져오기

## 5. 생성자 주입

### 프로젝트 예시

```kotlin
// BoardService.kt
class BoardService(
    private val repository: BoardRepository,
    private val timeProvider: TimeProvider
) : CreateBoardUseCase, GetBoardUseCase, /* ... */ {
    // Koin이 자동으로 의존성 주입
}

// Koin 모듈
val appModule = module {
    single { BoardService(get(), get()) }
    //                    ↑     ↑
    //                    │     └─ TimeProvider 주입
    //                    └─ BoardRepository 주입
}
```

### 왜 생성자 주입을 사용하나?
- **명시적**: 의존성이 명확히 보임
- **테스트 용이**: Mock 객체 주입 쉬움
- **불변성**: `val`로 선언 가능

## 6. named() 한정자

### 개념
같은 타입의 다른 구현체를 구분할 때 사용합니다.

### 예시

```kotlin
val appModule = module {
    // 프로덕션용 TimeProvider
    single(named("production")) { SystemTimeProvider() }
    
    // 테스트용 TimeProvider
    single(named("test")) { FixedTimeProvider() }
    
    // 사용
    single {
        BoardService(
            repository = get(),
            timeProvider = get(named("production"))
        )
    }
}
```

### 사용 시나리오
- 환경별 설정 (dev, staging, production)
- 같은 인터페이스의 다른 구현체
- 테스트용 Mock 객체

## 7. KoinComponent vs 생성자 주입

### KoinComponent (프로퍼티 주입)

```kotlin
class BoardController : KoinComponent {
    private val createBoardUseCase: CreateBoardUseCase by inject()
    private val getBoardUseCase: GetBoardUseCase by inject()
    
    fun Route.boardRoutes() {
        // 사용
    }
}
```

**장점:**
- 의존성이 많을 때 편리
- 선택적 의존성 가능

**단점:**
- 의존성이 숨겨짐
- 테스트 시 Mock 주입 어려움

### 생성자 주입 (권장)

```kotlin
class BoardController(
    private val createBoardUseCase: CreateBoardUseCase,
    private val getBoardUseCase: GetBoardUseCase,
    // ...
) {
    fun Route.boardRoutes() {
        // 사용
    }
}
```

**장점:**
- 의존성이 명확
- 테스트 용이
- 불변성 보장

**단점:**
- 의존성이 많으면 생성자가 길어짐

## 8. 테스트에서 Koin 활용

### 테스트 모듈

```kotlin
val testModule = module {
    single<BoardRepository> { MockBoardRepository() }
    single<TimeProvider> { FixedTimeProvider() }
    single { BoardService(get(), get()) }
}

class BoardServiceTest : KoinTest {
    @Before
    fun setup() {
        startKoin {
            modules(testModule)
        }
    }
    
    @After
    fun tearDown() {
        stopKoin()
    }
    
    @Test
    fun `게시글 생성 테스트`() {
        val service: BoardService by inject()
        
        // 테스트 코드
        val command = CreateBoardCommand("제목", "내용", "작성자")
        val result = runBlocking { service.execute(command) }
        
        assertTrue(result is Result.Success)
    }
}
```

### Mock 객체

```kotlin
class MockBoardRepository : BoardRepository {
    override suspend fun save(board: Board): Result<Board> {
        return Result.Success(board.copy(id = BoardId(1)))
    }
    
    override suspend fun findById(id: BoardId): Result<Board?> {
        return Result.Success(
            Board(
                id = id,
                title = "테스트 제목",
                content = "테스트 내용",
                author = "테스트 작성자",
                createdAt = LocalDateTime.now(),
                updatedAt = null
            )
        )
    }
}
```

## 9. 실전 패턴

### 패턴 1: 인터페이스와 구현체 분리

```kotlin
val appModule = module {
    // 인터페이스 타입으로 등록
    single<BoardRepository> { BoardRepositoryAdapter() }
    
    // 사용하는 쪽은 인터페이스만 알면 됨
    single { BoardService(get(), get()) }
}
```

### 패턴 2: 여러 인터페이스 구현

```kotlin
val appModule = module {
    // BoardService는 여러 UseCase 인터페이스를 구현
    single { BoardService(get(), get()) }
    
    // 각 UseCase 타입으로 등록
    single<CreateBoardUseCase> { get<BoardService>() }
    single<GetBoardUseCase> { get<BoardService>() }
    single<ListBoardsUseCase> { get<BoardService>() }
}
```

### 패턴 3: 모듈 분리

```kotlin
// 도메인 모듈
val domainModule = module {
    single { BoardService(get(), get()) }
}

// 인프라 모듈
val infrastructureModule = module {
    single<BoardRepository> { BoardRepositoryAdapter() }
    single<TimeProvider> { SystemTimeProvider() }
}

// 애플리케이션 모듈
val applicationModule = module {
    single { BoardController(get(), get(), get(), get(), get(), get()) }
}

// 전체 모듈
val appModules = listOf(domainModule, infrastructureModule, applicationModule)

// 사용
install(Koin) {
    modules(appModules)
}
```

## 10. 주의사항

### 1. 순환 의존성

```kotlin
// ❌ 잘못된 예: 순환 의존성
class A(val b: B)
class B(val a: A)

val module = module {
    single { A(get()) }
    single { B(get()) }  // 에러!
}
```

**해결 방법:**
- 의존성 방향 재설계
- 인터페이스 도입

### 2. 타입 모호성

```kotlin
// ❌ 잘못된 예: 같은 타입 여러 개
val module = module {
    single { "config1" }
    single { "config2" }  // 어떤 것을 주입?
}

// ✅ 올바른 예: named() 사용
val module = module {
    single(named("config1")) { "config1" }
    single(named("config2")) { "config2" }
}
```

### 3. 메모리 누수

```kotlin
// ❌ 잘못된 예: 무거운 객체를 single로
val module = module {
    single { HeavyObject() }  // 앱 종료까지 메모리 유지
}

// ✅ 올바른 예: factory 사용
val module = module {
    factory { HeavyObject() }  // 사용 후 GC 가능
}
```

## 11. 핵심 학습 포인트

1. **DSL 문법**: module { }, single, factory, scoped
2. **의존성 해결**: get() 함수로 자동 주입
3. **Ktor 통합**: install(Koin), by inject()
4. **생성자 주입**: 명시적이고 테스트 용이
5. **named() 한정자**: 같은 타입 구분
6. **테스트**: Mock 객체로 쉬운 테스트

## 12. Koin vs 다른 DI 프레임워크

### Koin의 장점
- Kotlin DSL로 간결한 코드
- 리플렉션 없어 빠른 시작
- 학습 곡선이 낮음
- Ktor와 완벽한 통합

### Koin의 단점
- 컴파일 타임 검증 없음 (런타임 에러 가능)
- 대규모 프로젝트에서는 Dagger가 더 나을 수 있음

## 결론

Koin은 Kotlin의 특징을 최대한 활용한 경량 DI 프레임워크입니다. 간단한 DSL로 의존성을 정의하고, Ktor와 완벽하게 통합되어 사용하기 쉽습니다. 헥사고날 아키텍처와 함께 사용하면 테스트 가능하고 유지보수하기 쉬운 코드를 작성할 수 있습니다.
