# 18장 - (p.739 - 769)

## 코루틴 내부에서 던져진 오류 처리

- 일시 중단 함수나 코루틴 빌더 안에서 발생하는 예외를 처리하기 위해 launch 나 async 호출을 try-catch 로 감싸는 것은 효과가 없음
    - 이들이 코루틴 빌더 함수이기 때문
- 코루틴 빌더는 실행할 새로운 코루틴을 생성하는데, 이 새로운 코루틴에서 발생한 예외는 (코루틴 빌더를 감싸고 있는) catch 블록에 의해 잡히지 않음
    - 마치 새로 생성된 스레드에서 발생한 예외가 스레드를 만든 코드에서 잡히지 않듯

```kotlin
import kotlinx.coroutines.*

fun main(): Unit = runBiocking {
	try {
		launch {
			throw unsupportedOperationException("Ouch!")
		}
	} catch (u: UnsupportedOperationException) {
		println("Handled $u") // 실행되지 않음
	}
}
```

```kotlin
import kotlinx.coroutines.*

fun main(): Unit = runBiocking {
	launch {
		try {
			throw unsupportedOperationException("Ouch!")
		} catch (u: UnsupportedOperationException) {
			println("Handled $u")
		}
	}
}

// Handled java.lang.UnsupportedOperationException: Ouch!
```

- async 로 생성된 코루틴이 예외를 던진다면 그 결과에 대해 await 을 호출할 때 이 예외가 다시 발생함
    - await 가 원하는 타입의 의미있는 값을 던져줄 수 없기 때문에 예외를 던저야만 함
    - await 을 try-catch 로 감싸면 이 예외를 처리할 수 있음

```kotlin
import kotlinx.coroutines.*

fun main(): Unit = runBiocking {
	val myDeferredInt: Deferred<Int> = async {
		throw UnsupportedOperationException("Ouch!")
	}
	try {
		val i: Int = myDeferredInt.await()
		println(i)
	} catch (u: UnsupportedOperationException) {
		println("Handled: $u")
	}
}
```

- 위 예제를 실행하면 await() 을 감싼 try-catch 에서 예외를 잡는 것을 확인할 수 있음
- 하지만 동시에 오류 콘솔에도 에러가 출력됨
    
    ```kotlin
    Handled: java.lang.UnsupportedOperationException: Ouch!
    Exception in thread "main" java.lang.UnsupportedOperationException: Ouch!
    	at MyExampleKt$main$1$myDeferred$1.invokeSuspend(MyExample.kt:6)
    	...
    ```
    
- 이는 await 이 예외를 다시 던지지만, 원래의 예외도 여전히 관찰되기 때문
    - async 는 예외를 부모 코루틴인 runBlocking 에 전파하고, 프로그램은 종료됨
- 자식 코루틴은 잡히지 않은 예외를 항상 부모 코루틴에 전파
    - 이는 부모 코루틴이 예외를 처리해야 할 책임을 가진다는 의미

## 코틀린 코루틴에서의 오류 전파

- 구조적 동시성 패러다임은 자식 코루틴에서 발생한 잡히지 않은 예외가 부모 코루틴에 의해 어떻게 처리되는지에 영향을 줌
    - 자식에게 작업을 나누는 방식에 따라 자식의 오류를 처리하는 방식도 달라짐
    - 자식 중 하나의 실패가 부모의 실패로 이어질 것인지 여부에 따라 2가지 방식으로 나눌 수 있음
        1. 코루틴이 작업을 동시적으로 분해해 처리하는 경우 자식 중 하나의 실패는 더 이상 최종 결과를 얻을 수 없다는 점을 의미
            1. 이런 경우 부모 코루틴도 예외로 완료돼야 하며 , 여전히 작업 중인 다른 자식은 더 이상 필요가 없는 결과를 생성하는 것을 피하기 위해 취소된다 
            2. 한 자식의 실패가 부모의 실패로 이어짐
        2. 두 번째 경우는 하나의 자식이 실패해도 전체 실패로는 이어지지 않을 때
            1. 자식들에게 벌어진 실패를 부모가 처리해야 하지만 자식의 실패로 인해 시스템 전쳬가 실패하면서 멈추지는 말아야 하는 경우를 자식이 부모의 실행을 감독한다고 말함
            2. 이러한 감독 코루틴은 일반적으로 코루틴 계층의 최상위에 위치
                1. 예를 들어 서버 프로세스는 여러 자식 작업을 시작하고 이들의 실행을 감독할 수 있음
                2. 다른 예로 최신 데이터를 가져오는 작업이 실패하더라도 UI 구성 요소는 계속 살아 있어야만 함
                3. 이런 경우 어느 한 자식의 실패가 부모의 실패로 이어지지 않음

