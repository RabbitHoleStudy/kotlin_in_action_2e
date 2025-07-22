# 코루틴 내부에서 던져진 예외 처리

- `launch`, `async`는 **새로운 코루틴을 시작하는 코루틴 빌더 함수**이다.
- 이 코루틴 안에서 예외가 발생해도, **외부 try-catch로는 잡을 수 없다**.
- 이는 마치 **새로운 스레드 안에서 발생한 예외가 스레드를 생성한 곳에서 잡히지 않는 것**과 같다.

```kotlin
fun main(): Unit = runBlocking {
    try {
        launch {
            throw UnsupportedOperationException("Ouch!")
        }
    } catch (e: UnsupportedOperationException) {
        println("Handled: $e") // 실행되지 않음
    }
}
```

## 올바른 처리법: launch 블록 안에서 try-catch 사용

```kotlin
fun main(): Unit = runBlocking {
    launch {
        try {
            throw UnsupportedOperationException("Ouch!")
        } catch (e: UnsupportedOperationException) {
            println("Handled: $e")
        }
    }
}
```

- 코루틴 **내부**에서 예외가 발생하고, **내부**에서 처리되므로 문제 없음.

## async의 경우에는 await 시 예외가 다시 발생

```kotlin
fun main(): Unit = runBlocking {
    val myDeferredInt: Deferred<Int> = async {
        throw UnsupportedOperationException("Ouch!")
    }

    try {
        val i: Int = myDeferredInt.await()
        println(i)
    } catch (e: UnsupportedOperationException) {
        println("Handled: $e")
    }
}
```

- `await()`는 `Deferred<T>`의 결과를 가져오는 suspend 함수.
- `async` 블록에서 예외가 발생하면, `await()` 호출 시 예외가 **다시 발생**함.
- 이 예외는 `try-catch`로 잡을 수 있음.
- 하지만 **콘솔에는 예외 메시지가 출력된다.**

**이유: 예외는 부모 코루틴(runBlocking)으로 전파되기 때문**

- 자식 코루틴에서 처리되지 않은 예외는 **부모 코루틴으로 전파된다.**
- 부모 코루틴이 예외를 처리하지 않으면 프로그램은 **그대로 종료된다.**
- 예외가 두 번 발생하는 것처럼 보이지만, 실제로는
    - `await()`에서 한 번
    - 부모(runBlocking)로 전파되어 다시 한 번

## 결론

- `launch` → 내부에서 try-catch로 직접 잡아야 한다.
- `async` → 예외는 `await()`에서 다시 발생하므로, 해당 지점에서 try-catch로 잡아야 한다.
- 부모 코루틴은 자식의 **잡히지 않은 예외**를 항상 전파받는다.
- 따라서 전체 코루틴 구조 내에서 **적절한 위치에서 예외를 소비하거나 처리하는 전략**이 필요하다.

# 코루틴에서의 오류 전파

## 구조적 동시성에서의 오류 전파 개념

- 구조적 동시성의 목적은 **자식의 실패를 부모가 인지하고 통제하게 하는 것**.
- 자식 코루틴에서 예외가 발생하면, 처리 방식은 부모 코루틴의 **Job 종류**에 따라 달라진다:
    - 일반 `Job`: 자식이 실패하면 부모도 실패 (형제 코루틴도 모두 취소)
    - `SupervisorJob`: 자식이 실패해도 부모는 실패하지 않음 (다른 자식은 계속 실행)

## 자식이 실패하면 부모도 실패하는 경우 (`Job`)

- 기본 코루틴 컨텍스트에서는 일반 Job을 사용
- 자식 코루틴 중 하나가 예외로 실패하면:
    1. 부모 코루틴이 해당 예외를 전파받고 예외로 종료
    2. 같은 스코프의 **형제 코루틴**도 전부 취소
    3. 예외는 코루틴 계층의 상위로 계속 전파

