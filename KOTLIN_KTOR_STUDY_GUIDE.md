# Kotlin + Ktor 프로젝트 학습 가이드

이 문서는 Coding Platform Backend 프로젝트를 통해 Kotlin, Ktor, 코루틴의 실전 사용법을 학습하기 위한 가이드입니다.

## 목차
1. [프로젝트 개요](#1-프로젝트-개요)
2. [Ktor 프레임워크 이해하기](#2-ktor-프레임워크-이해하기)
3. [Kotlin 코루틴 활용](#3-kotlin-코루틴-활용)
4. [Kotlin 고유 기능 활용](#4-kotlin-고유-기능-활용)
5. [데이터베이스 연동 (Exposed)](#5-데이터베이스-연동-exposed)
6. [의존성 주입 패턴](#6-의존성-주입-패턴)
7. [실전 예제 분석](#7-실전-예제-분석)

---

## 1. 프로젝트 개요

### 1.1 기술 스택
- **언어**: Kotlin 1.9.20
- **웹 프레임워크**: Ktor 2.3.6
- **데이터베이스**: PostgreSQL + Exposed ORM
- **캐시**: Redis (Lettuce)
- **인증**: JWT
- **비동기**: Kotlin Coroutines
- **컨테이너**: Docker (코드 실행 샌드박스)

### 1.2 프로젝트 구조
```
src/main/kotlin/com/codingplatform/
├── Application.kt              # 애플리케이션 진입점
├── plugins/                    # Ktor 플러그인 설정
│   ├── Routing.kt             # 라우팅 설정
│   ├── Security.kt            # JWT, CORS 설정
│   ├── Database.kt            # DB 연결 설정
│   ├── Serialization.kt       # JSON 직렬화
│   └── Monitoring.kt          # 메트릭 수집
├── routes/                     # HTTP 엔드포인트
├── services/                   # 비즈니스 로직
├── models/                     # 데이터 모델
├── database/                   # DB 스키마
└── executor/                   # Docker 실행 엔진
```


---

## 2. Ktor 프레임워크 이해하기

### 2.1 Ktor란?
Ktor는 JetBrains가 만든 **비동기 웹 프레임워크**로, Kotlin 코루틴을 기반으로 동작합니다.
- Spring Boot보다 가볍고 빠름
- 코루틴 네이티브 지원
- DSL 스타일의 직관적인 API

### 2.2 애플리케이션 시작하기

**파일**: `src/main/kotlin/com/codingplatform/Application.kt`

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080, host = "0.0.0.0") {
        module()  // 애플리케이션 설정 함수 호출
    }.start(wait = true)
}

fun Application.module() {
    val services = ServiceRegistry(environment)
    
    // 플러그인 설치 순서가 중요!
    configureMonitoring(services.metricsRegistry)
    configureSerialization()
    configureStatusPages()
    configureSecurity(services.jwtConfig)
    configureDatabase(services.databaseFactory)
    configureRouting(services)
    
    // 애플리케이션 종료 시 리소스 정리
    environment.monitor.subscribe(ApplicationStopping) {
        runBlocking { services.shutdown() }
    }
}
```

**핵심 개념**:
- `embeddedServer`: 내장 서버 생성 (Netty 엔진 사용)
- `Application.module()`: 확장 함수로 애플리케이션 설정
- `environment.monitor`: 애플리케이션 생명주기 이벤트 구독

### 2.3 라우팅 (Routing)

**파일**: `src/main/kotlin/com/codingplatform/plugins/Routing.kt`

```kotlin
fun Application.configureRouting(services: ServiceRegistry) {
    routing {
        // OpenAPI/Swagger 문서
        openAPI(path = "openapi", swaggerFile = "openapi/documentation.yaml")
        swaggerUI(path = "swagger", swaggerFile = "openapi/documentation.yaml")
        
        // 각 도메인별 라우트 설정
        configureAuthRoutes(services.authService, services.emailVerificationService)
        configureProblemRoutes(services.problemService)
        // ...
    }
}
```

**라우트 정의 예시**: `src/main/kotlin/com/codingplatform/routes/AuthRoutes.kt`

```kotlin
fun Route.configureAuthRoutes(
    authService: AuthService,
    emailVerificationService: EmailVerificationService
) {
    // POST /api/auth/register/code
    post("/api/auth/register/code") {
        val request = call.receive<VerificationCodeRequest>()
        authService.requestVerificationCode(request.email)
        call.respond(HttpStatusCode.Accepted, mapOf("message" to "인증 코드가 발송되었습니다."))
    }

    // POST /api/auth/register
    post("/api/auth/register") {
        val request = call.receive<RegisterRequest>()
        val (user, token) = authService.register(
            email = request.email,
            password = request.password,
            name = request.name,
            verificationCode = request.verificationCode,
            role = UserRole.USER
        )
        call.respond(HttpStatusCode.Created, AuthResponse(token = token, user = user))
    }
}
```

**핵심 개념**:
- `Route.확장함수`: 라우트를 모듈화
- `call.receive<T>()`: 요청 본문을 역직렬화
- `call.respond()`: 응답 반환 (자동 직렬화)
- HTTP 메서드: `get`, `post`, `put`, `delete`


### 4.2 데이터 클래스 (Data Classes)

```kotlin
// 자동으로 equals, hashCode, toString, copy 생성
data class User(
    val id: UUID,
    val email: String,
    val name: String,
    val role: UserRole,
    val createdAt: Instant,
    val updatedAt: Instant,
    val lastLoginAt: Instant?
)

// copy로 불변 객체 수정
val updatedUser = user.copy(name = "새 이름")
```

### 4.3 Null 안전성

Kotlin은 컴파일 타임에 null 체크를 강제합니다.

```kotlin
// Nullable 타입 명시
val name: String? = null

// Safe call
val length = name?.length  // null이면 null 반환

// Elvis 연산자
val length = name?.length ?: 0  // null이면 0 반환

// Non-null assertion (주의!)
val length = name!!.length  // null이면 예외 발생

// let 함수로 null이 아닐 때만 실행
name?.let { 
    println("이름: $it")
}
```

**실전 예시**:
```kotlin
suspend fun findById(id: UUID): User? =
    databaseFactory.dbQuery {
        Users.select { Users.id eq id }
            .limit(1)
            .firstOrNull()  // 결과가 없으면 null
            ?.let(::toRecord)
            ?.toModel()
    }
```

### 4.4 when 표현식 (패턴 매칭)

```kotlin
val message = when (executionResult.status) {
    ExecutionStatus.SUCCESS -> "실행 성공"
    ExecutionStatus.FAILED -> "실행 실패"
    ExecutionStatus.TIMEOUT -> "시간 초과"
    ExecutionStatus.ERROR -> "오류 발생"
}

// 조건식으로도 사용 가능
val score = when {
    passRate >= 0.9 -> "A"
    passRate >= 0.8 -> "B"
    passRate >= 0.7 -> "C"
    else -> "F"
}
```

### 4.5 스코프 함수 (Scope Functions)

**let**: null이 아닐 때 실행
```kotlin
user?.let { u ->
    println("사용자: ${u.name}")
}
```

**apply**: 객체 설정 후 객체 반환
```kotlin
val config = HikariConfig().apply {
    driverClassName = "org.postgresql.Driver"
    jdbcUrl = "jdbc:postgresql://localhost:5432/db"
    username = "admin"
    password = "password"
    maximumPoolSize = 8
}
```

**also**: 객체에 부가 작업 수행 후 객체 반환
```kotlin
val user = createUser().also { 
    logger.info("사용자 생성: ${it.id}")
}
```

**run**: 람다 실행 후 결과 반환
```kotlin
val result = run {
    val a = 10
    val b = 20
    a + b  // 30 반환
}
```

### 4.6 고차 함수와 람다

```kotlin
// 함수를 매개변수로 받기
suspend fun <T> dbQuery(block: Transaction.() -> T): T =
    withContext(Dispatchers.IO) {
        transaction(database) {
            block(this)  // 람다 실행
        }
    }

// 사용
val user = databaseFactory.dbQuery {
    Users.select { Users.id eq userId }
        .firstOrNull()
}
```

### 4.7 구조 분해 (Destructuring)

```kotlin
// Pair, Triple 구조 분해
val (user, token) = authService.login(email, password)

// 데이터 클래스 구조 분해
val (id, email, name) = user

// Map 순회
for ((key, value) in map) {
    println("$key: $value")
}
```

### 4.8 컬렉션 함수형 API

```kotlin
// map: 변환
val userIds = users.map { it.id }

// filter: 필터링
val activeUsers = users.filter { it.isActive }

// find: 첫 번째 요소 찾기
val admin = users.find { it.role == UserRole.ADMIN }

// groupBy: 그룹화
val usersByRole = users.groupBy { it.role }

// associate: Map 생성
val userMap = users.associate { it.id to it.name }

// 체이닝
val result = submissions
    .filter { it.status == ExecutionStatus.SUCCESS }
    .map { it.score }
    .average()
```

---

## 5. 데이터베이스 연동 (Exposed)

### 5.1 Exposed란?
JetBrains가 만든 Kotlin용 ORM 라이브러리로, 두 가지 API를 제공합니다:
- **DSL API**: SQL과 유사한 타입 안전 쿼리
- **DAO API**: 객체 지향적 접근

이 프로젝트는 **DSL API**를 사용합니다.


### 5.2 테이블 정의

**파일**: `src/main/kotlin/com/codingplatform/database/tables/Users.kt`

```kotlin
object Users : Table("users") {
    val id = uuid("id").autoGenerate()
    val email = varchar("email", 255).uniqueIndex()
    val passwordHash = varchar("password_hash", 255)
    val name = varchar("name", 100)
    val role = varchar("role", 50)
    val createdAt = datetime("created_at")
    val updatedAt = datetime("updated_at")
    val lastLoginAt = datetime("last_login_at").nullable()

    override val primaryKey = PrimaryKey(id)
}
```

**컬럼 타입**:
- `uuid()`: UUID
- `varchar(name, length)`: 문자열
- `integer()`: 정수
- `long()`: Long
- `datetime()`: LocalDateTime
- `bool()`: Boolean
- `text()`: 긴 텍스트
- `nullable()`: null 허용

### 5.3 데이터베이스 연결 설정

**파일**: `src/main/kotlin/com/codingplatform/database/DatabaseFactory.kt`

```kotlin
class DatabaseFactory(
    private val config: DatabaseConfig,
    private val tables: List<org.jetbrains.exposed.sql.Table>
) : Closeable {
    private val dataSource: DataSource = createDataSource()
    val database: Database = Database.connect(dataSource)

    fun init() {
        transaction(database) {
            // 테이블이 없으면 자동 생성
            SchemaUtils.createMissingTablesAndColumns(*tables.toTypedArray())
        }
    }

    // 모든 DB 쿼리는 이 함수를 통해 실행
    suspend fun <T> dbQuery(block: Transaction.() -> T): T =
        withContext(Dispatchers.IO) {
            transaction(database) {
                block(this)
            }
        }

    private fun createDataSource(): HikariDataSource {
        val hikariConfig = HikariConfig().apply {
            driverClassName = config.driver
            jdbcUrl = config.jdbcUrl
            username = config.username
            password = config.password
            maximumPoolSize = config.maxPoolSize
            isAutoCommit = false
            transactionIsolation = "TRANSACTION_REPEATABLE_READ"
            validate()
        }
        return HikariDataSource(hikariConfig)
    }

    override fun close() {
        TransactionManager.closeAndUnregister(database)
        if (dataSource is HikariDataSource) {
            dataSource.close()
        }
    }
}
```

### 5.4 CRUD 작업

**INSERT**:
```kotlin
suspend fun createUser(email: String, password: String, name: String): User =
    databaseFactory.dbQuery {
        val now = Instant.now()
        val id = UUID.randomUUID()
        
        Users.insert { row ->
            row[Users.id] = id
            row[Users.email] = email
            row[Users.passwordHash] = PasswordHasher.hash(password)
            row[Users.name] = name
            row[Users.role] = UserRole.USER.name
            row[Users.createdAt] = now.toLocalDateTimeUtc()
            row[Users.updatedAt] = now.toLocalDateTimeUtc()
        }
        
        User(id, email, name, UserRole.USER, now, now, null)
    }
```

**SELECT**:
```kotlin
suspend fun findById(id: UUID): User? =
    databaseFactory.dbQuery {
        Users.select { Users.id eq id }
            .limit(1)
            .firstOrNull()
            ?.let { row ->
                User(
                    id = row[Users.id].value,
                    email = row[Users.email],
                    name = row[Users.name],
                    role = UserRole.valueOf(row[Users.role]),
                    createdAt = row[Users.createdAt].toInstant(),
                    updatedAt = row[Users.updatedAt].toInstant(),
                    lastLoginAt = row[Users.lastLoginAt]?.toInstant()
                )
            }
    }

// 여러 행 조회
suspend fun findAll(): List<User> =
    databaseFactory.dbQuery {
        Users.selectAll()
            .map { row -> toUser(row) }
    }

// 조건부 조회
suspend fun findByRole(role: UserRole): List<User> =
    databaseFactory.dbQuery {
        Users.select { Users.role eq role.name }
            .map { row -> toUser(row) }
    }
```

**UPDATE**:
```kotlin
suspend fun updateLastLogin(userId: UUID) =
    databaseFactory.dbQuery {
        val now = Instant.now()
        Users.update({ Users.id eq userId }) {
            it[lastLoginAt] = now.toLocalDateTimeUtc()
            it[updatedAt] = now.toLocalDateTimeUtc()
        }
    }
```

**DELETE**:
```kotlin
suspend fun deleteUser(userId: UUID) =
    databaseFactory.dbQuery {
        Users.deleteWhere { Users.id eq userId }
    }
```

### 5.5 복잡한 쿼리

**JOIN**:
```kotlin
suspend fun getSubmissionsWithProblems(): List<SubmissionWithProblem> =
    databaseFactory.dbQuery {
        (Submissions innerJoin Problems)
            .selectAll()
            .map { row ->
                SubmissionWithProblem(
                    submission = toSubmission(row),
                    problem = toProblem(row)
                )
            }
    }
```

**집계 함수**:
```kotlin
suspend fun getUserStats(userId: UUID): UserStats =
    databaseFactory.dbQuery {
        val totalSubmissions = Submissions
            .select { Submissions.userId eq userId }
            .count()
        
        val avgScore = Submissions
            .slice(Submissions.score.avg())
            .select { Submissions.userId eq userId }
            .firstOrNull()
            ?.get(Submissions.score.avg())
            ?: 0.0
        
        UserStats(totalSubmissions, avgScore)
    }
```


---

## 6. 의존성 주입 패턴

이 프로젝트는 별도의 DI 프레임워크(Koin, Kodein) 없이 **수동 의존성 주입**을 사용합니다.

### 6.1 ServiceRegistry 패턴

**파일**: `src/main/kotlin/com/codingplatform/services/ServiceRegistry.kt`

```kotlin
class ServiceRegistry(environment: ApplicationEnvironment) {
    private val config: ApplicationConfig = environment.config

    // 공통 인프라
    val metricsRegistry = PrometheusMeterRegistry(PrometheusConfig.DEFAULT)
    val jwtConfig: JwtConfig
    val databaseFactory: DatabaseFactory
    val redisClient: RedisClient
    val redisConnection: StatefulRedisConnection<String, String>

    // 비즈니스 서비스
    val dockerManager: DockerManager
    val testRunnerService: TestRunnerService
    val dockerExecutorService: DockerExecutorService
    val emailService: EmailService
    val emailVerificationService: EmailVerificationService
    val authService: AuthService
    val userService: UserService
    val problemService: ProblemService
    val submissionService: SubmissionService
    val dashboardService: DashboardService
    val adminService: AdminService

    init {
        // 1. 설정 로드
        val dbConfig = DatabaseConfig(
            jdbcUrl = value("database.jdbcUrl", "DATABASE_URL") ?: buildJdbcUrl(),
            driver = value("database.driver", "DATABASE_DRIVER") ?: "org.postgresql.Driver",
            username = value("database.username", "POSTGRES_USER") ?: "admin",
            password = value("database.password", "POSTGRES_PASSWORD") ?: "password",
            maxPoolSize = value("database.poolSize", "DB_POOL_SIZE")?.toIntOrNull() ?: 8
        )

        // 2. 인프라 초기화
        databaseFactory = DatabaseFactory(
            config = dbConfig,
            tables = listOf(Users, Problems, Submissions, Executions, EmailVerifications)
        ).also { it.init() }

        jwtConfig = JwtConfig(
            secret = value("jwt.secret", "JWT_SECRET") ?: error("JWT_SECRET is required"),
            issuer = value("jwt.issuer", "JWT_ISSUER") ?: "coding-platform",
            audience = value("jwt.audience", "JWT_AUDIENCE") ?: "coding-platform-users",
            realm = value("jwt.realm", "JWT_REALM") ?: "coding-platform-auth",
            expirationSeconds = value("jwt.expirationSeconds", "JWT_EXP_SECONDS")?.toLongOrNull() ?: 60L * 60 * 6
        )

        val redisHost = value("redis.host", "REDIS_HOST") ?: "localhost"
        val redisPort = value("redis.port", "REDIS_PORT") ?: "6379"
        redisClient = RedisClient.create("redis://$redisHost:$redisPort")
        redisConnection = redisClient.connect()

        // 3. 서비스 초기화 (의존성 순서 중요!)
        dockerManager = DockerManager()
        testRunnerService = TestRunnerService()
        dockerExecutorService = DockerExecutorService(dockerManager, testRunnerService)
        
        emailService = EmailService(loadEmailConfig())
        emailVerificationService = EmailVerificationService(databaseFactory, emailService)
        
        authService = AuthService(databaseFactory, jwtConfig, emailVerificationService)
        problemService = ProblemService(databaseFactory).also { service ->
            runBlocking { service.seedDefaults() }  // 초기 데이터 생성
        }
        submissionService = SubmissionService(databaseFactory, problemService, dockerExecutorService)
        userService = UserService(databaseFactory, authService)
        dashboardService = DashboardService(databaseFactory)
        adminService = AdminService(databaseFactory)
    }

    suspend fun shutdown() {
        redisConnection.close()
        redisClient.shutdown()
        dockerManager.close()
        databaseFactory.close()
    }

    private fun value(path: String, envKey: String? = null): String? {
        val envValue = envKey?.let(System::getenv)
        if (!envValue.isNullOrBlank()) return envValue
        return config.propertyOrNull(path)?.getString()
    }
}
```

**장점**:
- 명시적인 의존성 관계
- 초기화 순서 제어 가능
- 추가 라이브러리 불필요
- 테스트 시 Mock 주입 용이

### 6.2 생성자 주입

```kotlin
class AuthService(
    private val databaseFactory: DatabaseFactory,
    private val jwtConfig: JwtConfig,
    private val emailVerificationService: EmailVerificationService
) {
    // 의존성이 생성자를 통해 주입됨
    suspend fun register(...): Pair<User, String> {
        // databaseFactory, jwtConfig, emailVerificationService 사용
    }
}
```

---

## 7. 실전 예제 분석

### 7.1 사용자 인증 플로우

**1단계: 인증 코드 요청**
```kotlin
// Route
post("/api/auth/register/code") {
    val request = call.receive<VerificationCodeRequest>()
    authService.requestVerificationCode(request.email)
    call.respond(HttpStatusCode.Accepted, mapOf("message" to "인증 코드가 발송되었습니다."))
}

// Service
suspend fun requestVerificationCode(email: String) {
    require(findByEmail(email) == null) { "이미 사용 중인 이메일입니다." }
    emailVerificationService.requestCode(email)
}
```

**2단계: 회원가입**
```kotlin
// Route
post("/api/auth/register") {
    val request = call.receive<RegisterRequest>()
    val (user, token) = authService.register(
        email = request.email,
        password = request.password,
        name = request.name,
        verificationCode = request.verificationCode,
        role = UserRole.USER
    )
    call.respond(HttpStatusCode.Created, AuthResponse(token = token, user = user))
}

// Service
suspend fun register(...): Pair<User, String> {
    // 1. 이메일 중복 체크
    require(findByEmail(email) == null) { "이미 사용 중인 이메일입니다." }
    
    // 2. 인증 코드 검증
    emailVerificationService.consumeCode(email, verificationCode)

    // 3. 사용자 생성
    val user = databaseFactory.dbQuery {
        val now = Instant.now()
        val id = UUID.randomUUID()
        Users.insert { row ->
            row[Users.id] = id
            row[Users.email] = email
            row[Users.passwordHash] = PasswordHasher.hash(password)
            row[Users.name] = name
            row[Users.role] = role.name
            row[Users.createdAt] = now.toLocalDateTimeUtc()
            row[Users.updatedAt] = now.toLocalDateTimeUtc()
        }
        User(id, email, name, role, now, now, null)
    }

    // 4. JWT 토큰 생성
    return user to jwtConfig.generateToken(user)
}
```


### 7.2 코드 제출 및 평가 플로우

**전체 흐름**:
1. 사용자가 코드 제출
2. 파일 시스템에 코드 저장
3. Docker 컨테이너에서 테스트 실행
4. 테스트 결과 파싱
5. 점수 계산 및 피드백 생성

**Route**: `src/main/kotlin/com/codingplatform/routes/SubmissionRoutes.kt`
```kotlin
authenticate("auth-jwt") {
    post("/api/submissions") {
        val principal = call.principal<JWTPrincipal>()
        val userId = principal?.payload?.subject?.let { UUID.fromString(it) }
            ?: throw IllegalArgumentException("인증 정보가 없습니다.")

        val request = call.receive<SubmissionRequest>()
        
        // 비동기로 제출 처리
        val submission = submissionService.createSubmission(
            userId = userId,
            problemId = request.problemId,
            files = request.files
        )
        
        call.respond(HttpStatusCode.Created, submission)
    }
}
```

**Service**: `src/main/kotlin/com/codingplatform/services/SubmissionService.kt`
```kotlin
suspend fun createSubmission(
    userId: UUID,
    problemId: UUID,
    files: Map<String, String>
): Submission {
    // 1. 문제 조회
    val problem = problemService.findById(problemId)
        ?: throw IllegalArgumentException("존재하지 않는 문제입니다.")

    // 2. 제출 레코드 생성
    val submission = databaseFactory.dbQuery {
        val id = UUID.randomUUID().toString()
        val now = Instant.now()
        
        Submissions.insert { row ->
            row[Submissions.id] = id
            row[Submissions.userId] = userId
            row[Submissions.problemId] = problemId
            row[Submissions.language] = problem.language.name
            row[Submissions.files] = Json.encodeToString(files)
            row[Submissions.status] = ExecutionStatus.PENDING.name
            row[Submissions.createdAt] = now.toLocalDateTimeUtc()
        }
        
        Submission(id, userId, problemId, problem.language, files, 
                   ExecutionStatus.PENDING, null, null, 0, now)
    }

    // 3. 비동기로 평가 실행 (백그라운드)
    launch {
        evaluateSubmission(submission.id)
    }

    return submission
}

private suspend fun evaluateSubmission(submissionId: String) {
    try {
        // 제출 정보 조회
        val submission = findById(submissionId) ?: return
        val problem = problemService.findById(submission.problemId) ?: return

        // Docker에서 테스트 실행
        val feedback = dockerExecutorService.evaluateSubmission(submission, problem)

        // 결과 저장
        databaseFactory.dbQuery {
            Submissions.update({ Submissions.id eq submissionId }) {
                it[status] = feedback.status.name
                it[score] = feedback.score
                it[feedback] = Json.encodeToString(feedback)
            }
        }
    } catch (e: Exception) {
        logger.error("평가 실패: $submissionId", e)
        // 에러 상태로 업데이트
        databaseFactory.dbQuery {
            Submissions.update({ Submissions.id eq submissionId }) {
                it[status] = ExecutionStatus.ERROR.name
            }
        }
    }
}
```

**DockerExecutorService**: 실제 코드 실행
```kotlin
suspend fun evaluateSubmission(
    submission: Submission, 
    problem: Problem
): SubmissionFeedback = withContext(Dispatchers.IO) {
    require(problem.testFiles.isNotEmpty()) { "테스트 코드가 등록되지 않은 문제입니다." }

    // 1. 테스트 실행
    val (executionResult, testResults) = runTests(submission, problem)

    // 2. 점수 계산
    val passRate = if (testResults.total == 0) 0.0 
                   else testResults.passed.toDouble() / testResults.total
    val score = (passRate * 100).roundToInt()

    // 3. 피드백 생성
    SubmissionFeedback(
        totalTests = testResults.total,
        passedTests = testResults.passed,
        failedTests = testResults.failed,
        passRate = passRate,
        score = score,
        status = executionResult.status,
        testResults = testResults,
        output = executionResult.output,
        message = if (testResults.failed == 0) "모든 테스트를 통과했습니다." 
                  else "${testResults.failed}개의 테스트가 실패했습니다."
    )
}

private fun runTests(submission: Submission, problem: Problem): TestRunResult {
    // 1. 파일 준비
    val buildFiles = prepareBuildFiles(problem)
    val testFiles = problem.testFiles
    val userSolutionFiles = prepareUserSolutionFiles(submission.files, problem.language)
    val allFiles = userSolutionFiles + testFiles + buildFiles

    // 2. 실행 명령 생성
    val command = when (problem.language) {
        Language.KOTLIN, Language.JAVA -> 
            listOf("sh", "-c", "gradle test --no-daemon")
        Language.PYTHON -> 
            listOf("sh", "-c", "pytest -vv --tb=long --junitxml=test-results.xml")
    }

    // 3. Docker 컨테이너에서 실행
    val executionResult = dockerManager.executeCode(
        executionId = submission.id,
        language = problem.language,
        files = allFiles,
        command = command
    )

    // 4. 테스트 결과 파싱
    val testResults = testRunnerService.parseTestResults(executionResult)

    return TestRunResult(executionResult, testResults)
}
```


### 7.3 Docker 컨테이너 실행 상세

**DockerManager**: `src/main/kotlin/com/codingplatform/executor/DockerManager.kt`

```kotlin
fun executeCode(
    executionId: String,
    language: Language,
    files: Map<String, String>,
    command: List<String>
): ExecutionResult {
    // 1. 보안 검증
    securityManager.validateFiles(files)

    // 2. 작업 공간 생성
    val workspace = createWorkspace(executionId, files)

    return try {
        // 3. Docker 이미지 준비
        val image = ensureImageBuilt(language)

        // 4. 컨테이너 생성
        val container = createContainer(executionId, image, workspace, command, language)

        // 5. 컨테이너 실행 및 결과 수집
        val result = runContainer(container, timeoutSeconds)

        ExecutionResult(
            executionId = executionId,
            status = result.status,
            output = result.stdout,
            error = result.stderr,
            exitCode = result.exitCode,
            executionTime = result.duration.toMillis(),
            memoryUsed = result.memoryUsed
        )
    } catch (ex: Exception) {
        ExecutionResult(
            executionId = executionId,
            status = ExecutionStatus.ERROR,
            output = "",
            error = ex.message,
            exitCode = -1,
            executionTime = 0,
            memoryUsed = 0
        )
    } finally {
        // 6. 정리
        cleanupWorkspace(workspace)
    }
}

private fun createContainer(
    executionId: String,
    image: String,
    workspace: File,
    command: List<String>,
    language: Language
): CreateContainerResponse {
    // 호스트 경로를 컨테이너 경로로 변환
    val hostPath = workspace.absolutePath.replace(executionWorkspace, dockerHostWorkspace)

    // 보안 설정이 적용된 HostConfig
    val hostConfig = HostConfig.newHostConfig()
        .withMemory(maxMemoryBytes)              // 메모리 제한
        .withMemorySwap(maxMemoryBytes)          // 스왑 제한
        .withCpuShares(maxCpuShares)             // CPU 제한
        .withNetworkMode("bridge")               // 네트워크 격리
        .withReadonlyRootfs(false)               // 루트 파일시스템
        .withBinds(Bind(hostPath, Volume("/workspace"), AccessMode.rw))  // 볼륨 마운트
        .withCapDrop(Capability.ALL)             // 모든 권한 제거
        .withSecurityOpts(listOf("no-new-privileges"))  // 권한 상승 방지

    return dockerClient.createContainerCmd(image)
        .withName("exec-$executionId")
        .withWorkingDir("/workspace")
        .withCmd(command)
        .withHostConfig(hostConfig)
        .withAttachStdout(true)
        .withAttachStderr(true)
        .withUser("1000:1000")  // 비특권 사용자
        .withEnv(listOf("EXECUTION_TIMEOUT=$timeoutSeconds"))
        .exec()
}

private fun runContainer(
    container: CreateContainerResponse, 
    timeout: Long
): ContainerExecutionResult {
    val containerId = container.id
    val start = System.nanoTime()

    // 컨테이너 시작
    dockerClient.startContainerCmd(containerId).exec()

    // 로그 수집
    val stdout = StringBuilder()
    val stderr = StringBuilder()

    val logCallback = object : LogContainerResultCallback() {
        override fun onNext(item: Frame) {
            when (item.streamType) {
                StreamType.STDOUT -> stdout.append(item.payload.decodeToString())
                StreamType.STDERR -> stderr.append(item.payload.decodeToString())
                else -> {}
            }
        }
    }

    dockerClient.logContainerCmd(containerId)
        .withStdOut(true)
        .withStdErr(true)
        .withFollowStream(true)
        .exec(logCallback)

    // 컨테이너 종료 대기 (타임아웃 적용)
    val exitCode = try {
        dockerClient.waitContainerCmd(containerId)
            .exec(WaitContainerResultCallback())
            .awaitStatusCode(timeout, TimeUnit.SECONDS)
    } catch (ex: Exception) {
        // 타임아웃 시 강제 종료
        dockerClient.killContainerCmd(containerId).exec()
        -1
    } finally {
        logCallback.awaitCompletion()
        logCallback.close()
    }

    val duration = Duration.ofNanos(System.nanoTime() - start)

    // 컨테이너 삭제
    dockerClient.removeContainerCmd(containerId)
        .withForce(true)
        .exec()

    val status = when {
        exitCode == 0 -> ExecutionStatus.SUCCESS
        exitCode == -1 -> ExecutionStatus.TIMEOUT
        else -> ExecutionStatus.FAILED
    }

    return ContainerExecutionResult(
        stdout = stdout.toString(),
        stderr = stderr.toString(),
        exitCode = exitCode,
        status = status,
        duration = duration,
        memoryUsed = 0
    )
}
```

**핵심 포인트**:
1. **보안**: 모든 권한 제거, 메모리/CPU 제한, 네트워크 격리
2. **타임아웃**: 무한 실행 방지
3. **로그 수집**: stdout/stderr 분리 수집
4. **리소스 정리**: 컨테이너 자동 삭제

---

## 8. 학습 로드맵

### 8.1 초급 (1-2주)
1. **Kotlin 기본 문법**
   - 변수, 함수, 클래스
   - Null 안전성
   - 데이터 클래스
   - when 표현식

2. **Ktor 기초**
   - 프로젝트 설정
   - 간단한 라우트 작성
   - JSON 직렬화/역직렬화
   - 에러 처리

3. **실습 과제**
   - Hello World API 만들기
   - CRUD API 구현 (메모리 저장)

### 8.2 중급 (2-3주)
1. **코루틴 기초**
   - suspend 함수
   - Dispatchers
   - async/await

2. **데이터베이스 연동**
   - Exposed DSL
   - CRUD 작업
   - 트랜잭션

3. **인증/인가**
   - JWT 토큰
   - 보호된 라우트

4. **실습 과제**
   - 사용자 인증 시스템 구현
   - 게시판 API 만들기

### 8.3 고급 (3-4주)
1. **고급 코루틴**
   - 구조화된 동시성
   - Flow
   - 에러 처리

2. **아키텍처 패턴**
   - 의존성 주입
   - 레이어드 아키텍처
   - 테스트 작성

3. **실전 기능**
   - 파일 업로드
   - 이메일 발송
   - 캐싱 (Redis)
   - 모니터링 (Prometheus)

4. **실습 과제**
   - 이 프로젝트 분석 및 기능 추가
   - 새로운 도메인 추가 (예: 댓글, 좋아요)

---

## 9. 추가 학습 자료

### 9.1 공식 문서
- [Kotlin 공식 문서](https://kotlinlang.org/docs/home.html)
- [Ktor 공식 문서](https://ktor.io/docs/)
- [Kotlin Coroutines 가이드](https://kotlinlang.org/docs/coroutines-guide.html)
- [Exposed Wiki](https://github.com/JetBrains/Exposed/wiki)

### 9.2 프로젝트 내 참고 파일
- `docs/API_SPEC.md`: API 명세
- `docs/CONTAINER_EXECUTION.md`: Docker 실행 상세
- `docs/TEST_SCORING.md`: 테스트 점수 계산
- `AGENTS.md`: 프로젝트 가이드라인

### 9.3 디버깅 팁
```bash
# 로그 확인
docker compose logs -f backend

# 데이터베이스 접속
docker exec -it postgres psql -U admin -d coding_platform

# Redis 접속
docker exec -it redis redis-cli

# 컨테이너 상태 확인
docker ps -a
```

---

## 10. 자주 하는 실수와 해결법

### 10.1 코루틴 관련
**문제**: `suspend` 함수를 일반 함수에서 호출
```kotlin
// ❌ 잘못된 예
fun getData() {
    val user = userService.findById(id)  // 컴파일 에러!
}

// ✅ 올바른 예
suspend fun getData() {
    val user = userService.findById(id)
}
```

### 10.2 Null 안전성
**문제**: Nullable 타입 처리 누락
```kotlin
// ❌ 잘못된 예
val user: User? = findUser()
println(user.name)  // 컴파일 에러!

// ✅ 올바른 예
val user: User? = findUser()
println(user?.name ?: "Unknown")
```

### 10.3 데이터베이스 트랜잭션
**문제**: `dbQuery` 밖에서 Exposed 함수 호출
```kotlin
// ❌ 잘못된 예
suspend fun getUsers(): List<User> {
    return Users.selectAll().map { toUser(it) }  // 에러!
}

// ✅ 올바른 예
suspend fun getUsers(): List<User> {
    return databaseFactory.dbQuery {
        Users.selectAll().map { toUser(it) }
    }
}
```

### 10.4 리소스 정리
**문제**: 리소스 누수
```kotlin
// ❌ 잘못된 예
val connection = redisClient.connect()
// 사용 후 close() 호출 안 함

// ✅ 올바른 예
val connection = redisClient.connect()
try {
    // 사용
} finally {
    connection.close()
}

// 또는 use 함수 사용
redisClient.connect().use { connection ->
    // 자동으로 close 호출됨
}
```

---

이 가이드를 통해 Kotlin, Ktor, 코루틴의 실전 사용법을 익히고, 
실제 프로덕션 수준의 백엔드 애플리케이션을 개발할 수 있는 능력을 키우시기 바랍니다!
