# Kotlin 관용적 패턴 학습 가이드

이 문서는 프로젝트에서 사용된 Kotlin의 관용적 패턴들을 정리합니다.

## 1. Data Class

### 개념
Data class는 데이터를 보관하는 것이 주 목적인 클래스입니다. 컴파일러가 자동으로 유용한 메서드들을 생성해줍니다.

### 자동 생성되는 메서드
- `equals()` / `hashCode()`
- `toString()`
- `copy()`
- `componentN()` (구조 분해)

### 프로젝트 예시

```kotlin
// Board.kt
data class Board(
    val id: BoardId,
    val title: String,
    val content: String,
    val author: String,
    val createdAt: LocalDateTime,
    val updatedAt: LocalDateTime?
)

// copy() 메서드 활용
fun update(title: String?, content: String?, updatedAt: LocalDateTime): Board {
    return copy(
        title = title ?: this.title,
        content = content ?: this.content,
        updatedAt = updatedAt
    )
}
```

### 왜 이렇게 작성했나?
- **불변성**: `val`로 선언하여 데이터 안전성 보장
- **copy()**: 불변 객체를 수정할 때 새 인스턴스 생성
- **Elvis 연산자 (`?:`)**: null인 경우 기존 값 유지

## 2. Sealed Class

### 개념
Sealed class는 제한된 클래스 계층 구조를 표현합니다. 모든 하위 클래스가 컴파일 타임에 알려져 있어 타입 안전성을 보장합니다.

### 프로젝트 예시

```kotlin
// Result.kt
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val code: ErrorCode) : Result<Nothing>()
}

// 사용 예시
when (val result = createBoardUseCase.execute(command)) {
    is Result.Success -> call.respond(HttpStatusCode.Created, result.data)
    is Result.Error -> call.respondError(result)
}
```

### 왜 이렇게 작성했나?
- **타입 안전성**: when 표현식에서 모든 케이스를 처리하도록 강제
- **명시적 에러 처리**: 예외 대신 Result 타입으로 성공/실패 표현
- **공변성 (`out`)**: Success는 T를 반환만 하므로 공변성 적용

## 3. Value Class (Inline Class)

### 개념
Value class는 래퍼 클래스의 오버헤드 없이 타입 안전성을 제공합니다. 컴파일 시 기본 타입으로 인라인됩니다.

### 프로젝트 예시

```kotlin
// BoardId.kt
@JvmInline
value class BoardId(val value: Long) {
    init {
        require(value >= 0) { "Board ID는 0 이상이어야 합니다" }
    }
}
```

### 왜 이렇게 작성했나?
- **타입 안전성**: Long 대신 BoardId 사용으로 실수 방지
- **성능**: 런타임에 Long으로 인라인되어 오버헤드 없음
- **유효성 검증**: init 블록에서 값 검증

## 4. 확장 함수 (Extension Functions)

### 개념
기존 클래스를 수정하지 않고 새로운 함수를 추가할 수 있습니다.

### 프로젝트 예시

```kotlin
// KtorExtensions.kt
suspend fun ApplicationCall.respondError(error: Result.Error) {
    val status = when (error.code) {
        ErrorCode.NOT_FOUND -> HttpStatusCode.NotFound
        ErrorCode.VALIDATION_ERROR -> HttpStatusCode.BadRequest
        ErrorCode.DATABASE_ERROR -> HttpStatusCode.InternalServerError
        ErrorCode.INTERNAL_ERROR -> HttpStatusCode.InternalServerError
    }
    respond(status, ErrorResponse(error.message, error.code.name))
}

// 사용
call.respondError(result)
```

### 왜 이렇게 작성했나?
- **가독성**: `call.respondError()`가 더 자연스러움
- **재사용성**: 여러 곳에서 동일한 에러 응답 로직 사용
- **관심사 분리**: 에러 응답 로직을 별도 파일로 분리

## 5. Scope 함수

### let