```kotlin
runBlocking {
    launch {
        // 무한 루프 하트비트
    }
    launch {
        delay(1000)
        throw UnsupportedOperationException("Oops")
    }
}
```

출력:

```
Heartbeat!
Heartbeat!
Heartbeat terminated: Parent job is Cancelling
Exception in thread "main": ...
```

- 자식 둘 중 하나가 실패 → 부모가 예외로 종료 → 다른 자식(하트비트)도 취소됨

## 예외를 스코프 내부에서 잡으면 취소 전파가 막힘

- `launch` 내부에서 try-catch로 예외를 직접 잡으면, 형제 코루틴에 영향을 주지 않음
- 이 경우 예외는 상위로 전파되지 않음

예제:

```kotlin
launch {
    try {
        delay(1.seconds)
        throw UnsupportedOperationException("Oops")
    } catch (e: UnsupportedOperationException) {
        println("Caught: $e")
    }
}

```

출력:

```
Caught: java.lang.UnsupportedOperationException: Oops
Heartbeat!
Heartbeat!
```

### SupervisorJob을 사용한 오류 격리

- `SupervisorJob` 또는 `supervisorScope`를 사용하면:
    - 자식 하나가 실패해도 부모와 형제 코루틴은 계속 실행
    - 예외는 전파되지 않음

예제:

```kotlin
runBlocking {
    supervisorScope {
        launch {
            while (true) {
                println("Heartbeat!")
                delay(500)
            }
        }
        launch {
            delay(1000)
            throw UnsupportedOperationException("Oops")
        }
    }
}
```

출력:

```
Heartbeat!
Heartbeat!
Exception in thread "main": ...
Heartbeat!
Heartbeat!
```

- 예외는 발생하지만, 하트비트 코루틴은 계속 실행됨
- supervisorScope는 자식 실패에 대해 **격리된 실패**를 허용

### SupervisorScope의 활용

- 앱의 최상위 레벨에서 **전체 시스템 중단 없이 일부 실패만 무시**하고 싶은 경우 사용
- 예: Android 앱의 Application scope, 서버의 요청 핸들러 외부 scope 등

적절한 위치에서 `SupervisorJob` 또는 `supervisorScope`를 쓰면:

- 요청 단위: 일반 `Job` (모든 계산이 중단되도록)
- 앱 단위: `SupervisorJob` (하나 실패해도 전체 서비스는 유지)

### 주의 사항

- **`⚠️ CancellationException`은 잡더라도 반드시 다시 던져야 함**
    
    → 생명주기 종료를 방해하면 안 됨
    
- 슈퍼바이저가 있다고 해도, **예외는 로깅되지 않으면 조용히 무시될 수 있음**
- launch는 예외를 전파하지 않으므로, **예외 처리를 명확히 해야 함** (18.3에서 더 다룸)

# CoroutineExceptionHandler: 예외 처리를 위한 마지막 수단

## CoroutineExceptionHandler 정의 예

```kotlin
val handler = CoroutineExceptionHandler { context, exception ->
    println("[ERROR] ${exception.message}")
}
```

- 코루틴의 **콘텍스트 요소로 추가**하여 사용한다.
- `SupervisorJob`과 함께 사용하는 것이 일반적.

## 실제 사용 예

```kotlin
class ComponentWithScope(
    dispatcher: CoroutineDispatcher = Dispatchers.Default
) {
    private val handler = CoroutineExceptionHandler { _, e ->
        println("[ERROR] ${e.message}")
    }

    private val scope = CoroutineScope(SupervisorJob() + dispatcher + handler)

    fun action() = scope.launch {
        throw UnsupportedOperationException("Ouch!")
    }
}

fun main() = runBlocking {
    val supervisor = ComponentWithScope()
    supervisor.action()
    delay(1.seconds)
}
```

- 출력 결과: `[ERROR] Ouch!`
- 예외는 핸들러에 의해 처리됨. 애플리케이션은 종료되지 않음.

