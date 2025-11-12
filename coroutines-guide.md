# Kotlin 코루틴 학습 가이드

이 문서는 프로젝트에서 사용된 Kotlin 코루틴의 개념과 패턴을 정리합니다.

## 1. suspend 함수 기본

### 개념
`suspend` 키워드는 함수가 일시 중단될 수 있음을 나타냅니다. 코루틴 컨텍스트에서만 호출 가능합니다.

### 프로젝트 예시

```kotlin
// BoardRepository.kt
interface BoardRepository {
    suspend fun save(board: Board): Result<Board>
    suspend fun findById(id: BoardId): Result<Board?>
    suspend fun findAll(page: Int, size: Int): Result<List<Board>>
}

// BoardService.kt
class BoardService(
    private val repository: BoardRepository
) : CreateBoardUseCase {
    override suspend fun execute(command: CreateBoardCommand): Result<Board> {
        // suspend 함수 호출
        return repository.save(board)
    }
}
```

### 주요 특징

#### 1. 순차적 코드 작성
```kotlin
override suspend fun execute(command: UpdateBoardCommand): Result<Board> {
    // 1. 먼저 조회
    val findResult = repository.findById(command.id)
    
    // 2. 결과 확인
    if (findResult is Result.Success) {
        val board = findResult.data ?: return Result.Error(...)
        
        // 3. 업데이트
        return repository.update(board)
    }
    
    return findResult
}
```

비동기 코드지만 동기 코드처럼 순차적으로 작성할 수 있습니다.

#### 2. 블로킹 없는 대기
```kotlin
suspend fun example() {
    val result1 = repository.findById(id)  // 블로킹 없이 대기
    val result2 = repository.save(board)   // 블로킹 없이 대기
}
```

### 왜 이렇게 작성했나?
- **가독성**: 콜백이나 Future 없이 순차적 코드
- **성능**: 스레드를 블로킹하지 않음
- **간결함**: async/await 없이 자연스러운 코드

## 2. Dispatchers

### 개념
Dispatcher는 코루틴이 실행될 스레드를 결정합니다.

### 주요 Dispatcher

#### Dispatchers.IO
I/O 작업(파일, 네트워크, 데이터베이스)에 최적화된 스레드 풀

```kotlin
// DatabaseConfig.kt
suspend fun <T> dbQuery(block: suspend () -> T): T =
    newSuspendedTransaction(Dispatchers.IO) {
        block()
    }
```

#### Dispatchers.Default
CPU 집약적 작업에 최적화된 스레드 풀

```kotlin
withContext(Dispatchers.Default) {
    // 복잡한 계산 작업
}
```

#### Dispatchers.Main
UI 스레드 (Android/Desktop 앱에서 사용)

### 프로젝트에서의 사용

```kotlin
// BoardRepositoryAdapter.kt
override suspend fun save(board: Board): Result<Board> = DatabaseConfig.dbQuery {
    // Dispatchers.IO에서 실행
    try {
        val id = BoardEntity.insert { /* ... */ }
        Result.Success(board.copy(id = BoardId(id)))
    } catch (e: Exception) {
        Result.Error("게시글 저장 실패", ErrorCode.DATABASE_ERROR)
    }
}
```

### 왜 이렇게 작성했나?
- **성능**: I/O 작업을 별도 스레드 풀에서 처리
- **안정성**: 메인 스레드를 블로킹하지 않음
- **최적화**: 작업 유형에 맞는 스레드 풀 사용

## 3. newSuspendedTransaction (Exposed와 코루틴 통합)

### 개념
Exposed ORM의 트랜잭션을 코루틴과 통합하여 사용합니다.

### 프로젝트 예시

```kotlin
// DatabaseConfig.kt
suspend fun <T> dbQuery(block: suspend () -> T): T =
    newSuspendedTransaction(Dispatchers.IO) {
        block()
    }

// 사용 예시
override suspend fun findAll(page: Int, size: Int): Result<List<Board>> = 
    DatabaseConfig.dbQuery {
        try {
            val boards = BoardEntity
                .selectAll()
                .orderBy(BoardEntity.createdAt to SortOrder.DESC)
                .limit(size, offset = ((page - 1) * size).toLong())
                .map { mapper.toDomain(it) }
            
            Result.Success(boards)
        } catch (e: Exception) {
            Result.Error("게시글 목록 조회 실패", ErrorCode.DATABASE_ERROR)
        }
    }
```

### 주요 특징

#### 1. 트랜잭션 자동 관리
```kotlin
newSuspendedTransaction {
    // 트랜잭션 시작
    BoardEntity.insert { /* ... */ }
    // 성공 시 자동 커밋, 실패 시 자동 롤백
}
```

#### 2. 코루틴 컨텍스트 전환
```kotlin
newSuspendedTransaction(Dispatchers.IO) {
    // I/O 스레드에서 실행
}
```

