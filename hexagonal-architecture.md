# 헥사고날 아키텍처 학습 가이드

이 문서는 프로젝트에 적용된 헥사고날 아키텍처(포트와 어댑터 패턴)를 정리합니다.

## 1. 헥사고날 아키텍처 개요

### 개념
헥사고날 아키텍처는 비즈니스 로직을 외부 의존성(데이터베이스, UI, 외부 API 등)으로부터 완전히 분리하는 아키텍처 패턴입니다.

### 다른 이름
- **Ports and Adapters Pattern** (포트와 어댑터 패턴)
- **Clean Architecture**의 한 형태
- **Onion Architecture**와 유사

### 핵심 아이디어
> "비즈니스 로직은 프레임워크, 데이터베이스, UI에 의존하지 않아야 한다"

## 2. 구조 다이어그램

```
                    ┌─────────────────────────────────┐
                    │      Primary Adapters           │
                    │   (Driving/Input Side)          │
                    │                                 │
                    │  ┌──────────────────────┐      │
                    │  │   REST API (Ktor)    │      │
                    │  │  BoardController.kt  │      │
                    │  └──────────┬───────────┘      │
                    └─────────────┼──────────────────┘
                                  │
                    ┌─────────────▼──────────────────┐
                    │      Primary Ports             │
                    │    (Input Ports)               │
                    │                                │
                    │  ┌──────────────────────┐     │
                    │  │  BoardUseCase.kt     │     │
                    │  │  (Interface)         │     │
                    │  └──────────────────────┘     │
                    └─────────────┬──────────────────┘
                                  │
        ┌─────────────────────────▼─────────────────────────┐
        │              Domain Core (Hexagon)                │
        │           Business Logic & Rules                  │
        │                                                    │
        │  ┌──────────────────────────────────────────┐    │
        │  │      BoardService.kt                     │    │
        │  │   (UseCase Implementation)               │    │
        │  │                                          │    │
        │  │  - 비즈니스 규칙                          │    │
        │  │  - 도메인 로직                            │    │
        │  │  - 외부 의존성 없음                       │    │
        │  └──────────────────────────────────────────┘    │
        │                                                    │
        │  ┌──────────────────────────────────────────┐    │
        │  │      Domain Models                       │    │
        │  │   Board.kt, Result.kt                    │    │
        │  └──────────────────────────────────────────┘    │
        └─────────────────────────┬──────────────────────────┘
                                  │
                    ┌─────────────▼──────────────────┐
                    │    Secondary Ports             │
                    │    (Output Ports)              │
                    │                                │
                    │  ┌──────────────────────┐     │
                    │  │ BoardRepository.kt   │     │
                    │  │   (Interface)        │     │
                    │  └──────────────────────┘     │
                    └─────────────┬──────────────────┘
                                  │
                    ┌─────────────▼──────────────────┐
                    │    Secondary Adapters          │
                    │  (Driven/Output Side)          │
                    │                                │
                    │  ┌──────────────────────┐     │
                    │  │  MySQL Adapter       │     │
                    │  │  (Exposed ORM)       │     │
                    │  │ BoardRepositoryImpl  │     │
                    │  └──────────────────────┘     │
                    └────────────────────────────────┘
```

## 3. 핵심 구성 요소

### Domain Core (도메인 코어)

비즈니스 로직과 규칙이 있는 중심부입니다.

```kotlin
// domain/model/Board.kt
data class Board(
    val id: BoardId,
    val title: String,
    val content: String,
    val author: String,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime?
) {
    init {
        require(title.isNotBlank()) { "제목은 필수입니다" }
        require(content.isNotBlank()) { "내용은 필수입니다" }
    }
    
    fun update(title: String?, content: String?, updatedAt: LocalDateTime): Board {
        return copy(
            title = title ?: this.title,
            content = content ?: this.content,
            updatedAt = updatedAt
        )
    }
}
```

**특징:**
- 외부 프레임워크에 의존하지 않음
- 순수 Kotlin 코드
- 비즈니스 규칙 포함

### Primary Ports (입력 포트)

애플리케이션이 외부에 제공하는 인터페이스입니다.

