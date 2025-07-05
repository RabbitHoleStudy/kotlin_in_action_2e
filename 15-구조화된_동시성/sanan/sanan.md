- 요약
    - **구조화된 동시성**은 코루틴의 실행 범위를 명확히 하여, **취소되지 않은 채로 남아있는 코루틴**을 방지한다.
    - `coroutineScope{}`는 **병렬 작업을 분해하고 결과를 통합**하는 데 사용하는 suspend 함수이며, **작업 단위 내의 스코프**를 제공한다.
    - **`CoroutineScope` 생성자**는 **클래스의 생명주기와 연관된 스코프**를 정의할 때 사용되며, 주로 SupervisorJob과 함께 사용한다.
    - `GlobalScope`는 예제에 자주 등장하지만, **구조화된 동시성을 해치는 대표적인 예**이므로 실제 앱 코드에서는 사용을 피해야 한다. - 어플리케이션의 라이프타임에 맞춰서 주기적으로 동기화해야하는 주요 작업 등에 사용할 가능성은 있다.
    - **코루틴 컨텍스트**는 코루틴의 실행 방식과 상속 구조를 제어하며, **부모-자식 계층 구조**를 형성하는 데 사용된다.
    - 코루틴 간 계층 구조는 **`Job` 객체**를 통해 설정되며, **스코프 간 취소 전파**에 관여한다.
    - 일시 중단 지점(suspension point)은 코루틴이 일시 중단되고, 다른 작업이 실행될 수 있도록 허용하는 시점이다.
    - **코루틴 취소는 일시 중단 지점에서 `CancellationException`을 던지는 방식**으로 이뤄지며, 예외는 무시하지 말고 재던지거나 그대로 흘려보내야 한다.
    - **취소는 비정상이 아니라 설계의 일부**이며, cancel, withTimeoutOrNull 등을 통해 명시적으로 처리할 수 있다.
    - **suspend 키워드만으로는 취소를 지원하지 않으며**, 취소 가능한 일시 중단 함수 작성을 위해 `isActive`, `ensureActive`, `yield` 등을 함께 사용해야 한다.
    - 프레임워크에서는 코루틴 스코프를 통해 **화면 표시나 요청 처리 등 생명주기 기반 작업**과 코루틴을 연결해준다. (`ViewModelScope`, `lifecycleScope` 등)
- 빌더 내부의 coroutineScope{} 를 이용한 async 예제
    
    ```kotlin
    suspend fun generateValue(): Int {
        delay(Random.nextInt(1, 4).seconds)
        return Random.nextInt()
    }
    
    suspend fun computeSum() {
        println("Computing a sum...")
        val sum = coroutineScope {
            val a = async {
                generateValue().also { println("Computed a value: $it") }
            }
            val b = async {
                generateValue().also { println("Computed a value: $it") }
            }
            a.await() + b.await()
        }
        println("Sum: $sum")
    }
    
    fun main() {
        runBlocking(Dispatchers.IO) {
            computeSum()
        }
    }
    ```
    
- 코루틴 빌더로부터 전파까지
    1. `launch` 또는 `async` 같은 코루틴 빌더가 호출되면,
    2. 내부적으로 새로운 `CoroutineContext`가 만들어지고 (기존 `scope`의 `context` + 추가 `context`)
    3. 거기서 새로운 `Job`이 생성되고 `context`에 추가됨.
    4. 이 `Job`은 부모 `Job`에 **자식으로 등록**되어 **취소나 완료 신호를 상속**받는다.
    5. `context` 내 `Dispatcher`는 실행할 스레드를 지정하고,
    6. 예외는 `context`에 설정된 `CoroutineExceptionHandler`로 전파됨.
- 커스텀 CoroutineContext.Element를 만들어서, 우리의 CoroutineScope에 원하는 값을 넣어보자
    
    ```kotlin
    class CustomCoroutineContextElement(
        val message: String,
    ) : CoroutineContext.Element {
    
        companion object Key : CoroutineContext.Key<CustomCoroutineContextElement>
    
        override val key: CoroutineContext.Key<*> = Key
    }
    
    fun main() {
        runBlocking(Dispatchers.IO + CustomCoroutineContextElement("이것은 나의 마음이 담긴 메시지여")) {
            val customMessage = coroutineContext[CustomCoroutineContextElement.Key]?.message
            launch {
                println("마음이 담긴 메세지는 과연??")
                delay(1.seconds)
                println("메시지 : $customMessage")
            }
        }
    }
    ```
    