## 예외가 전파되는 계층 구조의 원칙

- 예외는 **부모 코루틴에게 위임**되며 최상위에 이를 때까지 전파된다.
- 예외 핸들러는 **최상위 코루틴에만 적용된다**.
- **중간 코루틴에 설치된 CoroutineExceptionHandler는 호출되지 않는다.**

### 예: 중간 핸들러는 무시됨

```kotlin
val topLevelHandler = CoroutineExceptionHandler { _, e ->
    println("[TOP] ${e.message}")
}

val intermediateHandler = CoroutineExceptionHandler { _, e ->
    println("[INTERMEDIATE] ${e.message}")
}

fun main() {
    GlobalScope.launch(topLevelHandler) {
        launch(intermediateHandler) {
            throw UnsupportedOperationException("Ouch!")
        }
    }
    Thread.sleep(1000)
}
```

- 출력: `[TOP] Ouch!`
- `intermediateHandler`는 호출되지 않음
- 예외는 최상위 코루틴(`GlobalScope.launch`)의 핸들러에서만 처리된다

### launch vs async: 예외 처리 방식 차이

| 코루틴 빌더 | 예외 발생 시 | CoroutineExceptionHandler 작동 여부 |
| --- | --- | --- |
| `launch` | 내부에서 던진 예외는 즉시 전파 | O (최상위 launch에 한해) |
| `async` | 예외는 `await()` 호출 시 발생 | X (핸들러 호출 안 됨) |
- `async`로 시작된 코루틴은 예외가 Deferred 객체 내부에 **지연되어 저장**됨.
- 이 예외는 **await() 호출 시 다시 던져짐**.
- 따라서 `async` 예외는 **소비자(await 호출부)가 처리**해야 한다.

### 예제 비교: launch 사용

```kotlin
fun action() = scope.launch {
    async {
        throw UnsupportedOperationException("Ouch!")
    }
}
```

- 예외는 CoroutineExceptionHandler에 의해 처리됨 → `[ERROR] Ouch!`

### 예제 비교: async 사용

```kotlin
fun action() = scope.async {
    launch {
        throw UnsupportedOperationException("Ouch!")
    }
}
```

- 출력 없음
- 예외는 소비자가 `await()`할 때까지 대기함
- 핸들러는 호출되지 않음 → **직접 try-catch로 처리해야 함**

## 정리

- **CoroutineExceptionHandler는 launch 계열 최상위 코루틴에서만 작동**함
- async는 **Deferred를 반환하므로, 예외는 await() 시점에 처리**
- 중간 계층의 핸들러는 호출되지 않으며, 최상위 루트 코루틴에 있는 핸들러만 실행됨
- 예외를 안전하게 처리하려면:
    - launch 사용 시 → CoroutineExceptionHandler 설정
    - async 사용 시 → `await()`를 try-catch로 감쌀 것
- SupervisorJob과 함께 쓰면 예외 전파를 제어할 수 있고, 앱 전역 안정성이 높아짐

# 플로우에서 예외 처리

- 플로우의 생성(`flow {}`), 변환(`map`, `filter`, etc.), 수집(`collect`) 도중 예외가 발생할 수 있음
- 이 예외는 일반적인 `try-catch`로 `collect`를 감싸서 처리 가능

```kotlin
val exceptionalFlow = flow {
    repeat(5) { emit(it) }
    throw UnhappyFlowException()
}

runBlocking {
    try {
        exceptionalFlow.map { it * 2 }.collect {
            print("$it ")
        }
    } catch (e: UnhappyFlowException) {
        println("\nHandled: $e")
    }
}
```

출력:

```
0 2 4 6 8
Handled: UnhappyFlowException
```

### catch 연산자를 활용한 예외 처리 (업스트림 전용)