```kotlin
// domain/port/input/CreateBoardUseCase.kt
interface CreateBoardUseCase {
    suspend fun execute(command: CreateBoardCommand): Result<Board>
}

data class CreateBoardCommand(
    val title: String,
    val content: String,
    val author: String
)
```

**특징:**
- 인터페이스로 정의
- 비즈니스 유스케이스 표현
- 구현체는 Domain Core에 위치

### Primary Adapters (입력 어댑터)

외부에서 애플리케이션을 호출하는 방법입니다.

```kotlin
// adapter/input/rest/BoardController.kt
class BoardController(
    private val createBoardUseCase: CreateBoardUseCase,
    // ...
) {
    fun Route.boardRoutes() {
        post {
            val request = call.receive<CreateBoardRequest>()
            val command = mapper.toCommand(request)
            
            when (val result = createBoardUseCase.execute(command)) {
                is Result.Success -> call.respond(HttpStatusCode.Created, result.data)
                is Result.Error -> call.respondError(result)
            }
        }
    }
}
```

**특징:**
- UseCase 인터페이스에만 의존
- 구현체를 알지 못함
- DTO를 도메인 모델로 변환

### Secondary Ports (출력 포트)

애플리케이션이 외부 시스템과 통신하기 위한 인터페이스입니다.

```kotlin
// domain/port/output/BoardRepository.kt
interface BoardRepository {
    suspend fun save(board: Board): Result<Board>
    suspend fun findById(id: BoardId): Result<Board?>
    suspend fun findAll(page: Int, size: Int): Result<List<Board>>
}
```

**특징:**
- 인터페이스로 정의
- 외부 시스템 추상화
- Domain Core에 위치

### Secondary Adapters (출력 어댑터)

외부 시스템과의 실제 통신 구현입니다.

```kotlin
// adapter/output/persistence/BoardRepositoryAdapter.kt
class BoardRepositoryAdapter : BoardRepository {
    override suspend fun save(board: Board): Result<Board> = DatabaseConfig.dbQuery {
        try {
            val id = BoardEntity.insert { /* ... */ }
            Result.Success(board.copy(id = BoardId(id)))
        } catch (e: Exception) {
            Result.Error("게시글 저장 실패", ErrorCode.DATABASE_ERROR)
        }
    }
}
```

**특징:**
- Repository 인터페이스 구현
- 실제 데이터베이스 접근
- 예외를 Result 타입으로 변환

## 4. 의존성 방향

### 핵심 규칙

```
adapter.input (REST)  ──────►  domain.port.input (UseCase)
                                      │
                                      ▼
                              domain.service (구현)
                                      │
                                      ▼
                              domain.port.output (Repository)
                                      ▲
                                      │
adapter.output (MySQL) ───────────────┘
```

**모든 의존성은 Domain을 향합니다!**

### 의존성 역전 원칙 (DIP)

```kotlin
// ❌ 잘못된 방법: Service가 구현체에 의존
class BoardService(
    private val repository: BoardRepositoryAdapter  // 구현체에 의존
)

// ✅ 올바른 방법: Service가 인터페이스에 의존
class BoardService(
    private val repository: BoardRepository  // 인터페이스에 의존
)
```

## 5. 프로젝트 디렉토리 구조

```
src/main/kotlin/com/example/board/
├── domain/                    # 도메인 코어
│   ├── model/                # 도메인 모델
│   ├── port/
│   │   ├── input/           # Primary Ports
│   │   └── output/          # Secondary Ports
│   └── service/             # UseCase 구현
│
├── adapter/                  # 어댑터
│   ├── input/               # Primary Adapters
│   │   └── rest/           # REST API
│   └── output/              # Secondary Adapters
│       └── persistence/     # 데이터베이스
│
└── common/                   # 공통 유틸리티
```

## 6. 실제 적용 사례

### 사례 1: 게시글 생성

#### 1. Controller (Primary Adapter)
```kotlin
post {
    val request = call.receive<CreateBoardRequest>()
    val command = mapper.toCommand(request)  // DTO → Command
    
    val result = createBoardUseCase.execute(command)  // UseCase 호출
    // ...
}
```