- 현재 CoroutineContext의 Job이 종료될 때 리스너를 달아보자
    
    ```kotlin
    fun main() {
        runBlocking {
            val currentScopeJob = coroutineContext[Job]!!
            currentScopeJob.invokeOnCompletion { throwable ->
                if (throwable == null) println("와 성공이다")
                else println("헉 터졌다")
            }
    
            launch {
                delay(0.5.seconds)
                println("작업 1 완료!")
            }
            launch {
                delay(1.0.seconds)
                println("작업 2 완료!")
            }
    //        launch {
    //            delay(1.5.seconds)
    //            throw IllegalStateException("갑자기 에러가 터져버렸어잉")
    //        }
        }
    }
    ```
    
- 커스텀 Job을 이용해서 해당 coroutine이 종료될 때 리스너를 달아보자
    
    ```kotlin
    import kotlinx.coroutines.*
    import kotlin.coroutines.ContinuationInterceptor
    import kotlin.time.Duration.Companion.seconds
    
    class CustomCoroutineLoggingJob(
        private val origin: Job,
        private val tag: String,
    ) : Job by origin {
        init {
            origin.invokeOnCompletion {
                println("[$tag] finished!!")
            }
        }
    }
    
    fun main() {
        runBlocking(Dispatchers.IO) {
            println("current dispathcer = ${coroutineContext[ContinuationInterceptor]!!::class.simpleName}")
            withContext(CustomCoroutineLoggingJob(coroutineContext[Job]!!, "mine")) {
                println("inside dispathcer = ${coroutineContext[ContinuationInterceptor]!!::class.simpleName}")
                launch {
                    delay(0.3.seconds)
                    println("작업 A 완료")
                }
    
                launch {
                    delay(0.7.seconds)
                    println("작업 B 완료")
                }
            }
        }
    }
    ```
    
- runBlocking등의 builder를 사용할 때 Dispatcher를 지정하지 않은 경우
    
    ```kotlin
    fun main() {
        runBlocking {
            println("current dispathcer = ${coroutineContext[ContinuationInterceptor]!!::class.simpleName}")
        } // current dispathcer = BlockingEventLoop
    }
    ```
    
    `Dispatcher`가 나타나지 않고, `BlockingEventLoop`이 나타난다.
    
    - `BlockingEventLoop`?
        - `runBlocking`은 **현재 스레드를 차단(block)** 하면서, 코루틴 내부에서 `suspend` 함수들이 **비차단식(non-blocking)** 으로 실행되도록 만들어줌.
        - 이를 위해 `BlockingCoroutine`이라는 특수한 `CoroutineScope`를 만들고, 여기에 붙는 디스패처가 `BlockingEventLoop`.
        - `ContinuationInterceptor`로 동작하면서, **Coroutine을 순차적으로 실행하고, 필요한 경우에는 현재 스레드에서 직접 처리한다.**
    - 어째서 `Dispatchers.Default`가 아닐까?
        - `runBlocking`은 **진입점(main thread나 테스트 thread)** 에서 사용되므로, `Dispatchers.Default`처럼 새로운 스레드 풀을 사용하지 않음.
        - 대신 현재 스레드를 점유하면서 그 안에서 suspend 함수의 실행을 “이벤트 루프처럼” 처리함.
    - `ContinuationInterceptor`?
        
        **`ContinuationInterceptor`는 코루틴이 중단(`suspend`)된 뒤 재개(`resume`)될 때, 그 실행이 어디서 이어질지를 결정해주는 인터페이스다**.
        
        즉, *코루틴 재개 위치(스레드나 실행 컨텍스트)를 결정하는 핵심 컨텍스트 요소*라고 보면 됨.
        
        `CoroutineDispatcher`가 이 인터페이스를 구현하고 있다. ⇒ IO, Main, Default… 등 Dispatcher들은 모두 가지고 있음.
        
- `Deferred<T>`은 `Job`이다.
- `suspend fun`이어도 `suspend`되는 부분이 없다면 협력적(취소의 전파)이지 못하게 된다.

