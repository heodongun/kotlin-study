# Kotlin 코루틴 심화 가이드

이 문서는 Coding Platform 프로젝트에서 사용되는 Kotlin 코루틴의 고급 개념과 실전 패턴을 다룹니다.

## 목차
1. [코루틴 기본 개념](#1-코루틴-기본-개념)
2. [Dispatchers 상세](#2-dispatchers-상세)
3. [구조화된 동시성](#3-구조화된-동시성)
4. [코루틴 스코프](#4-코루틴-스코프)
5. [에러 처리](#5-에러-처리)
6. [실전 패턴](#6-실전-패턴)

---

## 1. 코루틴 기본 개념

### 1.1 코루틴이란?
코루틴(Coroutine)은 **일시 중단 가능한 계산**입니다.
- 스레드보다 훨씬 가볍습니다 (수천 개 동시 실행 가능)
- 메모리 오버헤드가 적습니다
- 컨텍스트 스위칭 비용이 거의 없습니다

### 1.2 suspend 함수의 동작 원리

```kotlin
suspend fun fetchUser(id: String): User {
    delay(1000)  // 1초 대기 (스레드 블로킹 없음!)
    return User(id, "홍길동")
}
```

**내부 동작**:
1. `delay(1000)` 호출 시 코루틴이 **일시 중단**됩니다
2. 스레드는 다른 작업을 수행할 수 있습니다
3. 1초 후 코루틴이 **재개**됩니다
4. 나머지 코드가 실행됩니다

**vs Thread.sleep()**:
```kotlin
// ❌ 스레드 블로킹 (나쁜 예)
fun fetchUser(id: String): User {
    Thread.sleep(1000)  // 스레드가 1초간 아무것도 못 함
    return User(id, "홍길동")
}

// ✅ 코루틴 일시 중단 (좋은 예)
suspend fun fetchUser(id: String): User {
    delay(1000)  // 스레드는 다른 작업 가능
    return User(id, "홍길동")
}
```

### 1.3 코루틴 빌더

**launch**: 결과를 반환하지 않는 코루틴 시작
```kotlin
GlobalScope.launch {
    println("코루틴 시작")
    delay(1000)
    println("1초 후")
}
```

**async**: 결과를 반환하는 코루틴 시작
```kotlin
val deferred = GlobalScope.async {
    delay(1000)
    "결과"
}
val result = deferred.await()  // 결과 대기
```

**runBlocking**: 코루틴을 블로킹 방식으로 실행
```kotlin
fun main() = runBlocking {
    delay(1000)
    println("1초 후")
}
```

---

## 2. Dispatchers 상세

### 2.1 Dispatcher 종류와 사용 시나리오

**Dispatchers.Default**
- CPU 집약적 작업에 사용
- 스레드 풀 크기: CPU 코어 수
- 예: 정렬, 계산, 데이터 변환

```kotlin
suspend fun processLargeData(data: List<Int>): List<Int> =
    withContext(Dispatchers.Default) {
        data.map { it * 2 }  // CPU 집약적 작업
            .sorted()
    }
```

**Dispatchers.IO**
- I/O 작업에 사용
- 스레드 풀 크기: 64개 (또는 코어 수 중 큰 값)
- 예: 파일 읽기/쓰기, 네트워크 요청, DB 쿼리

```kotlin
suspend fun readFile(path: String): String =
    withContext(Dispatchers.IO) {
        File(path).readText()
    }

suspend fun queryDatabase(): List<User> =
    withContext(Dispatchers.IO) {
        transaction {
            Users.selectAll().map { toUser(it) }
        }
    }
```

**프로젝트 예시**: `DatabaseFactory.kt`
```kotlin
suspend fun <T> dbQuery(block: Transaction.() -> T): T =
    withContext(Dispatchers.IO) {  // DB 작업은 IO Dispatcher
        transaction(database) {
            block(this)
        }
    }
```

**Dispatchers.Main**
- UI 스레드 (Android/Desktop)
- 백엔드에서는 사용하지 않음

**Dispatchers.Unconfined**
- 특정 스레드에 제한되지 않음
- 일반적으로 사용하지 않음 (디버깅 용도)


### 2.2 Dispatcher 전환

```kotlin
suspend fun complexOperation() {
    // 기본 Dispatcher에서 시작
    println("Thread: ${Thread.currentThread().name}")
    
    // IO Dispatcher로 전환
    withContext(Dispatchers.IO) {
        println("Thread: ${Thread.currentThread().name}")
        // 파일 읽기
    }
    
    // Default Dispatcher로 전환
    withContext(Dispatchers.Default) {
        println("Thread: ${Thread.currentThread().name}")
        // 데이터 처리
    }
    
    // 원래 Dispatcher로 자동 복귀
    println("Thread: ${Thread.currentThread().name}")
}
```

### 2.3 프로젝트 실전 예시

**DockerExecutorService.kt**:
```kotlin
class DockerExecutorService(
    private val dockerManager: DockerManager,
    private val testRunnerService: TestRunnerService
) {
    // Docker 작업은 I/O 작업이므로 Dispatchers.IO 사용
    suspend fun executeCode(
        language: Language, 
        files: Map<String, String>, 
        command: String
    ): ExecutionResult = withContext(Dispatchers.IO) {
        dockerManager.executeCode(
            executionId = UUID.randomUUID().toString(),
            language = language,
            files = files,
            command = listOf("sh", "-c", command)
        )
    }

    suspend fun evaluateSubmission(
        submission: Submission, 
        problem: Problem
    ): SubmissionFeedback = withContext(Dispatchers.IO) {
        // 1. 테스트 실행 (I/O 작업)
        val (executionResult, testResults) = runTests(submission, problem)
        
        // 2. 점수 계산 (CPU 작업이지만 간단하므로 같은 Dispatcher 사용)
        val passRate = if (testResults.total == 0) 0.0 
                       else testResults.passed.toDouble() / testResults.total
        val score = (passRate * 100).roundToInt()
        
        // 3. 피드백 생성
        SubmissionFeedback(/* ... */)
    }
}
```

---

## 3. 구조화된 동시성

### 3.1 개념
구조화된 동시성(Structured Concurrency)은 코루틴의 생명주기를 명확하게 관리하는 원칙입니다.

**핵심 원칙**:
1. 부모 코루틴이 취소되면 자식 코루틴도 취소됩니다
2. 부모 코루틴은 모든 자식 코루틴이 완료될 때까지 대기합니다
3. 자식 코루틴에서 예외가 발생하면 부모에게 전파됩니다

### 3.2 coroutineScope

```kotlin
suspend fun fetchUserData(userId: String): UserData = coroutineScope {
    // 이 블록 안의 모든 코루틴이 완료될 때까지 대기
    val userDeferred = async { fetchUser(userId) }
    val ordersDeferred = async { fetchOrders(userId) }
    val profileDeferred = async { fetchProfile(userId) }
    
    UserData(
        user = userDeferred.await(),
        orders = ordersDeferred.await(),
        profile = profileDeferred.await()
    )
}
```

**특징**:
- 모든 자식 코루틴이 완료될 때까지 함수가 반환되지 않습니다
- 하나의 자식 코루틴이 실패하면 다른 모든 자식도 취소됩니다

### 3.3 supervisorScope

```kotlin
suspend fun fetchDataWithFallback(userId: String): UserData = supervisorScope {
    val userDeferred = async { fetchUser(userId) }
    val ordersDeferred = async { 
        try {
            fetchOrders(userId)
        } catch (e: Exception) {
            emptyList()  // 실패 시 빈 리스트
        }
    }
    
    UserData(
        user = userDeferred.await(),
        orders = ordersDeferred.await()
    )
}
```

**특징**:
- 하나의 자식 코루틴이 실패해도 다른 자식은 계속 실행됩니다
- 독립적인 작업을 병렬로 실행할 때 유용합니다

### 3.4 실전 예시: 여러 제출 동시 평가

```kotlin
suspend fun evaluateMultipleSubmissions(
    submissionIds: List<String>
): List<Result<SubmissionFeedback>> = coroutineScope {
    submissionIds.map { id ->
        async {
            try {
                val submission = findById(id) ?: error("제출을 찾을 수 없습니다")
                val problem = problemService.findById(submission.problemId) 
                    ?: error("문제를 찾을 수 없습니다")
                Result.success(dockerExecutorService.evaluateSubmission(submission, problem))
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }.awaitAll()
}
```

---

## 4. 코루틴 스코프

### 4.1 GlobalScope (사용 지양)

```kotlin
// ❌ 나쁜 예: 생명주기 관리 안 됨
GlobalScope.launch {
    // 애플리케이션이 종료되어도 계속 실행될 수 있음
    delay(10000)
    println("10초 후")
}
```

### 4.2 CoroutineScope 생성

```kotlin
class MyService {
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    fun startBackgroundTask() {
        scope.launch {
            // 백그라운드 작업
        }
    }
    
    fun cleanup() {
        scope.cancel()  // 모든 코루틴 취소
    }
}
```

### 4.3 프로젝트 예시: SubmissionService

```kotlin
class SubmissionService(
    private val databaseFactory: DatabaseFactory,
    private val problemService: ProblemService,
    private val dockerExecutorService: DockerExecutorService
) {
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    suspend fun createSubmission(
        userId: UUID,
        problemId: UUID,
        files: Map<String, String>
    ): Submission {
        // 1. 제출 레코드 생성
        val submission = databaseFactory.dbQuery {
            // DB에 저장
        }

        // 2. 백그라운드에서 비동기 평가
        scope.launch {
            evaluateSubmission(submission.id)
        }

        return submission
    }
    
    private suspend fun evaluateSubmission(submissionId: String) {
        try {
            val submission = findById(submissionId) ?: return
            val problem = problemService.findById(submission.problemId) ?: return
            
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
        }
    }
    
    fun shutdown() {
        scope.cancel()
    }
}
```


---

## 5. 에러 처리

### 5.1 try-catch

```kotlin
suspend fun fetchUser(id: String): User? {
    return try {
        databaseFactory.dbQuery {
            Users.select { Users.id eq UUID.fromString(id) }
                .firstOrNull()
                ?.let { toUser(it) }
        }
    } catch (e: Exception) {
        logger.error("사용자 조회 실패: $id", e)
        null
    }
}
```

### 5.2 CoroutineExceptionHandler

```kotlin
val exceptionHandler = CoroutineExceptionHandler { _, exception ->
    logger.error("코루틴 예외 발생", exception)
}

val scope = CoroutineScope(Dispatchers.Default + exceptionHandler)

scope.launch {
    throw RuntimeException("에러!")  // exceptionHandler가 처리
}
```

### 5.3 SupervisorJob으로 격리

```kotlin
suspend fun processMultipleItems(items: List<String>) = supervisorScope {
    items.forEach { item ->
        launch {
            try {
                processItem(item)
            } catch (e: Exception) {
                logger.error("아이템 처리 실패: $item", e)
                // 다른 아이템 처리는 계속됨
            }
        }
    }
}
```

### 5.4 실전 예시: 제출 평가 에러 처리

```kotlin
private suspend fun evaluateSubmission(submissionId: String) {
    try {
        // 1. 데이터 조회
        val submission = findById(submissionId) 
            ?: throw IllegalArgumentException("제출을 찾을 수 없습니다: $submissionId")
        val problem = problemService.findById(submission.problemId) 
            ?: throw IllegalArgumentException("문제를 찾을 수 없습니다: ${submission.problemId}")

        // 2. 평가 실행
        val feedback = try {
            dockerExecutorService.evaluateSubmission(submission, problem)
        } catch (e: Exception) {
            logger.error("Docker 실행 실패: $submissionId", e)
            SubmissionFeedback(
                totalTests = 0,
                passedTests = 0,
                failedTests = 0,
                passRate = 0.0,
                score = 0,
                status = ExecutionStatus.ERROR,
                testResults = TestResults(0, 0, 0, emptyList()),
                output = "",
                message = "실행 중 오류가 발생했습니다: ${e.message}"
            )
        }

        // 3. 결과 저장
        databaseFactory.dbQuery {
            Submissions.update({ Submissions.id eq submissionId }) {
                it[status] = feedback.status.name
                it[score] = feedback.score
                it[this.feedback] = Json.encodeToString(feedback)
            }
        }
    } catch (e: Exception) {
        logger.error("평가 프로세스 실패: $submissionId", e)
        // 최종 에러 상태로 업데이트
        try {
            databaseFactory.dbQuery {
                Submissions.update({ Submissions.id eq submissionId }) {
                    it[status] = ExecutionStatus.ERROR.name
                }
            }
        } catch (dbError: Exception) {
            logger.error("DB 업데이트 실패: $submissionId", dbError)
        }
    }
}
```

---

## 6. 실전 패턴

### 6.1 타임아웃 설정

```kotlin
suspend fun fetchWithTimeout(url: String): String {
    return withTimeout(5000) {  // 5초 타임아웃
        httpClient.get(url)
    }
}

// 타임아웃 시 null 반환
suspend fun fetchWithTimeoutOrNull(url: String): String? {
    return withTimeoutOrNull(5000) {
        httpClient.get(url)
    }
}
```

**프로젝트 예시**: Docker 컨테이너 실행
```kotlin
private fun runContainer(
    container: CreateContainerResponse, 
    timeout: Long
): ContainerExecutionResult {
    val exitCode = try {
        dockerClient.waitContainerCmd(containerId)
            .exec(WaitContainerResultCallback())
            .awaitStatusCode(timeout, TimeUnit.SECONDS)  // 타임아웃 적용
    } catch (ex: Exception) {
        dockerClient.killContainerCmd(containerId).exec()  // 강제 종료
        -1
    }
    // ...
}
```

### 6.2 재시도 패턴

```kotlin
suspend fun <T> retryWithExponentialBackoff(
    times: Int = 3,
    initialDelay: Long = 100,
    maxDelay: Long = 1000,
    factor: Double = 2.0,
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(times - 1) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            logger.warn("시도 ${attempt + 1} 실패, ${currentDelay}ms 후 재시도", e)
        }
        delay(currentDelay)
        currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
    }
    return block()  // 마지막 시도
}

// 사용 예시
suspend fun fetchUserWithRetry(id: String): User {
    return retryWithExponentialBackoff(times = 3) {
        databaseFactory.dbQuery {
            Users.select { Users.id eq UUID.fromString(id) }
                .firstOrNull()
                ?.let { toUser(it) }
                ?: throw NoSuchElementException("사용자를 찾을 수 없습니다")
        }
    }
}
```

### 6.3 병렬 처리 패턴

**모든 작업 완료 대기**:
```kotlin
suspend fun processAllUsers(userIds: List<String>): List<User> = coroutineScope {
    userIds.map { id ->
        async { fetchUser(id) }
    }.awaitAll()
}
```

**첫 번째 완료된 작업 사용**:
```kotlin
suspend fun fetchFromMultipleSources(id: String): User = coroutineScope {
    select<User> {
        async { fetchFromDatabase(id) }.onAwait { it }
        async { fetchFromCache(id) }.onAwait { it }
        async { fetchFromAPI(id) }.onAwait { it }
    }
}
```

**배치 처리**:
```kotlin
suspend fun processBatch(items: List<String>, batchSize: Int = 10) {
    items.chunked(batchSize).forEach { batch ->
        coroutineScope {
            batch.forEach { item ->
                launch {
                    processItem(item)
                }
            }
        }
    }
}
```

### 6.4 캐싱 패턴

```kotlin
class CachedUserService(
    private val databaseFactory: DatabaseFactory
) {
    private val cache = ConcurrentHashMap<UUID, User>()
    private val mutex = Mutex()

    suspend fun getUser(id: UUID): User? {
        // 캐시 확인
        cache[id]?.let { return it }

        // 캐시 미스 - DB 조회
        return mutex.withLock {
            // Double-check locking
            cache[id]?.let { return it }

            val user = databaseFactory.dbQuery {
                Users.select { Users.id eq id }
                    .firstOrNull()
                    ?.let { toUser(it) }
            }

            user?.let { cache[id] = it }
            user
        }
    }

    fun invalidate(id: UUID) {
        cache.remove(id)
    }

    fun clear() {
        cache.clear()
    }
}
```

### 6.5 Rate Limiting 패턴

```kotlin
class RateLimiter(
    private val maxRequests: Int,
    private val timeWindow: Duration
) {
    private val semaphore = Semaphore(maxRequests)
    private val timestamps = mutableListOf<Long>()
    private val mutex = Mutex()

    suspend fun <T> execute(block: suspend () -> T): T {
        semaphore.acquire()
        try {
            mutex.withLock {
                val now = System.currentTimeMillis()
                timestamps.removeAll { now - it > timeWindow.inWholeMilliseconds }
                
                if (timestamps.size >= maxRequests) {
                    val oldestTimestamp = timestamps.first()
                    val waitTime = timeWindow.inWholeMilliseconds - (now - oldestTimestamp)
                    if (waitTime > 0) {
                        delay(waitTime)
                    }
                }
                
                timestamps.add(System.currentTimeMillis())
            }
            return block()
        } finally {
            semaphore.release()
        }
    }
}

// 사용 예시
val rateLimiter = RateLimiter(maxRequests = 10, timeWindow = Duration.seconds(1))

suspend fun callAPI(endpoint: String): String {
    return rateLimiter.execute {
        httpClient.get(endpoint)
    }
}
```


### 6.6 Flow - 비동기 스트림

Flow는 여러 값을 비동기로 반환하는 코루틴 기반 스트림입니다.

**기본 사용**:
```kotlin
fun getNumbers(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(100)
        emit(i)  // 값 방출
    }
}

// 수집
suspend fun collectNumbers() {
    getNumbers().collect { value ->
        println(value)
    }
}
```

**Flow 연산자**:
```kotlin
suspend fun processSubmissions() {
    getSubmissionFlow()
        .filter { it.status == ExecutionStatus.PENDING }  // 필터링
        .map { evaluateSubmission(it) }                   // 변환
        .onEach { saveFeedback(it) }                      // 부수 효과
        .catch { e -> logger.error("에러 발생", e) }      // 에러 처리
        .collect()                                         // 수집
}
```

**실전 예시: 실시간 제출 모니터링**:
```kotlin
class SubmissionMonitor(
    private val databaseFactory: DatabaseFactory
) {
    fun monitorSubmissions(): Flow<Submission> = flow {
        while (true) {
            val pendingSubmissions = databaseFactory.dbQuery {
                Submissions.select { Submissions.status eq ExecutionStatus.PENDING.name }
                    .map { toSubmission(it) }
            }
            
            pendingSubmissions.forEach { emit(it) }
            delay(5000)  // 5초마다 체크
        }
    }
}

// 사용
suspend fun startMonitoring() {
    submissionMonitor.monitorSubmissions()
        .collect { submission ->
            logger.info("대기 중인 제출: ${submission.id}")
            evaluateSubmission(submission.id)
        }
}
```

**StateFlow와 SharedFlow**:
```kotlin
class SubmissionStateManager {
    // StateFlow: 현재 상태를 유지하는 Hot Flow
    private val _submissionState = MutableStateFlow<SubmissionState>(SubmissionState.Idle)
    val submissionState: StateFlow<SubmissionState> = _submissionState.asStateFlow()

    // SharedFlow: 이벤트 브로드캐스트
    private val _submissionEvents = MutableSharedFlow<SubmissionEvent>()
    val submissionEvents: SharedFlow<SubmissionEvent> = _submissionEvents.asSharedFlow()

    suspend fun submitCode(code: String) {
        _submissionState.value = SubmissionState.Submitting
        _submissionEvents.emit(SubmissionEvent.Started)
        
        try {
            val result = evaluateCode(code)
            _submissionState.value = SubmissionState.Completed(result)
            _submissionEvents.emit(SubmissionEvent.Completed(result))
        } catch (e: Exception) {
            _submissionState.value = SubmissionState.Error(e.message)
            _submissionEvents.emit(SubmissionEvent.Failed(e))
        }
    }
}

sealed class SubmissionState {
    object Idle : SubmissionState()
    object Submitting : SubmissionState()
    data class Completed(val result: String) : SubmissionState()
    data class Error(val message: String?) : SubmissionState()
}

sealed class SubmissionEvent {
    object Started : SubmissionEvent()
    data class Completed(val result: String) : SubmissionEvent()
    data class Failed(val error: Exception) : SubmissionEvent()
}
```

---

## 7. 성능 최적화

### 7.1 불필요한 Dispatcher 전환 피하기

```kotlin
// ❌ 나쁜 예: 불필요한 전환
suspend fun processData(data: String): String {
    return withContext(Dispatchers.IO) {
        withContext(Dispatchers.Default) {  // 불필요한 전환
            data.uppercase()
        }
    }
}

// ✅ 좋은 예
suspend fun processData(data: String): String {
    return withContext(Dispatchers.Default) {
        data.uppercase()
    }
}
```

### 7.2 코루틴 재사용

```kotlin
// ❌ 나쁜 예: 매번 새로운 스코프 생성
class MyService {
    suspend fun doWork() {
        CoroutineScope(Dispatchers.Default).launch {
            // 작업
        }
    }
}

// ✅ 좋은 예: 스코프 재사용
class MyService {
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    fun doWork() {
        scope.launch {
            // 작업
        }
    }
    
    fun cleanup() {
        scope.cancel()
    }
}
```

### 7.3 병렬 처리 최적화

```kotlin
// ❌ 나쁜 예: 순차 처리
suspend fun fetchAllUsers(ids: List<String>): List<User> {
    val users = mutableListOf<User>()
    for (id in ids) {
        users.add(fetchUser(id))  // 하나씩 순차 처리
    }
    return users
}

// ✅ 좋은 예: 병렬 처리
suspend fun fetchAllUsers(ids: List<String>): List<User> = coroutineScope {
    ids.map { id ->
        async { fetchUser(id) }  // 모두 동시에 시작
    }.awaitAll()
}
```

### 7.4 메모리 효율적인 Flow 사용

```kotlin
// ❌ 나쁜 예: 모든 데이터를 메모리에 로드
suspend fun processAllSubmissions(): List<Result> {
    val submissions = databaseFactory.dbQuery {
        Submissions.selectAll().map { toSubmission(it) }  // 수만 개일 수 있음
    }
    return submissions.map { process(it) }
}

// ✅ 좋은 예: 스트리밍 처리
fun processAllSubmissions(): Flow<Result> = flow {
    var offset = 0
    val batchSize = 100
    
    while (true) {
        val batch = databaseFactory.dbQuery {
            Submissions.selectAll()
                .limit(batchSize, offset.toLong())
                .map { toSubmission(it) }
        }
        
        if (batch.isEmpty()) break
        
        batch.forEach { submission ->
            emit(process(submission))
        }
        
        offset += batchSize
    }
}
```

---

## 8. 디버깅 팁

### 8.1 코루틴 이름 지정

```kotlin
val scope = CoroutineScope(
    Dispatchers.Default + 
    SupervisorJob() + 
    CoroutineName("SubmissionProcessor")
)

scope.launch(CoroutineName("EvaluateSubmission-$submissionId")) {
    // 로그에서 코루틴 이름 확인 가능
}
```

### 8.2 로깅

```kotlin
suspend fun evaluateSubmission(id: String) {
    logger.info("평가 시작: $id, Thread: ${Thread.currentThread().name}")
    
    withContext(Dispatchers.IO) {
        logger.info("IO 작업 시작: $id, Thread: ${Thread.currentThread().name}")
        // 작업
    }
    
    logger.info("평가 완료: $id, Thread: ${Thread.currentThread().name}")
}
```

### 8.3 코루틴 덤프

```kotlin
// 실행 중인 모든 코루틴 정보 출력
fun dumpCoroutines() {
    println(DebugProbes.dumpCoroutinesInfo())
}
```

---

## 9. 자주 하는 실수

### 9.1 GlobalScope 사용

```kotlin
// ❌ 잘못된 예
GlobalScope.launch {
    // 생명주기 관리 안 됨
}

// ✅ 올바른 예
class MyService {
    private val scope = CoroutineScope(Dispatchers.Default + SupervisorJob())
    
    fun doWork() {
        scope.launch {
            // 생명주기 관리됨
        }
    }
}
```

### 9.2 블로킹 호출

```kotlin
// ❌ 잘못된 예
suspend fun fetchData(): String {
    return runBlocking {  // suspend 함수 안에서 runBlocking 사용 금지!
        httpClient.get("url")
    }
}

// ✅ 올바른 예
suspend fun fetchData(): String {
    return httpClient.get("url")
}
```

### 9.3 예외 무시

```kotlin
// ❌ 잘못된 예
scope.launch {
    throw Exception("에러!")  // 예외가 무시될 수 있음
}

// ✅ 올바른 예
scope.launch {
    try {
        // 작업
    } catch (e: Exception) {
        logger.error("에러 발생", e)
    }
}
```

### 9.4 Dispatcher 잘못 선택

```kotlin
// ❌ 잘못된 예: CPU 작업에 IO Dispatcher
suspend fun sortLargeList(list: List<Int>): List<Int> =
    withContext(Dispatchers.IO) {  // 잘못된 선택
        list.sorted()
    }

// ✅ 올바른 예
suspend fun sortLargeList(list: List<Int>): List<Int> =
    withContext(Dispatchers.Default) {  // CPU 작업은 Default
        list.sorted()
    }
```

---

## 10. 요약

### 핵심 원칙
1. **suspend 함수**로 비동기 작업을 동기 코드처럼 작성
2. **적절한 Dispatcher** 선택 (IO vs Default)
3. **구조화된 동시성**으로 생명주기 관리
4. **에러 처리**를 명확하게
5. **GlobalScope 사용 지양**, 명시적인 CoroutineScope 사용

### 프로젝트에서 배울 수 있는 패턴
- `DatabaseFactory.dbQuery`: Dispatcher 전환
- `DockerExecutorService`: IO 작업 처리
- `SubmissionService`: 백그라운드 작업 실행
- `ServiceRegistry`: 리소스 생명주기 관리

### 추가 학습 자료
- [Kotlin Coroutines 공식 가이드](https://kotlinlang.org/docs/coroutines-guide.html)
- [Coroutines Best Practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- [Flow 공식 문서](https://kotlinlang.org/docs/flow.html)

---

이 가이드를 통해 Kotlin 코루틴의 고급 개념을 이해하고, 
실전 프로젝트에서 효과적으로 활용할 수 있기를 바랍니다!