```kotlin
// BoardController.kt
val id = call.parameters["id"]?.toLongOrNull()?.let { BoardId(it) }
    ?: return@get call.respond(HttpStatusCode.BadRequest, "Invalid ID")
```

**용도**: null이 아닐 때만 변환 수행

### apply

```kotlin
val hikariConfig = HikariConfig().apply {
    jdbcUrl = config.jdbcUrl
    driverClassName = "com.mysql.cj.jdbc.Driver"
    username = config.user
    password = config.password
}
```

**용도**: 객체 초기화 시 여러 속성 설정

## 6. when 표현식

### 개념
Java의 switch보다 강력한 조건 분기 표현식입니다.

### 프로젝트 예시

```kotlin
// BoardService.kt
return when (val result = repository.findById(id)) {
    is Result.Success -> {
        result.data?.let { Result.Success(it) }
            ?: Result.Error("게시글을 찾을 수 없습니다", ErrorCode.NOT_FOUND)
    }
    is Result.Error -> result
}
```

### 왜 이렇게 작성했나?
- **스마트 캐스트**: `is` 체크 후 자동 타입 변환
- **표현식**: 값을 반환하므로 변수에 할당 가능
- **완전성 검사**: sealed class와 함께 사용 시 모든 케이스 처리 강제

## 7. Inline 함수

### 개념
함수 호출 오버헤드를 제거하고, 람다를 인라인하여 성능을 향상시킵니다.

### 프로젝트 예시

```kotlin
// Result.kt
inline fun <R> map(transform: (T) -> R): Result<R> = when (this) {
    is Success -> Success(transform(data))
    is Error -> this
}

inline fun onSuccess(action: (T) -> Unit): Result<T> {
    if (this is Success) action(data)
    return this
}
```

### 왜 이렇게 작성했나?
- **성능**: 람다 객체 생성 오버헤드 제거
- **non-local return**: 람다 내에서 외부 함수 return 가능

## 8. DSL 빌더 패턴

### 개념
Kotlin의 람다와 확장 함수를 활용하여 선언적 API를 만듭니다.

### 프로젝트 예시

```kotlin
// BoardController.kt
fun Route.boardRoutes() {
    route("/api/boards") {
        post {
            // 게시글 생성
        }
        
        get {
            // 목록 조회
        }
        
        get("/{id}") {
            // 상세 조회
        }
    }
}
```

### 왜 이렇게 작성했나?
- **가독성**: 라우트 구조가 명확하게 보임
- **타입 안전성**: 컴파일 타임에 오류 검출
- **Kotlin스러움**: Ktor의 DSL 스타일 활용

## 9. Nullable 타입과 안전한 호출

### 프로젝트 예시

```kotlin
// Board.kt
data class Board(
    val updatedAt: LocalDateTime?  // nullable
)

// 안전한 호출
board.updatedAt?.toString()  // null이면 null 반환

// Elvis 연산자
val title = command.title ?: this.title  // null이면 기본값 사용
```

### 왜 이렇게 작성했나?
- **null 안전성**: NullPointerException 방지
- **명시적**: nullable 여부가 타입에 표현됨
- **간결함**: `?.`와 `?:` 연산자로 간단하게 처리

## 10. 구조 분해 (Destructuring)

### 프로젝트 예시

```kotlin
// Koin 의존성 주입
val boardController by inject<BoardController>()

// data class 구조 분해 (사용 가능하지만 프로젝트에서는 미사용)
val (id, title, content) = board
```

## 핵심 학습 포인트

1. **불변성 우선**: `val` 사용, data class의 copy()
2. **null 안전성**: nullable 타입, 안전한 호출 연산자
3. **타입 안전성**: sealed class, value class
4. **표현력**: when 표현식, 확장 함수
5. **성능**: inline 함수, value class
6. **가독성**: DSL, scope 함수

이러한 패턴들을 조합하여 안전하고 읽기 쉬운 Kotlin 코드를 작성할 수 있습니다.