#### 2. UseCase (Primary Port)
```kotlin
interface CreateBoardUseCase {
    suspend fun execute(command: CreateBoardCommand): Result<Board>
}
```

#### 3. Service (Domain Core)
```kotlin
class BoardService(
    private val repository: BoardRepository  // Secondary Port
) : CreateBoardUseCase {
    override suspend fun execute(command: CreateBoardCommand): Result<Board> {
        val board = Board(/* ... */)
        return repository.save(board)  // Repository 호출
    }
}
```

#### 4. Repository (Secondary Port)
```kotlin
interface BoardRepository {
    suspend fun save(board: Board): Result<Board>
}
```

#### 5. RepositoryAdapter (Secondary Adapter)
```kotlin
class BoardRepositoryAdapter : BoardRepository {
    override suspend fun save(board: Board): Result<Board> {
        // 실제 데이터베이스 저장
    }
}
```

### 사례 2: 데이터베이스 교체

헥사고날 아키텍처의 장점은 외부 의존성을 쉽게 교체할 수 있다는 것입니다.

```kotlin
// MySQL → PostgreSQL 교체
class PostgreSQLBoardRepositoryAdapter : BoardRepository {
    override suspend fun save(board: Board): Result<Board> {
        // PostgreSQL 저장 로직
    }
}

// Koin 모듈에서만 변경
val appModule = module {
    single<BoardRepository> { PostgreSQLBoardRepositoryAdapter() }  // 이것만 변경!
}
```

**Domain Core는 전혀 수정하지 않아도 됩니다!**

## 7. 장점

### 1. 테스트 용이성

```kotlin
// Mock Repository로 Service 테스트
class MockBoardRepository : BoardRepository {
    override suspend fun save(board: Board): Result<Board> {
        return Result.Success(board.copy(id = BoardId(1)))
    }
}

val service = BoardService(MockBoardRepository())
```

### 2. 독립성
- 비즈니스 로직이 프레임워크에 독립적
- 데이터베이스 변경이 쉬움
- UI 변경이 쉬움 (REST → GraphQL)

### 3. 유지보수성
- 관심사가 명확히 분리됨
- 각 계층을 독립적으로 수정 가능
- 코드 이해가 쉬움

### 4. 확장성
- 새로운 어댑터 추가가 쉬움
- 기존 코드 수정 최소화

## 8. 단점 및 적용 시기

### 단점

1. **복잡성 증가**
   - 인터페이스와 구현체 분리로 파일 수 증가
   - 작은 프로젝트에는 과도할 수 있음

2. **초기 개발 속도**
   - 구조 설계에 시간 필요
   - 보일러플레이트 코드 증가

3. **학습 곡선**
   - 아키텍처 이해 필요
   - 팀원 교육 필요

### 적용 시기

#### ✅ 적용하면 좋은 경우
- 비즈니스 로직이 복잡한 경우
- 장기간 유지보수가 필요한 경우
- 외부 의존성이 자주 변경되는 경우
- 테스트가 중요한 경우

#### ❌ 적용하지 않아도 되는 경우
- 간단한 CRUD 애플리케이션
- 프로토타입이나 POC
- 단기 프로젝트
- 1인 개발

## 9. 핵심 학습 포인트

1. **의존성 역전**: 모든 의존성이 Domain을 향함
2. **포트와 어댑터**: 인터페이스로 외부 의존성 추상화
3. **도메인 중심**: 비즈니스 로직이 중심
4. **테스트 가능성**: Mock으로 쉽게 테스트
5. **유연성**: 외부 의존성 교체 용이

## 10. 다른 아키텍처와 비교

### Layered Architecture (전통적 3계층)
```
Controller → Service → Repository → Database
```
- 의존성이 한 방향으로만 흐름
- Repository가 데이터베이스에 의존

### Hexagonal Architecture
```
Controller → UseCase ← Service → Repository ← Adapter
```
- 의존성이 Domain을 향함
- Repository는 인터페이스, Adapter가 구현

## 결론

헥사고날 아키텍처는 비즈니스 로직을 보호하고, 외부 의존성을 쉽게 교체할 수 있게 해주는 강력한 패턴입니다. 프로젝트의 규모와 요구사항에 맞게 적절히 적용하는 것이 중요합니다.
