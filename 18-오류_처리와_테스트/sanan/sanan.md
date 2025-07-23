- 요약
    - **일반적인 예외 처리**는 단일 코루틴 내에서는 일반 코드처럼 try-catch로 처리할 수 있다.
    - **코루틴 경계를 넘는 예외**는 구조화된 동시성 원칙에 따라 **부모 및 형제 코루틴 모두를 취소**시킨다.
    - `supervisorScope`나 `SupervisorJob`은 **자식 코루틴의 실패가 다른 자식에게 영향을 주지 않도록** 해준다. 이 스코프에서는 예외가 **부모에게 전파되지 않는다.**
    - async로 생성한 코루틴에서 예외가 발생하면, **`await()` 호출 시점에 예외가 다시 던져진다.**
    - **슈퍼바이저 스코프**는 주로 앱의 **장기 실행 컴포넌트**에 사용되며, 안드로이드의 `Application` 클래스 같은 프레임워크 요소에 내장되기도 한다.
    - **예외 전파의 최종 지점**에서는 `CoroutineExceptionHandler`가 예외를 수신하게 되며, 이 핸들러가 **정의되지 않으면 시스템 기본 핸들러로 전달된다.**
        - JVM에서는 콘솔 로그 출력
        - Android에서는 앱 크래시 유발
    - `CoroutineExceptionHandler`는 **`launch`로 시작된 루트 코루틴에서만 호출**되며, `async`에서는 **`Deferred` 객체가 예외를 직접 처리해야** 한다.
    - **플로우의 예외 처리**:
        - collect {} 바깥을 try-catch로 감싸거나
        - catch {} 중간 연산자를 사용
        - catch는 **업스트림에서만** 발생한 예외를 처리하며, **다운스트림 예외는 처리하지 않는다.**
    - **retry 연산자**는 예외 발생 시 **전체 플로우를 처음부터 재시도**하는 방식으로, **복구 전략**을 가능하게 한다.
    - **코루틴 테스트**:
        - runTest는 **가상 시간**을 제공하여 테스트를 빠르게 실행할 수 있게 해준다.
        - `TestCoroutineScheduler`는 **가상 시간 흐름을 제어**하며, `runCurrent`, `advanceUntilIdle`로 시간 흐름을 제어 가능.
        - 테스트는 **단일 스레드 디스패처**를 사용하므로, 수동으로 **일시 중단 지점(`yield` 등)**을 제공해야 테스트가 안정적이다.
- 별도의 thread로 worker를 지정해서 작업을 수행하는 경우처럼, 해당 launch에 대해 try-catch를 한다고 에러 핸들링이 되지는 않는다.
    - 해당 launch block 내부에서 핸들링을 해줘야 한다.
- async에 대한 Job(Deferred)의 경우, eager한 실행 상태에서 exception이 발생했다면, 이후에 await을 할 때 해당 예외가 다시 던져진다.
- SupervisorJob에 대한 지정 여부는 사용자의 입장에서 원자적이어야 하는지, 개별적이어야 하는지, 혹은 부분적으로 성공해야하는지 등에 대해서 탑다운으로 파악하고 사용하는 것이 맞을 것 같다 - 구조적 동시성에 대한 활용을 알고 하자.
- 책에 나타나는 ‘코루틴 스코프를 넘는 예외’는 특정 코루틴 빌더-실행 스코프 내부에서 발생한 예외가 상위 코루틴에 영향을 끼치는 것을 의미하는 것으로 파악했다.
- `CoroutineExceptionHandler`는 각각의 코루틴의 컨텍스트에 지정할 수 있다.
    - 특정 코루틴에서 발생하면 exceptionHandler를 찾아보고 optional하게 핸들링하고, 그 상위로 넘어가서 다시 이것을 반복하는 식으로 쭉 타고 올라가게 될 줄 알았으나.. ExceptionHandler를 가진 최상위 부모를 찾을때까지 위로 올라가서 실행된다.
    - 즉, 중간 단계의 ExceptionHandler는 없다.
    - GPT 설명
        1. **자식 코루틴의 예외는 부모에게 전파됨.**
        2. 부모에 CoroutineExceptionHandler가 있으면 → **그 핸들러에서 처리되고 전파 종료.**
        3. **여러 핸들러가 있어도 오직 “예외가 처음 포착된 핸들러만” 호출됨.**
        4. **핸들러가 없는 경우** → 예외는 상위로 계속 전파되고, 마지막엔 runBlocking을 넘어서 JVM에서 종료됨.
        
        즉, **예외는 딱 한 번만 처리됨**. 여러 핸들러가 호출되는 방식은 **구조적으로 일어나지 않아.**
        

