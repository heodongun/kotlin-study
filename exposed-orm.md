# Exposed ORM 학습 가이드

이 문서는 프로젝트에서 사용된 Exposed ORM의 개념과 사용법을 정리합니다.

## 1. Exposed 개요

### 개념
Exposed는 Kotlin용 경량 SQL 라이브러리입니다. 두 가지 API를 제공합니다:
- **DSL API**: SQL과 유사한 타입 안전한 쿼리
- **DAO API**: 객체 지향적 접근 (프로젝트에서는 미사용)

이 프로젝트는 DSL API를 사용합니다.

## 2. 테이블 정의

### 프로젝트 예시

```kotlin
// BoardEntity.kt
object BoardEntity : Table("boards") {
    val id = long("id").autoIncrement()
    val title = varchar("title", 255)
    val content = text("content")
    val author = varchar("author", 100)
    val createdAt = datetime("created_at")
    val updatedAt = datetime("updated_at").nullable()
    
    override val primaryKey = PrimaryKey(id)
}
```

### 컬럼 타입

| Kotlin 타입 | Exposed 함수 | SQL 타입 |
|-------------|--------------|----------|
| Long | `long()` | BIGINT |
| String | `varchar(length)` | VARCHAR |
| String | `text()` | TEXT |
| LocalDateTime | `datetime()` | DATETIME |
| Boolean | `bool()` | BOOLEAN |
| Int | `integer()` | INT |

### 컬럼 옵션

```kotlin
val id = long("id")
    .autoIncrement()           // AUTO_INCREMENT
    .primaryKey()              // PRIMARY KEY

val email = varchar("email", 255)
    .uniqueIndex()             // UNIQUE INDEX

val updatedAt = datetime("updated_at")
    .nullable()                // NULL 허용

val status = varchar("status", 50)
    .default("ACTIVE")         // DEFAULT 값
```

### 왜 이렇게 작성했나?
- **타입 안전성**: 컴파일 타임에 컬럼 타입 검증
- **명시적**: 테이블 구조가 코드로 명확히 표현
- **Kotlin DSL**: 선언적이고 읽기 쉬운 코드

## 3. CRUD 쿼리

### Create (INSERT)

```kotlin
// BoardRepositoryAdapter.kt
override suspend fun save(board: Board): Result<Board> = DatabaseConfig.dbQuery {
    try {
        val id = BoardEntity.insert {
            it[title] = board.title
            it[content] = board.content
            it[author] = board.author
            it[createdAt] = board.createdAt
            it[updatedAt] = board.updatedAt
        } get BoardEntity.id
        
        Result.Success(board.copy(id = BoardId(id)))
    } catch (e: Exception) {
        Result.Error("게시글 저장 실패", ErrorCode.DATABASE_ERROR)
    }
}
```

**생성된 SQL:**
```sql
INSERT INTO boards (title, content, author, created_at, updated_at)
VALUES (?, ?, ?, ?, ?)
```

### Read (SELECT)

#### 단건 조회
```kotlin
override suspend fun findById(id: BoardId): Result<Board?> = DatabaseConfig.dbQuery {
    try {
        val entity = BoardEntity
            .select { BoardEntity.id eq id.value }
            .singleOrNull()
        
        Result.Success(entity?.let { mapper.toDomain(it) })
    } catch (e: Exception) {
        Result.Error("게시글 조회 실패", ErrorCode.DATABASE_ERROR)
    }
}
```

**생성된 SQL:**
```sql
SELECT * FROM boards WHERE id = ?
```