- 코틀린 코루틴에서 자식 코루틴을 부모가 어떻게 처리할지는 부모 코루틴의 콘텍스트에 Job (자식의 실패가 부모의 실패로 이어짐) 과 SupervisorJob (부모가 자식을 감독함) 중 어느 것이 있는지에 따라 달라짐

### 자식이 실패하면 모든 자식을 취소하는 코루틴

- 코루틴 간의 부모-자식 계층은 Job 객체를 통해 구축됨
    - 따라서 코루틴이 SupervisorJob 없이 생성된 경우 자식 코루틴에서 발생한 잡히지 않은 예외는 부모 코루틴을 예외로 완료시키는 방식으로 처리됨
    - 실패한 자식 코루틴은 자신의 실패를 부모에게 전파함
        - 부모는:
            - 불필요한 작업을 막기 위해 다른 모든 자식을 취소시킴
            - 같은 예외를 발생시키며 자신의 실행을 완료
            - 자신의 상위 계층으로 예외를 전파
    - 이런 동작은 같은 스코프 안에서 동시성 계산을 함께 수행하고 공통의 결과를 반환하는 코루틴 그룹에게 아주 유용함

```kotlin
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.milliseconds
import kotlin.time.Duration.Companion.seconds

fun main(): Unit = runBlocking {
	launch {
		try {
			while (true) {
				println("Heartbeat!")
				delay(580. milliseconds)
			}
		} catch (e: Exception) {
			println("Heartbeat terminated: $e")
			throw e
		}
	}
	launch {
		delay(1.seconds)
		throw UnsupportedOperationException("Ow!")
	}
}
```

- 형제 코루틴 중 하나가 예외를 던지면, 하트비트 코루틴도 취소됨
- 해당 오류 전파 동작은 launch 로 시작된 코루틴뿐만 아니라, 모든 코루틴에게도 적용됨
    - 자식 코루틴을 async 로 시작해도 같은 동작을 볼 수 있음

### 구조적 동시성은 코루틴 스코프를 넘는 예외에만 영향을 미침

- 형제 코루틴을 취소하고 예외를 코루틴 계층 상위로 전파하는 이 동작은 코루틴 스코프를 넘는 처리되지 않은 예외에만 영향을 미침
    - 따라서 이 동작을 피하는 가장 쉬운 방법은 처음부터 스코프를 넘는 예외를 던지지 않는 것
    - 한 코루틴 안에만 속해있는 try-catch 블록은 예상대로 동작함

- 처리되지 않은 예외를 코루틴 계층 위쪽으로 전파하고 형제 코루틴을 취소하는 것은 애플리케이션에서 구조적 동시성 패러다임을 강제하는 데 도움이 됨
    - 하지만 처리되지 않은 예외 하나가 전체 애플리케이션을 무너뜨려서는 안 됨
    - 이 오류 전파에 대한 경계를 정의할 수 있어야 함
    - 코틀린 코루틴에서는 슈퍼바이저 코루틴을 사용해 경계를 정의할 수 있음

### 슈퍼바이저는 부모와 형제가 취소되지 않게 함

