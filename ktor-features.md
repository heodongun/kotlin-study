# Ktor 주요 기능 학습 가이드

이 문서는 프로젝트에서 사용된 Ktor의 주요 기능들을 정리합니다.

## 1. 라우팅 DSL

### 개념
Ktor는 DSL 스타일로 라우트를 정의할 수 있습니다. 선언적이고 읽기 쉬운 코드를 작성할 수 있습니다.

### 프로젝트 예시

```kotlin
// BoardController.kt
fun Route.boardRoutes() {
    route("/api/boards") {
        post {
            val request = call.receive<CreateBoardRequest>()
            // 처리 로직
        }
        
        get {
            val page = call.request.queryParameters["page"]?.toIntOrNull() ?: 1
            // 처리 로직
        }
        
        get("/{id}") {
            val id = call.parameters["id"]?.toLongOrNull()
            // 처리 로직
        }
        
        put("/{id}") {
            // 수정 로직
        }
        
        delete("/{id}") {
            // 삭제 로직
        }
    }
}
```

### 주요 특징

#### 1. 확장 함수 활용
```kotlin
fun Route.boardRoutes() {
    // Route를 확장하여 라우트 정의
}
```

#### 2. 중첩 라우트
```kotlin
route("/api/boards") {
    // /api/boards
    get { }
    
    // /api/boards/{id}
    get("/{id}") { }
}
```

#### 3. HTTP 메서드
- `get { }`: GET 요청
- `post { }`: POST 요청
- `put { }`: PUT 요청
- `delete { }`: DELETE 요청
- `patch { }`: PATCH 요청

#### 4. 파라미터 접근
```kotlin
// Path 파라미터
val id = call.parameters["id"]

// Query 파라미터
val page = call.request.queryParameters["page"]
```

### 왜 이렇게 작성했나?
- **가독성**: 라우트 구조가 명확하게 보임
- **타입 안전성**: 컴파일 타임에 오류 검출
- **모듈화**: 확장 함수로 라우트를 분리하여 관리

## 2. Content Negotiation

### 개념
클라이언트와 서버 간의 데이터 형식을 자동으로 변환합니다.

### 프로젝트 예시

```kotlin
// PluginsConfig.kt
install(ContentNegotiation) {
    json(Json {
        prettyPrint = true
        isLenient = true
        ignoreUnknownKeys = true
    })
}
```

### 설정 옵션
- `prettyPrint`: JSON을 보기 좋게 포맷팅
- `isLenient`: 느슨한 JSON 파싱 (따옴표 없는 키 허용 등)
- `ignoreUnknownKeys`: 알 수 없는 필드 무시

### 사용 예시

```kotlin
// 요청 받기
val request = call.receive<CreateBoardRequest>()

// 응답 보내기
call.respond(HttpStatusCode.Created, response)
```

### 왜 이렇게 작성했나?
- **자동 직렬화**: 수동 JSON 파싱 불필요
- **타입 안전성**: Kotlin 객체로 직접 변환
- **kotlinx.serialization**: Kotlin 네이티브 직렬화 라이브러리 사용

## 3. StatusPages (에러 핸들링)

### 개념
전역 예외 처리를 설정하여 일관된 에러 응답을 제공합니다.

### 프로젝트 예시

```kotlin
// PluginsConfig.kt
install(StatusPages) {
    exception<Throwable> { call, cause ->
        call.application.environment.log.error("Unhandled exception", cause)
        call.respond(
            HttpStatusCode.InternalServerError,
            ErrorResponse("서버 오류가 발생했습니다", "INTERNAL_ERROR")
        )
    }
}
```

### 주요 기능

#### 1. 예외 타입별 처리
```kotlin
exception<IllegalArgumentException> { call, cause ->
    call.respond(HttpStatusCode.BadRequest, cause.message)
}

exception<NotFoundException> { call, cause ->
    call.respond(HttpStatusCode.NotFound, cause.message)
}
```

#### 2. 상태 코드별 처리
```kotlin
status(HttpStatusCode.NotFound) { call, status ->
    call.respond(TextContent("404: Page Not Found", ContentType.Text.Plain))
}
```

### 왜 이렇게 작성했나?
- **중앙 집중식 에러 처리**: 모든 예외를 한 곳에서 처리
- **일관성**: 모든 에러 응답이 동일한 형식
- **로깅**: 예외 발생 시 자동으로 로그 기록

## 4. CallLogging (요청/응답 로깅)

### 개념
HTTP 요청과 응답을 자동으로 로깅합니다.

### 프로젝트 예시

```kotlin
// PluginsConfig.kt
install(CallLogging) {
    level = Level.INFO
    filter { call -> call.request.path().startsWith("/api") }
    format { call ->
        val status = call.response.status()
        val httpMethod = call.request.httpMethod.value
        val path = call.request.path()
        "$httpMethod $path - $status"
    }
}
```