#### 목록 조회
```kotlin
override suspend fun findAll(page: Int, size: Int): Result<List<Board>> = DatabaseConfig.dbQuery {
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

**생성된 SQL:**
```sql
SELECT * FROM boards
ORDER BY created_at DESC
LIMIT ? OFFSET ?
```

#### 카운트
```kotlin
override suspend fun count(): Result<Long> = DatabaseConfig.dbQuery {
    try {
        val count = BoardEntity.selectAll().count()
        Result.Success(count)
    } catch (e: Exception) {
        Result.Error("게시글 수 조회 실패", ErrorCode.DATABASE_ERROR)
    }
}
```

**생성된 SQL:**
```sql
SELECT COUNT(*) FROM boards
```

### Update (UPDATE)

```kotlin
override suspend fun update(board: Board): Result<Board> = DatabaseConfig.dbQuery {
    try {
        BoardEntity.update({ BoardEntity.id eq board.id.value }) {
            it[title] = board.title
            it[content] = board.content
            it[updatedAt] = board.updatedAt
        }
        Result.Success(board)
    } catch (e: Exception) {
        Result.Error("게시글 수정 실패", ErrorCode.DATABASE_ERROR)
    }
}
```

**생성된 SQL:**
```sql
UPDATE boards
SET title = ?, content = ?, updated_at = ?
WHERE id = ?
```

### Delete (DELETE)

```kotlin
override suspend fun deleteById(id: BoardId): Result<Unit> = DatabaseConfig.dbQuery {
    try {
        BoardEntity.deleteWhere { BoardEntity.id eq id.value }
        Result.Success(Unit)
    } catch (e: Exception) {
        Result.Error("게시글 삭제 실패", ErrorCode.DATABASE_ERROR)
    }
}
```

**생성된 SQL:**
```sql
DELETE FROM boards WHERE id = ?
```

## 4. 조건절 (WHERE)

### 기본 연산자

```kotlin
// 같음
BoardEntity.select { BoardEntity.id eq 1 }

// 같지 않음
BoardEntity.select { BoardEntity.id neq 1 }

// 크다/작다
BoardEntity.select { BoardEntity.id greater 10 }
BoardEntity.select { BoardEntity.id less 100 }

// LIKE
BoardEntity.select { BoardEntity.title like "%kotlin%" }

// IN
BoardEntity.select { BoardEntity.id inList listOf(1, 2, 3) }

// IS NULL
BoardEntity.select { BoardEntity.updatedAt.isNull() }

// IS NOT NULL
BoardEntity.select { BoardEntity.updatedAt.isNotNull() }
```

### 복합 조건

```kotlin
// AND
BoardEntity.select {
    (BoardEntity.author eq "John") and (BoardEntity.title like "%Kotlin%")
}

// OR
BoardEntity.select {
    (BoardEntity.author eq "John") or (BoardEntity.author eq "Jane")
}

// NOT
BoardEntity.select {
    not(BoardEntity.author eq "John")
}
```

## 5. 정렬 및 제한

### 정렬 (ORDER BY)

```kotlin
// 오름차순
BoardEntity
    .selectAll()
    .orderBy(BoardEntity.createdAt to SortOrder.ASC)

// 내림차순
BoardEntity
    .selectAll()
    .orderBy(BoardEntity.createdAt to SortOrder.DESC)

// 다중 정렬
BoardEntity
    .selectAll()
    .orderBy(
        BoardEntity.author to SortOrder.ASC,
        BoardEntity.createdAt to SortOrder.DESC
    )
```

### 제한 (LIMIT/OFFSET)

```kotlin
// LIMIT
BoardEntity
    .selectAll()
    .limit(10)

// LIMIT + OFFSET (페이지네이션)
BoardEntity
    .selectAll()
    .limit(10, offset = 20)
```

## 6. 트랜잭션 관리

### newSuspendedTransaction

```kotlin
// DatabaseConfig.kt
suspend fun <T> dbQuery(block: suspend () -> T): T =
    newSuspendedTransaction(Dispatchers.IO) {
        block()
    }
