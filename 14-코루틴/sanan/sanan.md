- 요약
    - **동시성**은 여러 작업이 번갈아 실행되며 병렬적으로 처리되는 것처럼 보이게 하는 방식이다.
    - **병렬성**은 실제로 여러 작업이 동시에 실행되며, 멀티코어 시스템을 효율적으로 활용한다.
    - **코루틴**은 스레드 위에서 동작하는 **경량의 동시 실행 추상화**다.
    - **일시 중단 함수**는 코틀린 동시성의 핵심 요소로, 실행을 멈췄다가 다시 시작할 수 있는 함수다. 다른 코루틴이나 일시 중단 함수 안에서 호출된다.
    - **반응형 스트림, 콜백, 퓨처**와 달리, 일시 중단 함수를 사용하면 **코드가 순차적으로 보인다**, 즉 작성 방식이 기존 방식과 크게 다르지 않다.
    - **코루틴 인스턴스**는 일시 중단 가능한 계산의 실행 단위를 의미한다.
    - 코루틴은 스레드를 **블로킹하지 않으며**, 리소스를 절약하고 효율적인 처리를 가능하게 한다.
    - **runBlocking, launch, async** 등 **코루틴 빌더**를 사용해 새로운 코루틴을 생성할 수 있다.
    - *디스패처(Dispatcher)**는 코루틴이 실행될 스레드나 스레드 풀을 지정하는 요소다.
    - 기본 제공 디스패처:
        - Dispatchers.Default: 일반적인 CPU 중심 작업
        - Dispatchers.Main: UI 스레드에서 작업
        - Dispatchers.IO: 블로킹 I/O 작업
    - 대부분의 디스패처는 **다중 스레드를 사용**하므로, 여러 코루틴이 **같은 데이터를 병렬로 접근할 경우 주의**가 필요하다.
    - 코루틴 생성 시 디스패처를 직접 지정하거나, 실행 중에 **withContext**로 디스패처를 변경할 수 있다.
    - 코루틴 **컨텍스트(Context)**는 디스패처를 포함한 추가적인 정보를 담고 있는 구조이며, 각 코루틴과 연관된다.
- 코루틴에서 async를 사용하는 경우 이 job의 실행시점 여부를 start(enum : `CoroutineStart`) 파라미터로 전달해줄 수 있다. default는 `Default`로, `Lazy`로 하는 경우에 async job자체를 await이 호출되는 시점부터 사용한다.
- 본질적으로 코루틴은 특정 스레드에서 고정 실행되지 않는다.
- `Dispatcher`는 코루틴을 어떤 스레드-스레드 풀에서 사용할지 지정할 수 있게 한다.
    - `Dispatcher.Default`
        
        CPU 코어 수 만큼의 스레드 풀을 갖는다. 특별히 코루틴의 스레드풀을 지정할 일이 없다면 사용한다.
        
    - `Dispatchers.Main`
        - OS에서 특별히 관리하는 UI 전용 스레드로, UI의 일관성과 안정성을 위해 반드시 별도로 사용한다. 주로 UI와 관련된 프레임워크에서 사용한다.
