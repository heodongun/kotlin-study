# Ktor 심화 가이드

이 문서는 Coding Platform 프로젝트에서 사용되는 Ktor 프레임워크의 고급 기능과 실전 패턴을 다룹니다.

## 목차
1. [Ktor 아키텍처](#1-ktor-아키텍처)
2. [플러그인 시스템](#2-플러그인-시스템)
3. [라우팅 고급 기법](#3-라우팅-고급-기법)
4. [인증과 인가](#4-인증과-인가)
5. [요청/응답 처리](#5-요청응답-처리)
6. [에러 처리와 검증](#6-에러-처리와-검증)
7. [테스트](#7-테스트)

---

## 1. Ktor 아키텍처

### 1.1 Ktor의 특징
- **비동기 우선**: 코루틴 기반으로 설계
- **경량**: 필요한 기능만 플러그인으로 추가
- **DSL 스타일**: Kotlin의 강력한 DSL 활용
- **멀티플랫폼**: JVM, Native, JS 지원

### 1.2 애플리케이션 구조

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080, host = "0.0.0.0") {
        module()
    }.start(wait = true)
}

fun Application.module() {
    // 1. 인프라 초기화
    val services = ServiceRegistry(environment)
    
    // 2. 플러그인 설치 (순서 중요!)
    configureMonitoring(services.metricsRegistry)
    configureSerialization()
    configureStatusPages()
    configureSecurity(services.jwtConfig)
    configureDatabase(services.databaseFactory)
    configureRouting(services)
    
    // 3. 생명주기 관리
    environment.monitor.subscribe(ApplicationStopping) {
        runBlocking { services.shutdown() }
    }
}
```

**플러그인 설치 순서가 중요한 이유**:
1. `Monitoring`: 모든 요청을 추적하므로 가장 먼저
2. `Serialization`: 요청/응답 변환
3. `StatusPages`: 에러 처리
4. `Security`: 인증/인가
5. `Routing`: 실제 엔드포인트 (마지막)

### 1.3 환경 설정

**application.conf** (HOCON 형식):
```hocon
ktor {
    deployment {
        port = 8080
        port = ${?APP_PORT}
    }
    application {
        modules = [ com.codingplatform.ApplicationKt.module ]
    }
}

database {
    jdbcUrl = "jdbc:postgresql://localhost:5432/coding_platform"
    jdbcUrl = ${?DATABASE_URL}
    driver = "org.postgresql.Driver"
    username = "admin"
    username = ${?POSTGRES_USER}
    password = "password"
    password = ${?POSTGRES_PASSWORD}
}
```

**환경 변수 읽기**:
```kotlin
class ServiceRegistry(environment: ApplicationEnvironment) {
    private val config: ApplicationConfig = environment.config

    private fun value(path: String, envKey: String? = null): String? {
        // 1. 환경 변수 우선
        val envValue = envKey?.let(System::getenv)
        if (!envValue.isNullOrBlank()) return envValue
        
        // 2. application.conf 값
        return config.propertyOrNull(path)?.getString()
    }
}
```

---

## 2. 플러그인 시스템

### 2.1 Content Negotiation (직렬화)

```kotlin
fun Application.configureSerialization() {
    install(ContentNegotiation) {
        json(Json {
            prettyPrint = true
            isLenient = true
            ignoreUnknownKeys = true
            encodeDefaults = true
        })
    }
}
```

**사용 예시**:
```kotlin
@Serializable
data class User(
    val id: String,
    val email: String,
    val name: String
)

post("/api/users") {
    val user = call.receive<User>()  // 자동 역직렬화
    // ...
    call.respond(HttpStatusCode.Created, user)  // 자동 직렬화
}
```


### 2.2 CORS (Cross-Origin Resource Sharing)

```kotlin
fun Application.configureSecurity(jwtConfig: JwtConfig) {
    install(CORS) {
        // 개발 환경
        anyHost()
        
        // 프로덕션 환경 (특정 도메인만 허용)
        // allowHost("example.com", schemes = listOf("https"))
        // allowHost("localhost:3000")
        
        allowMethod(HttpMethod.Get)
        allowMethod(HttpMethod.Post)
        allowMethod(HttpMethod.Put)
        allowMethod(HttpMethod.Delete)
        allowMethod(HttpMethod.Options)
        allowMethod(HttpMethod.Patch)
        
        allowHeader(HttpHeaders.Authorization)
        allowHeader(HttpHeaders.ContentType)
        allowHeader(HttpHeaders.AccessControlAllowOrigin)
        
        allowCredentials = true
        maxAgeInSeconds = 3600
    }
}
```

### 2.3 Call Logging

```kotlin
fun Application.configureSecurity(jwtConfig: JwtConfig) {
    install(CallLogging) {
        level = Level.INFO
        filter { call -> call.request.path().startsWith("/api") }
        format { call ->
            val status = call.response.status()
            val httpMethod = call.request.httpMethod.value
            val uri = call.request.uri
            val duration = call.processingTimeMillis()
            "$httpMethod $uri - $status (${duration}ms)"
        }
    }
}
```

### 2.4 Status Pages (에러 처리)

```kotlin
fun Application.configureStatusPages() {
    install(StatusPages) {
        // 특정 예외 처리
        exception<IllegalArgumentException> { call, cause ->
            call.respond(
                HttpStatusCode.BadRequest,
                mapOf("error" to (cause.message ?: "잘못된 요청입니다"))
            )
        }
        
        exception<NoSuchElementException> { call, cause ->
            call.respond(
                HttpStatusCode.NotFound,
                mapOf("error" to (cause.message ?: "리소스를 찾을 수 없습니다"))
            )
        }
        
        exception<IllegalStateException> { call, cause ->
            call.respond(
                HttpStatusCode.Conflict,
                mapOf("error" to (cause.message ?: "충돌이 발생했습니다"))
            )
        }
        
        // 모든 예외 처리 (마지막 방어선)
        exception<Throwable> { call, cause ->
            call.application.environment.log.error("처리되지 않은 예외", cause)
            call.respond(
                HttpStatusCode.InternalServerError,
                mapOf("error" to "서버 오류가 발생했습니다")
            )
        }
        
        // HTTP 상태 코드 처리
        status(HttpStatusCode.NotFound) { call, status ->
            call.respond(
                status,
                mapOf("error" to "엔드포인트를 찾을 수 없습니다: ${call.request.uri}")
            )
        }
    }
}
```

### 2.5 Metrics (모니터링)

```kotlin
fun Application.configureMonitoring(registry: PrometheusMeterRegistry) {
    install(MicrometerMetrics) {
        this.registry = registry
        
        // 커스텀 메트릭
        meterBinders = listOf(
            JvmMemoryMetrics(),
            JvmGcMetrics(),
            ProcessorMetrics(),
            JvmThreadMetrics()
        )
    }
}

// Prometheus 엔드포인트
routing {
    get("/metrics") {
        call.respond(registry.scrape())
    }
}
```

---

## 3. 라우팅 고급 기법

### 3.1 라우트 그룹화

```kotlin
fun Application.configureRouting(services: ServiceRegistry) {
    routing {
        // API 버전 그룹
        route("/api") {
            route("/v1") {
                configureAuthRoutes(services.authService)
                configureProblemRoutes(services.problemService)
            }
            
            route("/v2") {
                // 새 버전 API
            }
        }
        
        // 관리자 전용 라우트
        route("/admin") {
            authenticate("auth-jwt") {
                configureAdminRoutes(services.adminService)
            }
        }
    }
}
```

### 3.2 경로 매개변수

```kotlin
fun Route.configureProblemRoutes(problemService: ProblemService) {
    route("/api/problems") {
        // GET /api/problems/{id}
        get("/{id}") {
            val id = call.parameters["id"]?.let { UUID.fromString(it) }
                ?: throw IllegalArgumentException("유효하지 않은 ID입니다")
            
            val problem = problemService.findById(id)
                ?: throw NoSuchElementException("문제를 찾을 수 없습니다")
            
            call.respond(problem)
        }
        
        // GET /api/problems/{id}/submissions
        get("/{id}/submissions") {
            val problemId = call.parameters["id"]?.let { UUID.fromString(it) }
                ?: throw IllegalArgumentException("유효하지 않은 ID입니다")
            
            val submissions = submissionService.findByProblemId(problemId)
            call.respond(submissions)
        }
    }
}
```

### 3.3 쿼리 파라미터

```kotlin
get("/api/problems") {
    // GET /api/problems?page=1&size=10&difficulty=MEDIUM
    val page = call.request.queryParameters["page"]?.toIntOrNull() ?: 1
    val size = call.request.queryParameters["size"]?.toIntOrNull() ?: 20
    val difficulty = call.request.queryParameters["difficulty"]
    
    val problems = problemService.findAll(
        page = page,
        size = size,
        difficulty = difficulty?.let { Difficulty.valueOf(it) }
    )
    
    call.respond(problems)
}
```

### 3.4 요청 헤더

```kotlin
get("/api/profile") {
    val token = call.request.header(HttpHeaders.Authorization)
        ?.removePrefix("Bearer ")
        ?: throw IllegalArgumentException("인증 토큰이 없습니다")
    
    val userId = jwtConfig.verifyToken(token)
    val user = userService.findById(userId)
    
    call.respond(user)
}
```

### 3.5 응답 헤더 설정

```kotlin
post("/api/problems") {
    val problem = call.receive<ProblemRequest>()
    val created = problemService.create(problem)
    
    call.response.header(
        HttpHeaders.Location,
        "/api/problems/${created.id}"
    )
    call.respond(HttpStatusCode.Created, created)
}
```


---

## 4. 인증과 인가

### 4.1 JWT 인증 설정

```kotlin
fun Application.configureSecurity(jwtConfig: JwtConfig) {
    install(Authentication) {
        jwt("auth-jwt") {
            realm = jwtConfig.realm
            verifier(jwtConfig.verifier())
            
            validate { credential ->
                val subject = credential.payload.subject
                val role = credential.payload.getClaim("role").asString()
                
                if (!subject.isNullOrBlank()) {
                    JWTPrincipal(credential.payload)
                } else {
                    null
                }
            }
        }
    }
}
```

**JwtConfig 구현**:
```kotlin
class JwtConfig(
    val secret: String,
    val issuer: String,
    val audience: String,
    val realm: String,
    val expirationSeconds: Long
) {
    private val algorithm = Algorithm.HMAC256(secret)

    fun verifier(): JWTVerifier = JWT
        .require(algorithm)
        .withIssuer(issuer)
        .withAudience(audience)
        .build()

    fun generateToken(user: User): String = JWT.create()
        .withSubject(user.id.toString())
        .withIssuer(issuer)
        .withAudience(audience)
        .withClaim("email", user.email)
        .withClaim("name", user.name)
        .withClaim("role", user.role.name)
        .withExpiresAt(Date(System.currentTimeMillis() + expirationSeconds * 1000))
        .sign(algorithm)
}
```

### 4.2 보호된 라우트

```kotlin
fun Route.configureUserRoutes(userService: UserService) {
    // 인증 필요
    authenticate("auth-jwt") {
        get("/api/profile") {
            val principal = call.principal<JWTPrincipal>()
            val userId = principal?.payload?.subject?.let { UUID.fromString(it) }
                ?: throw IllegalArgumentException("인증 정보가 없습니다")
            
            val user = userService.findById(userId)
                ?: throw NoSuchElementException("사용자를 찾을 수 없습니다")
            
            call.respond(user)
        }
        
        put("/api/profile") {
            val principal = call.principal<JWTPrincipal>()
            val userId = principal?.payload?.subject?.let { UUID.fromString(it) }
                ?: throw IllegalArgumentException("인증 정보가 없습니다")
            
            val request = call.receive<UpdateProfileRequest>()
            val updated = userService.updateProfile(userId, request)
            
            call.respond(updated)
        }
    }
}
```

### 4.3 역할 기반 인가 (RBAC)

**커스텀 인가 플러그인**:
```kotlin
class RoleBasedAuthorization(config: Configuration) {
    private val requiredRoles = config.roles

    class Configuration {
        var roles: Set<UserRole> = emptySet()
    }

    companion object Plugin : RouteScopedPlugin<Configuration, RoleBasedAuthorization> {
        override val key = AttributeKey<RoleBasedAuthorization>("RoleBasedAuthorization")

        override fun install(
            pipeline: ApplicationCallPipeline,
            configure: Configuration.() -> Unit
        ): RoleBasedAuthorization {
            val config = Configuration().apply(configure)
            val plugin = RoleBasedAuthorization(config)

            pipeline.intercept(ApplicationCallPipeline.Plugins) {
                val principal = call.principal<JWTPrincipal>()
                    ?: throw IllegalArgumentException("인증이 필요합니다")

                val userRole = principal.payload.getClaim("role").asString()
                    ?.let { UserRole.valueOf(it) }
                    ?: throw IllegalArgumentException("역할 정보가 없습니다")

                if (userRole !in plugin.requiredRoles) {
                    call.respond(
                        HttpStatusCode.Forbidden,
                        mapOf("error" to "권한이 없습니다")
                    )
                    finish()
                }
            }

            return plugin
        }
    }
}
```

**사용 예시**:
```kotlin
fun Route.configureAdminRoutes(adminService: AdminService) {
    authenticate("auth-jwt") {
        install(RoleBasedAuthorization) {
            roles = setOf(UserRole.ADMIN)
        }
        
        route("/api/admin") {
            get("/users") {
                val users = adminService.getAllUsers()
                call.respond(users)
            }
            
            delete("/users/{id}") {
                val userId = call.parameters["id"]?.let { UUID.fromString(it) }
                    ?: throw IllegalArgumentException("유효하지 않은 ID입니다")
                
                adminService.deleteUser(userId)
                call.respond(HttpStatusCode.NoContent)
            }
        }
    }
}
```

### 4.4 API 키 인증

```kotlin
fun Application.configureSecurity() {
    install(Authentication) {
        // JWT 인증
        jwt("auth-jwt") { /* ... */ }
        
        // API 키 인증
        bearer("api-key") {
            authenticate { tokenCredential ->
                val apiKey = tokenCredential.token
                if (isValidApiKey(apiKey)) {
                    UserIdPrincipal("api-user")
                } else {
                    null
                }
            }
        }
    }
}

// 사용
authenticate("api-key") {
    get("/api/public/stats") {
        // API 키로 인증된 요청만 허용
    }
}
```

---

## 5. 요청/응답 처리

### 5.1 요청 본문 검증

```kotlin
@Serializable
data class RegisterRequest(
    val email: String,
    val password: String,
    val name: String,
    val verificationCode: String
) {
    init {
        require(email.matches(Regex("^[A-Za-z0-9+_.-]+@(.+)$"))) {
            "유효하지 않은 이메일 형식입니다"
        }
        require(password.length >= 8) {
            "비밀번호는 최소 8자 이상이어야 합니다"
        }
        require(name.isNotBlank()) {
            "이름은 필수입니다"
        }
        require(verificationCode.length == 6) {
            "인증 코드는 6자리여야 합니다"
        }
    }
}

post("/api/auth/register") {
    val request = try {
        call.receive<RegisterRequest>()
    } catch (e: IllegalArgumentException) {
        throw IllegalArgumentException("요청 검증 실패: ${e.message}")
    }
    
    val (user, token) = authService.register(
        email = request.email,
        password = request.password,
        name = request.name,
        verificationCode = request.verificationCode
    )
    
    call.respond(HttpStatusCode.Created, AuthResponse(token, user))
}
```

### 5.2 페이지네이션

```kotlin
@Serializable
data class PageRequest(
    val page: Int = 1,
    val size: Int = 20
) {
    init {
        require(page >= 1) { "페이지는 1 이상이어야 합니다" }
        require(size in 1..100) { "페이지 크기는 1-100 사이여야 합니다" }
    }
    
    val offset: Long get() = ((page - 1) * size).toLong()
}

@Serializable
data class PageResponse<T>(
    val items: List<T>,
    val page: Int,
    val size: Int,
    val total: Long,
    val totalPages: Int
)

get("/api/problems") {
    val page = call.request.queryParameters["page"]?.toIntOrNull() ?: 1
    val size = call.request.queryParameters["size"]?.toIntOrNull() ?: 20
    val pageRequest = PageRequest(page, size)
    
    val (items, total) = problemService.findAll(pageRequest)
    val totalPages = (total + size - 1) / size
    
    call.respond(PageResponse(
        items = items,
        page = page,
        size = size,
        total = total,
        totalPages = totalPages.toInt()
    ))
}
```

### 5.3 파일 업로드

```kotlin
post("/api/problems/{id}/files") {
    val problemId = call.parameters["id"]?.let { UUID.fromString(it) }
        ?: throw IllegalArgumentException("유효하지 않은 ID입니다")
    
    val multipart = call.receiveMultipart()
    val files = mutableMapOf<String, String>()
    
    multipart.forEachPart { part ->
        when (part) {
            is PartData.FileItem -> {
                val fileName = part.originalFileName ?: "unknown"
                val content = part.streamProvider().readBytes().decodeToString()
                files[fileName] = content
            }
            is PartData.FormItem -> {
                // 폼 데이터 처리
            }
            else -> {}
        }
        part.dispose()
    }
    
    problemService.addFiles(problemId, files)
    call.respond(HttpStatusCode.OK, mapOf("uploaded" to files.keys))
}
```

### 5.4 스트리밍 응답

```kotlin
get("/api/submissions/{id}/logs") {
    val submissionId = call.parameters["id"]
        ?: throw IllegalArgumentException("제출 ID가 필요합니다")
    
    call.respondTextWriter(contentType = ContentType.Text.Plain) {
        // 실시간 로그 스트리밍
        submissionService.streamLogs(submissionId).collect { log ->
            write(log)
            write("\n")
            flush()
        }
    }
}
```


---

## 6. 에러 처리와 검증

### 6.1 커스텀 예외 클래스

```kotlin
sealed class AppException(message: String) : Exception(message)

class ValidationException(message: String) : AppException(message)
class NotFoundException(message: String) : AppException(message)
class UnauthorizedException(message: String) : AppException(message)
class ForbiddenException(message: String) : AppException(message)
class ConflictException(message: String) : AppException(message)

fun Application.configureStatusPages() {
    install(StatusPages) {
        exception<ValidationException> { call, cause ->
            call.respond(HttpStatusCode.BadRequest, mapOf("error" to cause.message))
        }
        
        exception<NotFoundException> { call, cause ->
            call.respond(HttpStatusCode.NotFound, mapOf("error" to cause.message))
        }
        
        exception<UnauthorizedException> { call, cause ->
            call.respond(HttpStatusCode.Unauthorized, mapOf("error" to cause.message))
        }
        
        exception<ForbiddenException> { call, cause ->
            call.respond(HttpStatusCode.Forbidden, mapOf("error" to cause.message))
        }
        
        exception<ConflictException> { call, cause ->
            call.respond(HttpStatusCode.Conflict, mapOf("error" to cause.message))
        }
    }
}
```

**사용 예시**:
```kotlin
suspend fun findById(id: UUID): Problem {
    return problemService.findById(id)
        ?: throw NotFoundException("문제를 찾을 수 없습니다: $id")
}

suspend fun register(email: String, password: String): User {
    val existing = findByEmail(email)
    if (existing != null) {
        throw ConflictException("이미 사용 중인 이메일입니다: $email")
    }
    // ...
}
```

### 6.2 검증 유틸리티

```kotlin
object Validators {
    fun validateEmail(email: String) {
        require(email.matches(Regex("^[A-Za-z0-9+_.-]+@(.+)$"))) {
            "유효하지 않은 이메일 형식입니다: $email"
        }
    }
    
    fun validatePassword(password: String) {
        require(password.length >= 8) {
            "비밀번호는 최소 8자 이상이어야 합니다"
        }
        require(password.any { it.isUpperCase() }) {
            "비밀번호는 대문자를 포함해야 합니다"
        }
        require(password.any { it.isLowerCase() }) {
            "비밀번호는 소문자를 포함해야 합니다"
        }
        require(password.any { it.isDigit() }) {
            "비밀번호는 숫자를 포함해야 합니다"
        }
    }
    
    fun validateUUID(id: String): UUID {
        return try {
            UUID.fromString(id)
        } catch (e: IllegalArgumentException) {
            throw ValidationException("유효하지 않은 UUID 형식입니다: $id")
        }
    }
}
```

### 6.3 요청 검증 인터셉터

```kotlin
class RequestValidation(config: Configuration) {
    private val validator = config.validator

    class Configuration {
        var validator: (ApplicationCall) -> Unit = {}
    }

    companion object Plugin : RouteScopedPlugin<Configuration, RequestValidation> {
        override val key = AttributeKey<RequestValidation>("RequestValidation")

        override fun install(
            pipeline: ApplicationCallPipeline,
            configure: Configuration.() -> Unit
        ): RequestValidation {
            val config = Configuration().apply(configure)
            val plugin = RequestValidation(config)

            pipeline.intercept(ApplicationCallPipeline.Plugins) {
                try {
                    plugin.validator(call)
                } catch (e: Exception) {
                    call.respond(
                        HttpStatusCode.BadRequest,
                        mapOf("error" to (e.message ?: "검증 실패"))
                    )
                    finish()
                }
            }

            return plugin
        }
    }
}

// 사용
route("/api/problems") {
    install(RequestValidation) {
        validator = { call ->
            val contentType = call.request.contentType()
            require(contentType.match(ContentType.Application.Json)) {
                "Content-Type은 application/json이어야 합니다"
            }
        }
    }
    
    post {
        // 검증 통과 후 실행
    }
}
```

---

## 7. 테스트

### 7.1 기본 테스트 설정

```kotlin
class AuthRoutesTest {
    @Test
    fun `회원가입 성공`() = testApplication {
        // 테스트 환경 설정
        environment {
            config = MapApplicationConfig(
                "jwt.secret" to "test-secret",
                "database.jdbcUrl" to "jdbc:h2:mem:test"
            )
        }
        
        application {
            module()
        }
        
        // 요청 테스트
        val response = client.post("/api/auth/register") {
            contentType(ContentType.Application.Json)
            setBody("""
                {
                    "email": "test@example.com",
                    "password": "Password123",
                    "name": "테스트",
                    "verificationCode": "123456"
                }
            """.trimIndent())
        }
        
        // 검증
        assertEquals(HttpStatusCode.Created, response.status)
        val body = response.body<AuthResponse>()
        assertNotNull(body.token)
        assertEquals("test@example.com", body.user.email)
    }
    
    @Test
    fun `이메일 중복 시 실패`() = testApplication {
        application {
            module()
        }
        
        // 첫 번째 회원가입
        client.post("/api/auth/register") {
            contentType(ContentType.Application.Json)
            setBody("""
                {
                    "email": "test@example.com",
                    "password": "Password123",
                    "name": "테스트",
                    "verificationCode": "123456"
                }
            """.trimIndent())
        }
        
        // 중복 회원가입 시도
        val response = client.post("/api/auth/register") {
            contentType(ContentType.Application.Json)
            setBody("""
                {
                    "email": "test@example.com",
                    "password": "Password123",
                    "name": "테스트2",
                    "verificationCode": "123456"
                }
            """.trimIndent())
        }
        
        assertEquals(HttpStatusCode.BadRequest, response.status)
    }
}
```

### 7.2 인증이 필요한 엔드포인트 테스트

```kotlin
@Test
fun `프로필 조회 - 인증 성공`() = testApplication {
    application {
        module()
    }
    
    // 1. 로그인하여 토큰 획득
    val loginResponse = client.post("/api/auth/login") {
        contentType(ContentType.Application.Json)
        setBody("""
            {
                "email": "test@example.com",
                "password": "Password123"
            }
        """.trimIndent())
    }
    val authResponse = loginResponse.body<AuthResponse>()
    val token = authResponse.token
    
    // 2. 토큰으로 프로필 조회
    val profileResponse = client.get("/api/profile") {
        header(HttpHeaders.Authorization, "Bearer $token")
    }
    
    assertEquals(HttpStatusCode.OK, profileResponse.status)
    val user = profileResponse.body<User>()
    assertEquals("test@example.com", user.email)
}

@Test
fun `프로필 조회 - 인증 실패`() = testApplication {
    application {
        module()
    }
    
    val response = client.get("/api/profile")
    
    assertEquals(HttpStatusCode.Unauthorized, response.status)
}
```

### 7.3 Mock을 사용한 단위 테스트

```kotlin
class ProblemServiceTest {
    private val databaseFactory = mockk<DatabaseFactory>()
    private val problemService = ProblemService(databaseFactory)
    
    @Test
    fun `문제 조회 성공`() = runBlocking {
        val problemId = UUID.randomUUID()
        val expectedProblem = Problem(
            id = problemId,
            title = "테스트 문제",
            description = "설명",
            difficulty = Difficulty.EASY,
            language = Language.KOTLIN,
            testFiles = emptyMap()
        )
        
        coEvery {
            databaseFactory.dbQuery<Problem?>(any())
        } returns expectedProblem
        
        val result = problemService.findById(problemId)
        
        assertEquals(expectedProblem, result)
        coVerify { databaseFactory.dbQuery<Problem?>(any()) }
    }
}
```

### 7.4 통합 테스트

```kotlin
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class SubmissionIntegrationTest {
    private lateinit var testServer: TestApplicationEngine
    private lateinit var services: ServiceRegistry
    
    @BeforeAll
    fun setup() {
        testServer = TestApplicationEngine(createTestEnvironment {
            config = MapApplicationConfig(
                "database.jdbcUrl" to "jdbc:h2:mem:test",
                "jwt.secret" to "test-secret"
            )
        })
        
        testServer.start()
        testServer.application.module()
        services = testServer.application.attributes[ServiceRegistryKey]
    }
    
    @AfterAll
    fun teardown() {
        runBlocking { services.shutdown() }
        testServer.stop(0, 0)
    }
    
    @Test
    fun `코드 제출 및 평가 전체 플로우`() = runBlocking {
        // 1. 사용자 생성 및 로그인
        val token = createUserAndLogin()
        
        // 2. 문제 생성
        val problem = createProblem()
        
        // 3. 코드 제출
        val submission = submitCode(token, problem.id)
        
        // 4. 평가 완료 대기
        delay(5000)
        
        // 5. 결과 확인
        val result = getSubmission(token, submission.id)
        assertEquals(ExecutionStatus.SUCCESS, result.status)
        assertTrue(result.score > 0)
    }
}
```

---

## 8. 성능 최적화

### 8.1 연결 풀 설정

```kotlin
private fun createDataSource(): HikariDataSource {
    val hikariConfig = HikariConfig().apply {
        driverClassName = config.driver
        jdbcUrl = config.jdbcUrl
        username = config.username
        password = config.password
        
        // 연결 풀 설정
        maximumPoolSize = 20
        minimumIdle = 5
        connectionTimeout = 30000
        idleTimeout = 600000
        maxLifetime = 1800000
        
        // 성능 최적화
        isAutoCommit = false
        transactionIsolation = "TRANSACTION_REPEATABLE_READ"
        
        // 연결 검증
        connectionTestQuery = "SELECT 1"
        validationTimeout = 5000
    }
    return HikariDataSource(hikariConfig)
}
```

### 8.2 캐싱

```kotlin
class CachedProblemService(
    private val problemService: ProblemService,
    private val redisConnection: StatefulRedisConnection<String, String>
) {
    private val commands = redisConnection.sync()
    private val cacheExpiration = 3600L  // 1시간
    
    suspend fun findById(id: UUID): Problem? {
        val cacheKey = "problem:$id"
        
        // 캐시 확인
        commands.get(cacheKey)?.let { cached ->
            return Json.decodeFromString<Problem>(cached)
        }
        
        // 캐시 미스 - DB 조회
        val problem = problemService.findById(id) ?: return null
        
        // 캐시 저장
        commands.setex(
            cacheKey,
            cacheExpiration,
            Json.encodeToString(problem)
        )
        
        return problem
    }
    
    suspend fun invalidate(id: UUID) {
        commands.del("problem:$id")
    }
}
```

### 8.3 압축

```kotlin
fun Application.configureSerialization() {
    install(ContentNegotiation) {
        json()
    }
    
    install(Compression) {
        gzip {
            priority = 1.0
            minimumSize(1024)  // 1KB 이상만 압축
        }
        deflate {
            priority = 10.0
            minimumSize(1024)
        }
    }
}
```

---

## 9. 보안 Best Practices

### 9.1 Rate Limiting

```kotlin
class RateLimitPlugin(config: Configuration) {
    private val limiter = config.limiter

    class Configuration {
        var limiter: RateLimiter = RateLimiter(100, Duration.minutes(1))
    }

    companion object Plugin : RouteScopedPlugin<Configuration, RateLimitPlugin> {
        override val key = AttributeKey<RateLimitPlugin>("RateLimit")

        override fun install(
            pipeline: ApplicationCallPipeline,
            configure: Configuration.() -> Unit
        ): RateLimitPlugin {
            val config = Configuration().apply(configure)
            val plugin = RateLimitPlugin(config)

            pipeline.intercept(ApplicationCallPipeline.Plugins) {
                val clientId = call.request.header("X-Forwarded-For")
                    ?: call.request.local.remoteHost

                if (!plugin.limiter.tryAcquire(clientId)) {
                    call.respond(
                        HttpStatusCode.TooManyRequests,
                        mapOf("error" to "요청 한도를 초과했습니다")
                    )
                    finish()
                }
            }

            return plugin
        }
    }
}

// 사용
route("/api") {
    install(RateLimitPlugin) {
        limiter = RateLimiter(100, Duration.minutes(1))
    }
}
```

### 9.2 입력 검증 및 새니타이징

```kotlin
object SecurityUtils {
    fun sanitizeInput(input: String): String {
        return input
            .replace("<", "&lt;")
            .replace(">", "&gt;")
            .replace("\"", "&quot;")
            .replace("'", "&#x27;")
            .replace("/", "&#x2F;")
    }
    
    fun validateFileContent(content: String) {
        require(content.length <= 1_000_000) {
            "파일 크기는 1MB를 초과할 수 없습니다"
        }
        
        val dangerousPatterns = listOf(
            "eval\\(",
            "exec\\(",
            "system\\(",
            "__import__"
        )
        
        dangerousPatterns.forEach { pattern ->
            require(!content.contains(Regex(pattern))) {
                "위험한 코드 패턴이 감지되었습니다"
            }
        }
    }
}
```

---

## 10. 요약

### Ktor의 핵심 개념
1. **플러그인 시스템**: 필요한 기능만 추가
2. **DSL 스타일**: Kotlin의 강력한 표현력 활용
3. **코루틴 네이티브**: 비동기 처리가 기본
4. **경량**: Spring Boot보다 빠른 시작 시간

### 프로젝트에서 배울 수 있는 패턴
- `Application.module()`: 애플리케이션 설정
- `Route.확장함수`: 라우트 모듈화
- `authenticate()`: JWT 인증
- `StatusPages`: 통합 에러 처리
- `ServiceRegistry`: 의존성 관리

### 추가 학습 자료
- [Ktor 공식 문서](https://ktor.io/docs/)
- [Ktor 샘플 프로젝트](https://github.com/ktorio/ktor-samples)
- [Ktor 플러그인 목록](https://ktor.io/docs/plugins.html)

이 가이드를 통해 Ktor의 고급 기능을 마스터하고, 
프로덕션 수준의 백엔드 API를 개발할 수 있기를 바랍니다!