```

### 트랜잭션 특징

#### 1. 자동 커밋/롤백
```kotlin
newSuspendedTransaction {
    BoardEntity.insert { /* ... */ }
    // 성공 시 자동 커밋
    // 예외 발생 시 자동 롤백
}
```

#### 2. 중첩 트랜잭션
```kotlin
newSuspendedTransaction {
    BoardEntity.insert { /* ... */ }
    
    newSuspendedTransaction {
        // 중첩 트랜잭션
    }
}
```

#### 3. 명시적 롤백
```kotlin
newSuspendedTransaction {
    BoardEntity.insert { /* ... */ }
    rollback()  // 명시적 롤백
}
```

## 7. 코루틴 통합

### 프로젝트 패턴

```kotlin
// 1. dbQuery 헬퍼 함수 정의
suspend fun <T> dbQuery(block: suspend () -> T): T =
    newSuspendedTransaction(Dispatchers.IO) {
        block()
    }

// 2. Repository에서 사용
override suspend fun save(board: Board): Result<Board> = dbQuery {
    // Dispatchers.IO에서 실행
    // 트랜잭션 자동 관리
    BoardEntity.insert { /* ... */ }
}
```

### 왜 이렇게 작성했나?
- **비동기**: suspend 함수로 블로킹 없이 실행
- **성능**: I/O 스레드에서 데이터베이스 작업
- **안전성**: 트랜잭션 자동 관리

## 8. ResultRow 매핑

### Mapper 패턴

```kotlin
// BoardEntityMapper.kt
class BoardEntityMapper {
    fun toDomain(row: ResultRow): Board {
        return Board(
            id = BoardId(row[BoardEntity.id]),
            title = row[BoardEntity.title],
            content = row[BoardEntity.content],
            author = row[BoardEntity.author],
            createdAt = row[BoardEntity.createdAt],
            updatedAt = row[BoardEntity.updatedAt]
        )
    }
}

// 사용
val boards = BoardEntity
    .selectAll()
    .map { mapper.toDomain(it) }
```

### 왜 이렇게 작성했나?
- **분리**: 데이터베이스 엔티티와 도메인 모델 분리
- **재사용**: 매핑 로직을 한 곳에서 관리
- **타입 안전성**: 컴파일 타임에 컬럼 타입 검증

## 9. 데이터베이스 초기화

### 프로젝트 예시

```kotlin
// DatabaseConfig.kt
object DatabaseConfig {
    fun init(config: DatabaseSettings) {
        val hikariConfig = HikariConfig().apply {
            jdbcUrl = config.jdbcUrl
            driverClassName = "com.mysql.cj.jdbc.Driver"
            username = config.user
            password = config.password
            maximumPoolSize = 10
        }
        
        val dataSource = HikariDataSource(hikariConfig)
        Database.connect(dataSource)
        
        // 테이블 생성
        transaction {
            SchemaUtils.create(BoardEntity)
        }
    }
}
```

### SchemaUtils

```kotlin
// 테이블 생성
SchemaUtils.create(BoardEntity)

// 테이블 삭제
SchemaUtils.drop(BoardEntity)

// 테이블 존재 여부 확인 후 생성
SchemaUtils.createMissingTablesAndColumns(BoardEntity)
```

## 10. 실전 팁

### 1. 로깅 활성화

```kotlin
// logback.xml
<logger name="Exposed" level="DEBUG"/>
```

실행되는 SQL 쿼리를 확인할 수 있습니다.

### 2. 배치 삽입

```kotlin
BoardEntity.batchInsert(boards) { board ->
    this[BoardEntity.title] = board.title
    this[BoardEntity.content] = board.content
}
```

### 3. 조인 (프로젝트에서는 미사용)

```kotlin
(BoardEntity innerJoin CommentEntity)
    .select { BoardEntity.id eq 1 }
```

## 핵심 학습 포인트

1. **DSL API**: SQL과 유사한 타입 안전한 쿼리
2. **테이블 정의**: object로 테이블 구조 선언
3. **CRUD**: insert, select, update, delete 메서드
4. **트랜잭션**: newSuspendedTransaction으로 자동 관리
5. **코루틴 통합**: suspend 함수로 비동기 처리
6. **매핑**: ResultRow를 도메인 모델로 변환

Exposed는 Kotlin의 타입 안전성과 DSL을 활용하여 SQL을 더 안전하고 읽기 쉽게 작성할 수 있게 해줍니다.