### 설정 옵션
- `level`: 로그 레벨 (TRACE, DEBUG, INFO, WARN, ERROR)
- `filter`: 특정 요청만 로깅
- `format`: 로그 메시지 형식 커스터마이징

### 출력 예시
```
INFO  - GET /api/boards - 200 OK
INFO  - POST /api/boards - 201 Created
INFO  - GET /api/boards/1 - 404 Not Found
```

### 왜 이렇게 작성했나?
- **디버깅**: 요청/응답 추적 용이
- **모니터링**: API 사용 패턴 파악
- **필터링**: API 엔드포인트만 로깅하여 노이즈 감소

## 5. CORS (Cross-Origin Resource Sharing)

### 개념
다른 도메인에서의 API 접근을 허용합니다.

### 프로젝트 예시

```kotlin
// PluginsConfig.kt
install(CORS) {
    allowMethod(HttpMethod.Options)
    allowMethod(HttpMethod.Get)
    allowMethod(HttpMethod.Post)
    allowMethod(HttpMethod.Put)
    allowMethod(HttpMethod.Delete)
    allowHeader(HttpHeaders.ContentType)
    anyHost()
}
```

### 설정 옵션
- `allowMethod()`: 허용할 HTTP 메서드
- `allowHeader()`: 허용할 헤더
- `anyHost()`: 모든 호스트 허용 (개발 환경용)
- `allowHost()`: 특정 호스트만 허용 (프로덕션 권장)

### 프로덕션 설정 예시
```kotlin
install(CORS) {
    allowHost("example.com", schemes = listOf("https"))
    allowHeader(HttpHeaders.ContentType)
    allowMethod(HttpMethod.Get)
    allowMethod(HttpMethod.Post)
}
```

### 왜 이렇게 작성했나?
- **프론트엔드 통합**: 다른 도메인의 웹 앱에서 API 호출 가능
- **보안**: 허용할 메서드와 헤더를 명시적으로 지정

## 6. 플러그인 시스템

### 개념
Ktor는 플러그인 기반 아키텍처로 기능을 확장합니다.

### 플러그인 설치 패턴

```kotlin
fun Application.configurePlugins() {
    install(ContentNegotiation) { /* 설정 */ }
    install(CallLogging) { /* 설정 */ }
    install(StatusPages) { /* 설정 */ }
    install(CORS) { /* 설정 */ }
}
```

### 커스텀 플러그인 예시

```kotlin
val CustomPlugin = createApplicationPlugin(name = "CustomPlugin") {
    onCall { call ->
        // 요청 전 처리
    }
    
    onCallRespond { call ->
        // 응답 후 처리
    }
}
```

### 왜 이렇게 작성했나?
- **모듈화**: 기능별로 플러그인 분리
- **재사용성**: 플러그인을 여러 프로젝트에서 사용 가능
- **확장성**: 필요한 기능만 설치

## 7. Application과 Routing

### Application 설정

```kotlin
// Application.kt
fun Application.module() {
    // Koin 초기화
    install(Koin) {
        slf4jLogger()
        modules(appModule)
    }
    
    // Database 초기화
    DatabaseConfig.init(getDatabaseConfig())
    
    // 플러그인 설정
    configurePlugins()
    
    // 라우팅 설정
    routing {
        val boardController by inject<BoardController>()
        boardController.apply { boardRoutes() }
    }
}
```

### 왜 이렇게 작성했나?
- **확장 함수**: Application을 확장하여 모듈 정의
- **의존성 주입**: Koin으로 컨트롤러 주입
- **분리**: 설정, 플러그인, 라우팅을 명확히 분리

## 8. 코루틴 통합

### 개념
Ktor는 Kotlin 코루틴을 기본으로 지원합니다.

### 프로젝트 예시

```kotlin
// 모든 라우트 핸들러는 자동으로 코루틴 컨텍스트에서 실행
get("/{id}") {
    // suspend 함수 호출 가능
    val result = getBoardUseCase.execute(id)
    call.respond(result)
}
```

### 왜 이렇게 작성했나?
- **비동기 처리**: 블로킹 없이 많은 요청 처리
- **간결함**: async/await 없이 순차적 코드 작성
- **성능**: 적은 스레드로 많은 동시 요청 처리

## 핵심 학습 포인트

1. **DSL 스타일**: 선언적이고 읽기 쉬운 라우팅
2. **플러그인 시스템**: 모듈화된 기능 확장
3. **코루틴 통합**: 비동기 처리의 간결함
4. **타입 안전성**: Content Negotiation으로 자동 직렬화
5. **중앙 집중식 설정**: 플러그인으로 공통 기능 관리

Ktor는 Kotlin의 특징을 최대한 활용하여 간결하고 강력한 웹 애플리케이션을 만들 수 있게 해줍니다.