- 슈퍼바이저는
    - 자식이 실패하더라도 생존함
    - Job 을 사용하는 스코프와 달리 슈퍼바이저는 일부 자식이 실패를 보고하더라도 실패하지 않음
    - 다른 자식 코루틴을 취소하지 않음
    - 예외를 구조적 동시성 계층 상위로 전파하지 않음
    - 이 떄문에 종종 코루틴 계층의 최상위 코루틴으로써 슈퍼바이저가 쓰임
- 코루틴이 자식 코루틴의 슈퍼바이저가 되려면 그 코루틴에 연관된 Job 이 일반적인 Job 이 아니라 SupervisorJob 이여야 함
- 예외가 출력됐는데도 애플리케이션이 계속 실행될 수 있는 이유
    - SupervisorJob 이 launch 빌더로 시작된 자식 코루틴에 대해 CoroutineExceptionHandler 를 호출하기 때문
    - 코루틴을 지원하는 프레임워크는 종종 슈퍼바이저 역할을 하는 코루틴 스코프를 기본적으로 제공
- 미세한 작업 함수들은 일반적으로 슈퍼바이저를 사용하지 않음
    - 이는 불필요한 작업이 취소되는 것이 코루틴 오류 전파에서 바람직한 특성이기 때문

## CoroutineExceptionHandler: 에러 처리를 위한 마지막 수단

- 자식 코루틴은 처리되지 않은 예외를 부모 코루틴에 전파함
    - 이때 예외가 슈퍼바이저에 도달하거나 계층의 최상위로 가서 부모가 없는 루트 코루틴에 도달하면 에외는 더 이상 전파되지 않음
    - 이 시점에서 처리되지 않은 예외는 CoroutineExceptionHandler 라는 특별한 핸들러에게 전달됨
    - 이 핸들러는 코루틴 콘텍스트의 일부
    - 코루틴 콘텍스트에 예외 핸들러가 없다면 처리되지 않은 시스템 예외는 전역 예외 핸들러로 이동
- JVM 프로젝트와 안드로이드 프로젝트의 시스템 전역 예외 핸들러는 다름
    - JVM: 핸들러가 예외 스택 트레이스를 오류 콘솔에 출력
    - 안드로이드: 시스템 전역 예외 핸들러가 앱을 오류와 함께 종료시킴

- CoroutineExceptionHandler 를 코루틴 콘텍스트에 제공하면 처리되지 않은 예외를 처리하는 동작을 커스텀화할 수 있음
- CoroutineExceptionHandler 를 코루틴 콘텍스트에 원소로 추가할 수 있음

```kotlin
import kotlinx.coroutines.*
import kotlin.time.Duration.Companion.seconds

class ComponentWithScope(dispatcher: CoroutineDispatcher = Dispatchers.Default) {
	private val exceptionHandler = CoroutineExceptionHandler { _, e ->
		println("[ERROR] ${e.message}")
	}
	private val scope = CoroutineScope(
		SupervisorJob() + dispatcher + exceptionHandler
	)
	fun action() = scope.launch {
		throw UnsupportedOperationException("Ouch!")
	}
}

fun main() = runBlocking {
	val supervisor = ComponentWithScope()
	supervisor. action()
	delay(1.seconds)
}

// [ERROR] Ouch!
```

- 예외가 커스텀 예외 핸들러에 의해 처리됨

- 자식 코루틴, 즉 코루틴 스코프에서 시작된 코루틴이나 다른 코루틴에서 시작된 코루틴은 처리되지 않은 예외의 처리를 부모에게 위임
    - 부모는 다시 이 처리를 자신의 부모에게 위임하며 계층의 최상위에 이를 떄까지 이런 위임이 계속됨
    - 따라서 ‘중간에 있는’ CoroutineExceptionHandler 따위는 없음
    - 루트 코루틴이 아닌 코루틴의 콘텍스트에 설치된 핸들러는 결코 사용되지 않음