- 문제 1
    
    다음 코드의 실행 결과를 예측해보자.
    
    ```kotlin
    import kotlinx.coroutines.CoroutineExceptionHandler
    import kotlinx.coroutines.delay
    import kotlinx.coroutines.launch
    import kotlinx.coroutines.runBlocking
    import kotlin.time.Duration.Companion.seconds
    
    val RootHandler = CoroutineExceptionHandler { ctx, e ->
        println("RootHandler got $e - ${e.localizedMessage}")
    }
    
    val FirstHandler = CoroutineExceptionHandler { ctx, e ->
        println("FirstHandler got $e - ${e.localizedMessage}")
    }
    
    val ThirdHandler = CoroutineExceptionHandler { ctx, e ->
        println("ThirdHandler got $e - ${e.localizedMessage}")
    }
    
    fun main(): Unit = runBlocking {
        launch(RootHandler) {
            launch(FirstHandler) {
                launch {
                    launch(ThirdHandler) {
                        delay(0.1.seconds)
                        throw IllegalStateException("제일 하위 코루틴에서 에러가 발생!")
                    }
                }
            }
        }
    }
    ```
    
    - 답
        
        에러만 찍히고 println은 나타나지 않는다. 왜 일까?
        
        1. 코루틴에서 발생한 예외는 반드시 상위로 전파된다.
        2. 중간에 있는 exceptionHandler는 호출되지 않는다. 왜냐하면..
        3. coroutineExceptionHandler는 최상위(Root) 코루틴에서 지정되어 한번만 핸들링하기 때문이다.
            
            ![image.png](attachment:18b9e020-6e7f-4403-9672-0264763538df:image.png)
            
        
- 문제 2
    
    다음 코드의 실행결과를 예측해보자.
    
    ```kotlin
    import kotlinx.coroutines.CoroutineExceptionHandler
    import kotlinx.coroutines.delay
    import kotlinx.coroutines.launch
    import kotlinx.coroutines.runBlocking
    import kotlin.time.Duration.Companion.seconds
    
    val RootHandler = CoroutineExceptionHandler { ctx, e ->
        println("RootHandler got $e - ${e.localizedMessage}")
    }
    
    fun main(): Unit = runBlocking {
        launch(RootHandler) {
            delay(0.1.seconds)
            throw IllegalStateException("제일 하위 코루틴에서 에러가 발생!")
        }
    }
    ```
    
    - 답
        
        역시나 에러만 찍힌다. 왜?
        
        결국 최상위 코루틴은 코루틴 빌더인 runBlocking에서 컨텍스트를 가지고 있기에, runBlocking에 전달된다.
        
        그래서 runBlocking에 handler를 달아준다면..?
        
        ```kotlin
        fun main(): Unit = runBlocking(RootHandler) {
            launch {
                delay(0.1.seconds)
                throw IllegalStateException("제일 하위 코루틴에서 에러가 발생!")
            }
        }
        ```
        
        그래도 아무 일도 안 일어난다. 다해줬잖아.. 왜 안 되는데?
        
        - 사유
            
            runBlocking은 **블로킹 방식으로 실행되고**, 그 내부에서 launch한 코루틴에서 예외가 발생하면, 해당 예외는 runBlocking 스코프를 **벗어나지 못하고 스레드를 터뜨려버려**. 이게 핵심이야.
            
            그리고 **CoroutineExceptionHandler는 launch로 시작한 최상위 코루틴(즉, 부모가 없는 코루틴)에서만 예외를 잡을 수 있어.**
            
            하지만 runBlocking { launch(RootHandler) { ... } } 에서 launch는 runBlocking의 **자식**이기 때문에 **여기서의 RootHandler는 무시되는 것처럼 보일 수 있어**.
            
        
        launch로 호출된 경우에만 핸들링이 가능하다니.. 정말 그런가? 찾아보자.
        
        - 서칭
            
            ## 코루틴 Exception Handler의 동작 원리
            
            Kotlin 코루틴에서 **`CoroutineExceptionHandler`**는 **"최상위(root) 코루틴"** 혹은 **루트 스코프**의 컨텍스트에 **명시**해야만 예외 처리를 할 수 있습니다.
            
            ## 주요 특징 및 동작
            
            - **`CoroutineExceptionHandler`**는 launch로 생성된 **최상위 코루틴**(즉, 더 이상 부모가 없는 코루틴)이나, **`SupervisorJob`**이 붙은 스코프에 명시적으로 추가할 때만 동작합니다.
            - **자식 코루틴**에 Handler를 달아도, 예외가 발생하면 부모에게 예외가 전파되기 때문에 해당 자식의 Handler는 동작하지 않습니다.
            - 예를 들어, **`CoroutineScope(... + CoroutineExceptionHandler)`**에서 launch를 통해 코루틴을 만들면 root에서 Handler가 동작하며, 컨텍스트를 추가로 전달하지 않는 한 하위 launch에서는 다시 지정하지 않아도 됩니다.
            - **`async`**로 만든 Deferred는 예외를 내부적으로 보관하고 **`await()`**를 통해 호출할 때 예외가 발생합니다. 이 경우에도 Exception Handler는 동작하지 않고, 직접 try-catch로 처리해야 합니다.
        
        그래서 결론적으로는 runBlocking이 아닌 아래처럼 실행하면 핸들링이 된다.
        
        ```kotlin
        fun main() {
            CoroutineScope(Dispatchers.Default + RootHandler).launch {
                launch {
                    delay(0.1.seconds)
                    throw IllegalStateException("제일 하위 코루틴에서 에러가 발생!")
                }
            }
        
            Thread.sleep(2000L)
        }
        ```
