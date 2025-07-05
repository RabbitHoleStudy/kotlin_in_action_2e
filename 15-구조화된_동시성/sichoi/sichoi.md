## 코루틴 생성과 실행

### launch와 runBlocking

```kotlin
fun main() = runBlocking {
    launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
}
```

- `runBlocking`: 메인 스레드에서 코루틴 시작, 완료될 때까지 블로킹.
- `launch`: 코루틴 빌더, Job 반환. 비동기 수행.
- `delay`: 코루틴을 중단하지만 스레드를 블로킹하지 않음.

### async/await

```kotlin
suspend fun generateValue(): Int {
    delay(500)
    return Random.nextInt(0, 10)
}

suspend fun computeSum() = coroutineScope {
    val a = async { generateValue() }
    val b = async { generateValue() }
    println("Sum is ${a.await() + b.await()}")
}

```

- `async`는 `Deferred`를 반환.
- `await()`는 suspend 함수이며 결과를 기다림.

# 실전 코루틴

## 코루틴 취소와 예외 처리

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("job: I'm working $i ...")
        delay(500L)
    }
}
delay(1300L)
job.cancel() // Job 취소
```

- `cancel()`은 Job 상태를 `Cancelled`로 설정.
- `delay` 같은 일시 중단 지점에서 코루틴이 취소됨.
- CPU 바운드 작업에는 `ensureActive()` 또는 `yield()`로 중단 지점 명시 필요.

```kotlin
val job = launch {
    repeat(1000) {
        ensureActive() // 명시적으로 취소 가능성 체크
        doCpuHeavyWork()
    }
}
```

## 구조화된 동시성 (Structured Concurrency)

- `coroutineScope` 안에서 시작된 모든 자식 코루틴은 부모가 완료될 때까지 완료되어야 함.
- 예외 전파 및 취소 관리가 쉬움.

```kotlin
suspend fun doWork() = coroutineScope {
    launch {
        delay(1000)
        println("Child 1 done")
    }
    launch {
        delay(500)
        println("Child 2 done")
    }
}
```

## SupervisorJob

- 자식 중 하나가 실패해도 나머지 자식에게 영향을 주지 않게 함.

```kotlin
val scope = CoroutineScope(SupervisorJob())

scope.launch {
    throw RuntimeException("fail!") // 다른 launch에는 영향 없음
}
```

## withTimeout / withTimeoutOrNull

- 특정 시간 안에 코루틴을 완료하지 못하면 취소됨.

```kotlin
val result = withTimeoutOrNull(1000L) {
    longRunningTask()
}
println(result ?: "Timeout!")
```

## CoroutineScope를 활용한 컴포넌트 관리

```kotlin
class Component {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    fun start() {
        scope.launch {
            while (true) {
                delay(1000)
                println("working...")
            }
        }
    }

    fun stop() {
        scope.cancel() // 모든 코루틴 종료
    }
}
```

- `SupervisorJob`은 다른 실패와 무관하게 코루틴 관리.
- `scope.cancel()`을 통해 컴포넌트 단위 코루틴 일괄 정리 가능.

## 예외 처리

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("Caught exception: $exception")
}

GlobalScope.launch(handler) {
    throw RuntimeException("Boom!")
}
```

- `CoroutineExceptionHandler`는 `launch`에서만 작동 (not async).
- `async`에서는 반드시 `await()` 시에 try-catch로 직접 잡아야 함.

## 취소에 안전한 코드 작성법

- CPU 바운드 루프 안에서는 반드시 `ensureActive()` 삽입.
- 정리(clean-up)는 `finally` 블록에 작성.

```kotlin
try {
    delay(1000)
} finally {
    println("Cleaning up")
}
```

# 결론 정리

| 개념 | 설명 | 관련 키워드 |
| --- | --- | --- |
| 코루틴 빌더 | 코루틴 실행 방법 | `launch`, `async`, `runBlocking` |
| 중단 함수 | 일시 중단 가능한 함수 | `suspend`, `delay` |
| 취소 | 코루틴을 중단 | `cancel`, `isActive`, `ensureActive` |
| 동시성 제어 | 부모-자식 관계로 제어 | `coroutineScope`, `SupervisorJob` |
| 시간 제한 | 실행 시간 초과 제어 | `withTimeout`, `withTimeoutOrNull` |
| 예외 처리 | 코루틴 내 에러 처리 | `CoroutineExceptionHandler`, `try-catch` |
| 컴포넌트 관리 | 스코프를 컴포넌트 수명에 묶기 | `CoroutineScope`, `cancel()` |

# 퀴즈 문제

```kotlin
import kotlinx.coroutines.*
import java.util.concurrent.CancellationException
import kotlin.coroutines.*

fun main() = runBlocking {
    val job = launch {
        try {
            coroutineScope {
                launch {
                    delay(500)
                    println("Child 1 done")
                }

                launch {
                    delay(300)
                    throw CancellationException("Cancelled from Child 2")
                }
            }
        } catch (e: Exception) {
            println("Caught in outer: $e")
        } finally {
            println("Finalizing job")
        }
    }

    job.join()
    println("Done")
}
```

- 해당 코드의 출력 결과는 ?