- 계층의 최상위에 있는 코루틴 예외 핸들러만 실행되며, 중간에 있는 핸들러는 쓰이지 않음
    - 이는 예외가 여전히 부모 코루틴에게 전파될 수 있기 때문
        - 중간에 launch 호출에 코루틴 예외 핸들러가 있음에도 루트 코루틴이 아니기 때문에 예외가 계속해서 코루틴 계층을 따라 전파됨
        - 그 결과, 최상위 코루틴인 GlobalScope.launch 의 예외 핸들러만 호출됨

### CoroutineExceptionHandler 를 launch 와 async 에 적용할 때의 차이점

- CoroutineExceptionHandler 살펴볼 때 예외 핸들러는 계층의 최상위 코루틴이 launch 생성된 경우에만 호출된다는 점에 유의
    - 최상위 코루틴이 async 로 생성된 경우에는 CoroutineExceptionHandler 가 호출되지 않음

- 최상위 코루틴이 async 로 시작되면, 이 예외를 처리하는 책임은 await() 를 호출하는 Deferred 의 소비자에게 있음
    - 따라서 코루틴 예외 핸들러는 이 예외를 무시할 수 있음
    - 그리고 소비자 코드는 await 호출을 try-catch 블록으로 감싸는 방식으로 예외를 처리할 수 있음
    - 이 경우에는 try-catch 가 코루틴 취소에 영향을 끼치지 못함

## 플로우에서 예외 처리

- 일반적으로 플로우의 일부분에서 예외가 발생하면 collect 에서 예외가 던저짐
    - 이는 collect 를 try-catch 블록으로 감싸면 예상대로 동작한다는 뜻
    - 이때 플로우에 중간 연산자가 적용됐는지 여부와는 관계가 없음

### catch 연산자로 업스트림 예외 처리

- catch 는 플로우에서 발생한 예외를 처리할 수 있는 중간 연산자
- 이 함수에 연결된 람다 안에서 플로우에 발생한 예외에 접근할 수 있음
    - 이때 예외는 람다의 파라미터로 전달됨
- 이 연산자는 취소 예외를 자동으로 인식하기 때문에 취소가 발생한 경우에는 catch 블록이 호출되지 않음
- 게다가 catch는 스스로 값을 방출할 수도 있기 때문에 예외를 오류 값으로 변환해 다운스트림 플로우에서 소비할 수도 있음

### 술어가 참일 때 플로우 수집 재시도: retry

- catch와 마찬가지로 retry는 업스트림의 예외를 잡음
    - 예외를 처리하고 Boolean 값을 반환하는 람다를 사용할 수 있음
    - 람다가 true 를 반환하면 재시도가 시작됨
    - 재시도 동안 업스트림의 플로우가 처음부터 다시 수집되면서 모든 중간 연산이 다시 실행됨

## 코루틴과 플로우 테스트

- 테스트 메서드에서 코루틴을 사용하려면 runlest 코루틴 빌더를 사용하면 됨
- rurBlocking 빌더 함수는 일반 코틀린 코드와 동시성 코틀린 코드 사이에 다리를 놓는 역할을 하기 때문에 일시 중단 함
수나 코루틴, 플로우를 사용하는 코드를 테스트할 때도 이를 쓸 수 있음
    - runBlocking을 사용하면 테스트가 실시간으로 실행됨
        - 이는 코드에 delay가 지정된 경우에 결과가 계산되기 전에 시간 지연이 전부실행된다는 뜻

### 코루틴을 사용하는 테스트를 빠르게 만들기: 가상 시간과 디스패처

- 코틀린 코루틴은 가상 시간을 사용해 테스트 실행을 빠르게 진행할 수 있게 해줌
    - 가상 시간을 사용할 때는 지연이 자동으로 빠르게 진행됨

## 요약

- 한 코루틴에만 국한된 예외는 코루틴이 아닌 일반적인 코드와 마찬가지로 처리할 수 있음
    - 코루틴 경계를 넘는 예외는 좀 더 주의를 기울여야 함