- 문제 1
    - invokeOnCompletion과 종료/예외 - 자식 launch에서 exception이 나는 경우에도 context에 invokeOnCompletion에 종료시 처리를 넣으면 동작할까?
    
    **다음 중 invokeOnCompletion을 사용하는 목적과 동작 방식에 대한 설명으로 올바른 것을 고르시오.**
    
    A. invokeOnCompletion은 항상 자신이 붙은 Job의 자식 Job에도 자동 등록된다.
    
    B. invokeOnCompletion은 취소와 예외 구분 없이 항상 실행되며, 예외 정보는 콜백으로 전달받을 수 있다.
    
    C. invokeOnCompletion은 반드시 코루틴이 정상 종료되었을 때만 호출된다.
    
    D. invokeOnCompletion은 launch 내부에서 예외가 발생할 경우 호출되지 않는다.
    
    ```kotlin
    정답: B
    
    해설:
    	•	A ❌ 자식 Job에는 자동 등록되지 않음. 각 Job에 직접 붙여야 함
    	•	B ✅ invokeOnCompletion { cause -> ... }에서 cause가 null이면 정상 종료, null이 아니면 예외/취소 사유 전달됨
    	•	C ❌ 정상 종료뿐 아니라 취소/예외 종료에도 호출됨
    	•	D ❌ 예외가 발생하더라도 호출됨 (단, 예외 정보를 넘겨받을 수 있음)
    ```
    
- 문제 2
    
    **withTimeout으로 코루틴을 감쌌을 때, 예외 메시지가 콘솔에 출력되지 않는 주요 원인으로 가장 적절한 것은?**
    
    A. withTimeout은 내부적으로 예외를 swallow(삼킴) 하도록 설계되어 있기 때문이다.
    
    B. launch는 자식 코루틴에서 발생한 예외를 부모로 자동 전파하지 않기 때문이다.
    
    C. TimeoutCancellationException은 Exception이 아니라 Error라서 출력되지 않는다.
    
    D. runBlocking의 내부 예외 처리기가 자동으로 예외를 출력하지 않도록 막기 때문이다.
    
    - 답
        
        > 정답: B
        > 
        
        ### **해설:**
        
        - A ❌ swallow되진 않음. 단지 적절히 catch하지 않으면 출력되지 않음
        - B ✅ launch는 예외를 **자동으로 전파하지 않으며**, TimeoutCancellationException은 CancellationException의 하위이므로 **기본적으로 무시됨**
        - C ❌ TimeoutCancellationException은 Exception의 하위이며, CancellationException 기반
        - D ❌ runBlocking은 최상위 예외를 포착해 출력할 수 있지만, 이 예외는 그 바깥 scope에서 처리되지 않았기 때문에 무시됨
        
        ---
        
        - SuperVisorJob의 경우
            
            ```kotlin
            import kotlinx.coroutines.*
            
            fun main() = runBlocking {
                val supervisor = SupervisorJob()
                val scope = CoroutineScope(supervisor + Dispatchers.Default)
            
                // 첫 번째 자식: 예외 발생
                val job1 = scope.launch {
                    delay(100)
                    println("job1: throwing exception")
                    throw RuntimeException("job1 failed!")
                }
            
                // 두 번째 자식: 지연 후 실행
                val job2 = scope.launch {
                    try {
                        delay(500)
                        println("job2: completed successfully")
                    } catch (e: Exception) {
                        println("job2: got cancelled due to ${e.message}")
                    }
                }
            
                joinAll(job1, job2)
            }
            ```
            
        
        ---
        
        ## **왜 withTimeout 에서 예외가 나는데 왜 아무 예외가 안 보일까?**
        
        그 이유는 **예외가 특정 코루틴 내부에서 발생했고, launch는 예외를 자동으로 전파하지 않기 때문이다**. 
        
        - `withTimeout`은 `TimeoutCancellationException`을 **던져서 코루틴을 취소**함
        - 이건 `launch` 내부에서 발생한 예외이기 때문에, 부모 코루틴이 `supervisorScope`가 아닌 이상 **부모에게 예외를 전파하고 전체 scope를 취소함**
        - 그런데 `runBlocking` 바깥에선 예외를 *잡지 않았고*, 그게 `CancellationException`류이기 때문에 **기본적으로 무시되고 출력되지 않음**
        
        즉, 예외가 발생했지만 catch하지 않으면 아무것도 안 나올 수 있음.
        
        참고 : https://kimansu.medium.com/cancellationexception-%EC%9E%A1%EC%A7%80-%EB%A7%90%EA%B3%A0-%EB%8D%98%EC%A7%80%EC%84%B8%EC%9A%94-1599864cbf2c