- `catch`는 중간 연산자로, **업스트림에서 발생한 예외만 처리**함
- `CancellationException`은 자동으로 무시됨 (catch로 잡히지 않음)
- `emit()`을 통해 기본 값을 대신 방출할 수도 있음

```kotlin
exceptionalFlow
    .catch { e ->
        println("Handled: $e")
        emit(-1)
    }
    .collect { println(it) }
```

출력:

```
0
1
2
3
4
Handled: UnhappyFlowException
-1
```

> catch 이후에 발생하는 예외는 잡히지 않음
> 

**catch가 동작하지 않는 경우**

- `catch`는 **자신보다 이전(업스트림)의 예외**만 처리할 수 있음
- `onEach`, `collect` 등 **catch 이후**에서 발생한 예외는 잡히지 않음

```kotlin
exceptionalFlow
    .map { it + 1 }
    .catch { println("Handled: $it") }
    .onEach { throw UnhappyFlowException() }
    .collect()
```

결과:

```
Exception in thread "main" UnhappyFlowException
```

> catch 이후에 발생한 예외이므로 무시됨
> 

### collect에서 발생한 예외는 try-catch로 직접 처리해야 함

```kotlin
runBlocking {
    try {
        exceptionalFlow.collect { value ->
            if (value > 2) throw RuntimeException("Too big")
            println(value)
        }
    } catch (e: RuntimeException) {
        println("Handled in collect: $e")
    }
}
```

### retry 연산자: 재시도 로직

- `retry`는 업스트림에서 발생한 예외가 특정 조건을 만족할 경우 **플로우 전체를 다시 수집**함
- 람다가 `true`를 반환하면 재시도
- 최대 재시도 횟수 지정 가능

```kotlin
val unreliableFlow = flow {
    println("Starting flow")
    repeat(10) {
        if (Random.nextDouble() < 0.1) throw CommunicationException()
        emit(it)
    }
}

unreliableFlow
    .retry(5) { cause ->
        println("Handled: $cause")
        cause is CommunicationException
    }
    .collect { println(it) }

```

예상 출력:

```
Starting flow
0
1
2
Handled: CommunicationException
Starting flow
0
1
2
3
...
```

> 매 재시도 시 flow {} 내부 코드가 처음부터 다시 실행됨
> 
> 
> **부수 효과(side-effect)가 있는 코드**에서는 주의 필요 (예: DB 저장, 네트워크 호출 등)
> 

### 핵심 정리

| 방법 | 대상 | 특징 |
| --- | --- | --- |
| `try-catch` | collect 전체 | 가장 바깥에서 예외를 처리할 수 있음 |
| `catch` | 업스트림 | 중간 연산자로, catch 이후의 예외는 처리 불가 |
| `retry` | 업스트림 | 조건에 맞는 예외 발생 시 플로우 전체를 재시도 |

> 플로우에서의 예외 처리는 catch 위치가 **`정확히 예외 발생 지점의 "뒤"`**에 있어야 함
> 
> 
> `catch → onEach → collect` 순서라면, **onEach/collect 예외는 catch로 처리되지 않음**
> 

# 코루틴과 플로우 테스트

- 코루틴 코드는 `runBlocking`으로도 테스트할 수 있지만, **실시간 delay가 발생**하므로 느림
- 해결책: **가상 시간을 활용**해 빠르게 테스트 실행 → `runTest` 사용

## 가상 시간 기반 테스트: `runTest`

- `runTest`는 내부적으로 **TestDispatcher**와 **TestScheduler**를 사용해 delay나 시간 흐름을 가상으로 처리
- `delay(20.seconds)` 같은 코드도 즉시 실행 가능

```kotlin
@Test
fun testDelay() = runTest {
    val startTime = System.currentTimeMillis()
    delay(20.seconds)
    println(System.currentTimeMillis() - startTime) // 수 밀리초 출력
}
```

### runTest의 특징