- 기본적으로 코루틴에서 처리되지 않은 예외가 발생하면 부모 코루틴과 모든 형제 코루틴이 취소됨
    - 이를 통해 구조적 동시성 개념이 강제로 적용됨
- `supervisorScope` 나 `supervisorJob` 을 사용하는 다른 코루틴 스코프에서 사용되는 슈퍼바이저는 자식 코루틴 중 하나가 실패해도 다른 자식 코루틴을 취소하지 않음
    - 또한 처리하지 않은 예외를 코루틴 계층의 위로 전파하지도 않음
- `await` 은 `async` 코루틴에서 발생한 예외를 다시 던짐
- 슈퍼바이저는 애플리케얘츠에서 오랫동안 실행되는 부분에 자주 사용됨
    - 종종 케이토의 Application 처럼 프레임워크에 내장된 부품으로 제공되는 경우도 있음
- 처리되지 않은 예외는 슈퍼바이저를 만나거나 코루틴 계층의 최상단에 도달할 때까지 전파됨
    - 이 시점에서 처리되지 않은 예외는 코루틴 콘텍스트의 일부인 `CoroutineExceptionHandler` 에게 전달됨
    - 콘텍스트에 코루틴 예외 핸들러가 없으면 시스템의 전역 예외 핸들러에 전달됨
- JVM과 안드로이드에서 기본 시스템 예외 핸들러가 다름
    - JVM 에서는 스택 트레이스를 오류 콘솔에 기록하고, 안드로이드에서는 오류를 발생시키면서 앱을 중단시킴
- `CoroutineExceptionHandler` 는 예외를 처리하는 마지막 수단
    - 예외를 잡을 수는 없지만, 예외가 기록되는 방식을 정의할 수 있음
    - `CoroutineExceptionHandler` 는 계층의 최상단에 있는 루트 코루틴의 콘텍스트에 위치
- 최상단 코루틴을 `launch` 빌더로 시작한 경우에만 `CoroutineExceptionHandler` 가 호출됨
    - `async` 빌더로 시작한 경우에는 이 핸들러가 호출되지 않음
        - `Deferred` 를 기다리는 코드가 예외를 처리해야 함
- 플로우에서 오류 처리는 `collect` 를 `try-catch` 문으로 감싸거나, 전용 `catch` 연산자를 사용
- catch 연산자는 업스트림에서 발생한 예외만 처리하며, 다운스트림의 예외는 무시
    - 심지어 예외를 다시 던져 다운스트림에서 처리하기 위해 `catch` 를 사용할 수도 있음
- `retry` 를 사용해 예외가 발생했을 때 플로우 수집을 처음부터 다시 시작할 수 있음
    - 이를 통해 코드가 오류를 복구할 수 있는 기회를 가질 수 있음
- `runTest` 의 가상 시간을 활용하면 코루틴 코드 테스트 속도를 높일 수 있음
    - 모든 지연 시간이 자동으로 빠르게 진행됨
- `TestCoroutineScheduler` 는 `runTest` 가 노출시키는 `TestScope` 의 일부로, 현재 가상 시간을 추적하며 `runCurrent` 와 `advanceUtilIdle` 같은 함수로 테스트 실행을 세밀하게 제어할 수 있음
- 테스트 디스패처는 단일 스레드로 작동
    - 이에 따라 테스트 단언문을 호출하기 전에 새로 시작한 코루틴들이 실행될 수 있는 시간을 수동으로 (yield 나 다른 일시 중단 지점을 호출하는 방식으로) 보장해줘야 함
- 터빈 라이브러리는 플로우 기반의 코드를 간편하게 테스트하게 해줌
    - 핵심 API 는 `test` 확장 함수
        - 플로우에서 원소를 수집하고 awaitItem 과 같은 함수를 사용해 테스트 중인 플로우의 원소 배출을 확인할 수 있음