### 왜 이렇게 작성했나?
- **트랜잭션 안전성**: 자동 커밋/롤백
- **코루틴 통합**: suspend 함수로 자연스럽게 사용
- **성능**: I/O 스레드에서 데이터베이스 작업 수행

## 4. 구조화된 동시성 (Structured Concurrency)

### 개념
코루틴은 항상 특정 스코프 내에서 실행되며, 스코프가 취소되면 모든 자식 코루틴도 취소됩니다.

### Ktor에서의 코루틴 스코프

```kotlin
// Ktor 라우트 핸들러는 자동으로 코루틴 스코프를 제공
get("/{id}") {
    // 이 블록은 코루틴 컨텍스트에서 실행
    val result = getBoardUseCase.execute(id)
    call.respond(result)
}
```

### 병렬 실행 예시 (프로젝트에서는 미사용)

```kotlin
suspend fun example() {
    coroutineScope {
        val deferred1 = async { repository.findById(id1) }
        val deferred2 = async { repository.findById(id2) }
        
        val result1 = deferred1.await()
        val result2 = deferred2.await()
    }
}
```

### 왜 이렇게 작성했나?
- **안전성**: 스코프가 취소되면 모든 작업 취소
- **리소스 관리**: 메모리 누수 방지
- **예측 가능성**: 코루틴 생명주기 명확

## 5. 예외 처리

### 프로젝트 예시

```kotlin
override suspend fun save(board: Board): Result<Board> = DatabaseConfig.dbQuery {
    try {
        val id = BoardEntity.insert { /* ... */ }
        Result.Success(board.copy(id = BoardId(id)))
    } catch (e: Exception) {
        Result.Error("게시글 저장 실패: ${e.message}", ErrorCode.DATABASE_ERROR)
    }
}
```

### 패턴

#### 1. try-catch로 예외 처리
```kotlin
suspend fun example(): Result<Board> {
    return try {
        val board = repository.save(board)
        Result.Success(board)
    } catch (e: Exception) {
        Result.Error(e.message, ErrorCode.DATABASE_ERROR)
    }
}
```

#### 2. Result 타입으로 에러 전파
```kotlin
// 예외를 던지지 않고 Result.Error 반환
when (val result = repository.findById(id)) {
    is Result.Success -> // 성공 처리
    is Result.Error -> return result  // 에러 전파
}
```

### 왜 이렇게 작성했나?
- **명시적 에러 처리**: Result 타입으로 실패 가능성 표현
- **안전성**: 예외가 상위로 전파되지 않음
- **타입 안전성**: when 표현식으로 모든 케이스 처리

## 6. 코루틴 vs 스레드

### 차이점

| 특징 | 코루틴 | 스레드 |
|------|--------|--------|
| 생성 비용 | 매우 낮음 | 높음 |
| 메모리 | 수백 바이트 | 수 MB |
| 컨텍스트 스위칭 | 빠름 | 느림 |
| 동시 실행 수 | 수십만 개 가능 | 수천 개 제한 |

### 프로젝트에서의 이점

```kotlin
// 수천 개의 동시 요청 처리 가능
routing {
    get("/api/boards") {
        // 각 요청이 코루틴으로 처리
        // 블로킹 없이 데이터베이스 조회
        val result = listBoardsUseCase.execute(query)
        call.respond(result)
    }
}
```

## 7. 실전 패턴

### 패턴 1: Repository 계층

```kotlin
interface BoardRepository {
    suspend fun save(board: Board): Result<Board>
}

class BoardRepositoryAdapter : BoardRepository {
    override suspend fun save(board: Board): Result<Board> = 
        DatabaseConfig.dbQuery {
            // Dispatchers.IO에서 실행
            // 트랜잭션 자동 관리
        }
}
```

### 패턴 2: Service 계층

```kotlin
class BoardService(
    private val repository: BoardRepository
) : CreateBoardUseCase {
    override suspend fun execute(command: CreateBoardCommand): Result<Board> {
        // 순차적으로 작성
        // suspend 함수 호출
        return repository.save(board)
    }
}
```

### 패턴 3: Controller 계층

```kotlin
post {
    val request = call.receive<CreateBoardRequest>()
    // Ktor가 자동으로 코루틴 컨텍스트 제공
    val result = createBoardUseCase.execute(command)
    call.respond(result)
}
```

## 핵심 학습 포인트

1. **suspend 함수**: 비동기 코드를 동기 코드처럼 작성
2. **Dispatchers**: 작업 유형에 맞는 스레드 풀 선택
3. **newSuspendedTransaction**: Exposed와 코루틴 통합
4. **구조화된 동시성**: 안전한 코루틴 생명주기 관리
5. **예외 처리**: Result 타입으로 명시적 에러 처리
6. **성능**: 적은 리소스로 많은 동시 작업 처리

코루틴은 Kotlin의 가장 강력한 기능 중 하나로, 비동기 프로그래밍을 간결하고 안전하게 만들어줍니다.