- 기본 timeout은 **60초 (가상 시간 기준)**
- 기본 디스패처는 단일 스레드 → 일시 중단 없는 launch는 순차 실행됨
- 실제 테스트에서 동시성 확인하려면 `yield()` 또는 `delay()` 삽입 필요

## 가상 시간 제어: `runCurrent`, `advanceUntilIdle`

```kotlin
@Test
fun testSchedule() = runTest {
    var x = 0
    launch { delay(500); x++ }
    launch { delay(1000); x++ }

    println(currentTime) // 0
    delay(600)
    assertEquals(1, x)
    println(currentTime) // 600
    delay(500)
    assertEquals(2, x)
    println(currentTime) // 1100
}
```

또는 명시적으로 실행 예약된 작업을 제어:

```kotlin
@Test
fun testAdvance() = runTest {
    var x = 0
    launch { x++ }
    launch {
        delay(200)
        x++
    }

    runCurrent() // 현재 예약된 코루틴 실행
    assertEquals(1, x)

    advanceUntilIdle() // 예약된 모든 코루틴 실행
    assertEquals(2, x)
}
```

> 주의: Dispatchers.Default로 실행된 코루틴은 가상 시간에 영향받지 않음 → 반드시 TestDispatcher를 주입 가능하게 구조 설계할 것
> 

## 플로우 테스트 방법

### 기본: `toList()` 활용

```kotlin
val myFlow = flowOf(1, 2, 3)

@Test
fun testFlow() = runTest {
    val result = myFlow.toList()
    assertEquals(listOf(1, 2, 3), result)
}
```

### 고급: Turbine 라이브러리 사용

- [Turbine GitHub](https://github.com/cashapp/turbine)
- `test` 확장 함수 제공 → `awaitItem()`, `awaitComplete()`, `awaitError()`로 플로우 동작 검증

```kotlin
@Test
fun turbineTest() = runTest {
    myFlow.test {
        assertEquals(1, awaitItem())
        assertEquals(2, awaitItem())
        assertEquals(3, awaitItem())
        awaitComplete()
    }
}
```

- **장점**: 무한 스트림, 예외 처리, 시간 순서 검증 등 복잡한 플로우 테스트에 적합
- **보장**: 방출된 모든 항목이 테스트에서 소비되도록 보장

### 핵심 요약

| 기능 | 설명 |
| --- | --- |
| `runBlocking` | 실시간 delay 발생, 테스트 느림 |
| `runTest` | 가상 시간 사용, 빠르고 정확한 테스트 가능 |
| `delay()`, `yield()` | 실행 순서 제어용 일시 중단 함수 |
| `TestCoroutineScheduler` | `runCurrent()`, `advanceUntilIdle()`로 실행 타이밍 세밀 제어 |
| `toList()` | 유한한 플로우 결과 수집 및 검증 |
| `Turbine` | 복잡한 플로우 테스트를 위한 확장 함수 및 어설션 제공 |

# 퀴즈

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun fetchUserIds(): Flow<Int> = flow {
    repeat(5) { emit(it) }
}

fun main() = runBlocking {
    fetchUserIds()
        .map { it + 1 }
        .catch { e ->
            println("Caught: $e")
        }
        .onEach {
            if (it == 3) throw IllegalStateException("Invalid user ID: $it")
            println("User ID: $it")
        }
        .collect()
}
```

다음중 출력 결과로 올바른 것은?

### 1번 - catch 되어 에러 문구 출력

```kotlin
User ID: 1
User ID: 2
"Caught: java.lang.IllegalStateException: Invalid user ID: 3"
```

### 2번 - 오류 발생없이 정상 동작

```kotlin
User ID: 1
User ID: 2
User ID: 3
User ID: 4
User ID: 5
```

### 3번 - 처리되지 않은 예외 발생

```kotlin

User ID: 1
User ID: 2
Exception in thread "main" java.lang.IllegalStateException: Invalid user ID: 3
```