- 질문
    
    <aside>
    ❓
    
    **coroutine dispatcher가 제공하는 thread pool은 default, main, io 각각 결국에는 같은 OS의 스레드를 할당받는 것 아니야? 그럼에도 Main을 주로 UI-Main Thread로 두어서 하나의 스레드로 사용할 수 있게하는 데에는 이유가 뭐야? 시스템/OS 적으로 스레드 리소스에 차이가 있는거야 아니면 사용하는 방법론적인 부분에 따라 분류하는거야**
    
    </aside>
    
    <aside>
    📌
    
    ## 1. Dispatcher별 Thread Pool 구조
    
    - **Dispatchers.Default**: CPU 연산에 최적화된 스레드 풀을 사용하며, 기본적으로 CPU 코어 수만큼의 스레드를 할당합니다. CPU 바운드 작업(예: 데이터 파싱, 정렬 등)에 적합합니다[1](https://www.baeldung.com/kotlin/io-and-default-dispatcher)[2](https://stackoverflow.com/questions/59039991/difference-between-usage-of-dispatcher-io-and-default)[3](https://blog.qubartech.com/what-is-dispatcher-thread-and-threadpool).
    - **Dispatchers.IO**: I/O 작업(네트워크, 파일 읽기/쓰기 등)에 최적화된 스레드 풀을 사용합니다. Default보다 훨씬 많은(최소 `64`개) 스레드를 사용할 수 있어, 블로킹이 많은 작업에 적합합니다. 실제로 IO와 Default는 내부적으로 스레드 풀을 공유하지만, IO는 더 많은 동시 작업을 허용하도록 설계되어 있습니다[1](https://www.baeldung.com/kotlin/io-and-default-dispatcher)[4](https://www.droidcon.com/2024/11/21/exploring-the-secrets-of-dispatchers-default-and-io-in-kotlin-coroutines/)[2](https://stackoverflow.com/questions/59039991/difference-between-usage-of-dispatcher-io-and-default)[3](https://blog.qubartech.com/what-is-dispatcher-thread-and-threadpool).
    - **Dispatchers.Main**: UI 프레임워크(예: Android, JavaFX 등)에서 제공하는 메인(UI) 스레드에 바인딩됩니다. 이 스레드는 오직 하나이며, UI 갱신 및 사용자 이벤트 처리를 담당합니다[5](https://developer.android.com/kotlin/coroutines/coroutines-adv)[6](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)[7](https://github.com/kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md).
    
    ## 2. OS 스레드 리소스 관점
    
    - **Default/IO**: 둘 다 OS의 백그라운드 스레드 풀을 사용합니다. 실제로 IO와 Default는 같은 OS 스레드 풀을 공유하며, 필요에 따라 스레드 수와 동작 방식만 다릅니다. 즉, OS 레벨에서 보면 같은 종류의 스레드를 사용합니다[1](https://www.baeldung.com/kotlin/io-and-default-dispatcher)[2](https://stackoverflow.com/questions/59039991/difference-between-usage-of-dispatcher-io-and-default)[8](https://manuelvivo.dev/coroutines-and-threads)[3](https://blog.qubartech.com/what-is-dispatcher-thread-and-threadpool).
    - **Main**: OS에서 특별히 지정된 UI 스레드(예: Android의 Main Looper, JavaFX의 Application Thread 등)를 사용합니다. 이 스레드는 앱 전체에서 단 하나만 존재하며, UI 관련 작업만 처리합니다[5](https://developer.android.com/kotlin/coroutines/coroutines-adv)[9](https://stackoverflow.com/questions/71221947/why-can-you-run-a-kotlin-coroutine-on-the-main-thread)[7](https://github.com/kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md).
    
    ## 3. Main Dispatcher를 UI 전용으로 사용하는 이유
    
    - **UI 일관성 보장**: UI 프레임워크는 UI 요소의 변경이 반드시 메인 스레드에서만 일어나도록 강제합니다. 여러 스레드에서 UI를 동시에 변경하면 race condition, deadlock, 예기치 않은 동작이 발생할 수 있습니다[5](https://developer.android.com/kotlin/coroutines/coroutines-adv)[9](https://stackoverflow.com/questions/71221947/why-can-you-run-a-kotlin-coroutine-on-the-main-thread)[7](https://github.com/kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md).
    - **이벤트 루프 구조**: 메인 스레드는 이벤트 루프를 돌며, 사용자 입력, 화면 갱신 등 모든 UI 관련 작업을 순차적으로 처리합니다. 이 구조 덕분에 UI가 일관되고 예측 가능하게 동작합니다[9](https://stackoverflow.com/questions/71221947/why-can-you-run-a-kotlin-coroutine-on-the-main-thread)[7](https://github.com/kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md).
    - **코루틴의 장점**: 코루틴을 사용하면 메인 스레드에서 suspend(일시 중단)와 resume(재개)이 가능해, UI를 블로킹하지 않고도 비동기 작업을 쉽게 처리할 수 있습니다. 예를 들어, delay나 네트워크 요청 중에도 UI가 멈추지 않습니다[5](https://developer.android.com/kotlin/coroutines/coroutines-adv)[7](https://github.com/kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md).
    
    ## 4. 시스템/OS적 차이 vs 방법론적 분류
    
    - **OS적 차이**: Main은 OS에서 특별히 관리하는 UI 스레드이고, Default/IO는 일반적인 백그라운드 스레드 풀을 사용합니다. 하지만 Default와 IO는 OS 관점에서 특별한 차이가 없습니다. Main만이 특별한 스레드입니다[5](https://developer.android.com/kotlin/coroutines/coroutines-adv)[6](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)[3](https://blog.qubartech.com/what-is-dispatcher-thread-and-threadpool).
    - **방법론적 분류**: 실제로는 사용하는 목적(=방법론)에 따라 dispatcher를 분리합니다. UI 작업은 Main, CPU 바운드는 Default, I/O 바운드는 IO로 명확히 나누는 것이 코드의 안정성과 성능, 유지보수성에 유리합니다[1](https://www.baeldung.com/kotlin/io-and-default-dispatcher)[5](https://developer.android.com/kotlin/coroutines/coroutines-adv)[3](https://blog.qubartech.com/what-is-dispatcher-thread-and-threadpool).
    
    ## 5. 요약
    
    - Default와 IO는 같은 종류의 OS 스레드를 사용하지만, 목적과 동작 방식(스레드 풀 크기 등)이 다릅니다.
    - Main은 OS에서 특별히 관리하는 UI 전용 스레드로, UI의 일관성과 안정성을 위해 반드시 별도로 사용해야 합니다.
    - Dispatcher의 분류는 시스템적 차이(Main vs 나머지)와 방법론적 목적(작업 종류별 분리) 모두에 기반합니다[1](https://www.baeldung.com/kotlin/io-and-default-dispatcher)[5](https://developer.android.com/kotlin/coroutines/coroutines-adv)[3](https://blog.qubartech.com/what-is-dispatcher-thread-and-threadpool).
    
    ### 추가
    
    **Main thread는 OS에서 특별히 지정한 스레드가아니고 OS가 할당해준 스레드에서 어플리케이션 내부에서 특정한 목적(단일 스레드 사용)을 위해 지정한 것아니야?**
    
    → 정확히 말하면, **메인 스레드는 OS가 애플리케이션 프로세스를 시작할 때 자동으로 생성해서 할당해주는 첫 번째(=기본) 스레드**입니다. 이 스레드는 프로세스가 시작되는 시점에 main 함수(또는 진입점)를 실행하는 주체가 되며, 모든 애플리케이션에 반드시 최소 하나 이상 존재합니다[4](https://recipes4dev.tistory.com/143)[5](https://hosyonty.tistory.com/entry/Process-And-Thread)[9](https://myroute.tistory.com/pages/android-thread).
    
    그러나 **메인 스레드가 "특별히 지정된 스레드"냐**에 대해서는 두 가지 관점이 있습니다.
    
    - **OS 관점**: OS는 "메인 스레드"라는 이름으로 특별히 관리하지 않습니다. OS가 할당하는 것은 일반적인 스레드이며, 단순히 프로세스가 시작될 때 첫 번째로 생성되는 스레드입니다. 즉, OS 입장에서는 단순히 "스레드"입니다.
    - **애플리케이션/프레임워크 관점**: 안드로이드, JavaFX, iOS 등 대부분의 UI 프레임워크는 이 첫 번째 스레드를 "메인 스레드" 또는 "UI 스레드"로 지정하여 UI 관련 작업만 처리하도록 강제합니다. 이는 애플리케이션(또는 프레임워크) 내부에서 정해진 규칙입니다. 즉, OS가 할당해준 스레드를 프레임워크가 특정 목적(UI, 메인 이벤트 루프 등)으로 사용하는 것입니다[4](https://recipes4dev.tistory.com/143)[6](https://jungwoon.github.io/android/2019/07/16/Process-Thread.html)[7](https://yoou-yoou.tistory.com/81).
    
    따라서,
    
    **메인 스레드는 OS가 일반적으로 할당해준 스레드 중 하나이지만, 애플리케이션이나 UI 프레임워크가 그 스레드에 특별한 역할(UI, 단일 스레드 사용 등)을 부여해서 사용하는 것**이 맞습니다.
    
    즉, OS가 "메인 스레드"라는 이름으로 특별히 관리하지는 않지만, 애플리케이션이 그 스레드를 특별하게 취급해서 사용하는 구조입니다.
    
    이 구조는 안드로이드뿐 아니라 대부분의 데스크톱, 모바일 UI 프레임워크에서 공통적으로 적용됩니다.
    
    ---
    
    </aside>
    
- 코루틴을 통한 병렬 실행 - 동시성 문제와 해결방안 : [참고(코틀린 공식문서)](https://kotlinlang.org/docs/shared-mutable-state-and-concurrency.html#volatiles-are-of-no-help)
    - Volatile은 도움이 되지 않는다.
        - `@Volatile` 어노테이션은 가시성 문제(CPU 캐시가 아닌 메인 메모리에 변수 적재로 happened-before 관계)를 해결해주지, 원자성을 보장하지는 않는다.
            
            → 그냥 생짜 변수를 돌리는 것 보다 예상치에 가까운 값을 보여줄 확률이 높으나, 여전히 완벽한 해결책은 되지 못한다. ~~(예제에서 한번에 예상값이 나와서 어떻게 이게 가능한지 물어봤는데 다시 돌리니까 아니었다)~~ 
            
    - Thread-safe한 데이터 구조
        - `AtomicInteger` 등은 원자성을 보장하는 `incrementAndGet`등의 연산을 통해서 동시성 문제를 해결해준다. 하지만 복잡한 상태구조 등에서는 쉽게 스케일러블하지 않다는 단점이 있다.
            - 왜 쉽게 scalable하지 않다는거지?
                
                Atomic 연산은 "단일 메모리 위치에 대한 저수준 원자성"만 제공합니다. 복잡한 비즈니스 로직에는 **락, 액터, STM** 등 상위 추상화 도구가 필수적이며, 이는 Java의 **`synchronized`**, Kotlin의 **`Mutex`**, Go의 **`Mutex`**, Erlang의 액터 모델 등으로 구현됩니다
                
    - Thread confinement coarse-grained : 스레드 컨텍스트 제한하기
        - 특정 연산(변수 참조)의 해당하는 부분에 대해서 해당 코루틴의 context를 하나의 스레드로 제한하여 해당 연산이 진행되는 부분을 특정 스레드에서만 진행하도록 한정하여 해결한다.
            
            ```kotlin
            val counterContext = newSingleThreadContext("CounterContext")
            var counter = 0
            
            fun main() = runBlocking {
                // confine everything to a single-threaded context
                withContext(counterContext) {
                    massiveRun {
                        counter++
                    }
                }
                println("Counter = $counter")
            }
            ```
            
    - Mutex : 뮤텍스를 이용한 임계 구역 설정
        
        ```kotlin
        val mutex = Mutex()
        var counter = 0
        
        fun main() = runBlocking {
            withContext(Dispatchers.Default) {
                massiveRun {
                    // protect each increment with lock
                    mutex.withLock {
                        counter++
                    }
                }
            }
            println("Counter = $counter")
        }
        ```
        
- 문제 1
    
    **다음 중 Kotlin 코루틴에서 동시성 문제 해결을 위한 방식으로 적절하지 않은 것은 무엇인가?**
    
    A. Mutex를 활용하여 임계 구역을 설정한다.
    
    B. AtomicInteger를 사용하여 공유 상태를 원자적으로 갱신한다.
    
    C. @Volatile 어노테이션을 사용하여 변수의 원자성을 보장한다.
    
    D. newSingleThreadContext를 사용하여 하나의 스레드에서만 변수 접근을 허용한다.
    
    - 답
        
        C. @Volatile은 메모리 가시성을 보장하지만, **원자성(atomicity)** 은 보장하지 않는다. 따라서 동시성 문제 해결 수단으로는 부적절하다.
        
- 문제 2
    
    다음 중 **Kotlin의 코루틴(Coroutine)** 에 대한 설명으로 **가장 적절한 것**은 무엇인가?
    
    A. 코루틴은 JVM의 스레드와 1:1로 매핑되어 병렬 처리를 수행한다.
    
    B. 코루틴은 백그라운드에서 영구적으로 실행되는 스레드 풀을 생성하는 기법이다.
    
    C. 코루틴은 비동기 처리를 위해 Kotlin에서 제공하는 **경량 스레드**로, **중단 가능한(suspendable)** 비동기 흐름을 표현할 수 있다.
    
    D. 코루틴은 모든 작업이 시작부터 끝까지 **Main Thread에서만** 실행되도록 보장한다.
    
    - 답
        
        C. 코루틴은 Kotlin에서 제공하는 경량 스레드(lightweight thread) 개념으로, 기존 스레드보다 메모리 소모가 적고 수천 개 이상 생성해도 효율적이다. suspend 키워드를 통해 중단 가능한 비동기 처리 흐름을 표현할 수 있고, 스레드와 직접적으로 1:1로 매핑되지 않으며 유연하게 컨텍스트를 전환할 수 있다.
