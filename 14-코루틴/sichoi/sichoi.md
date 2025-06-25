# 동시성(Concurrency)과 병렬성(parallelism)

| 개념 | 동시성 (Concurrency) | 병렬성 (Parallelism) |
| --- | --- | --- |
| 의미 | 여러 작업을 **같이 다루는 전략** | 여러 작업을 **진짜 물리적으로 동시에 실행** |
| 초점 | **작업의 설계와 전환** | **물리적인 동시 실행** |
| 실행 방식 | 번갈아가며 실행 (인터리빙) 가능 | 동시에 실행 (멀티코어 등) 필요 |
| 사용 목적 | **응답성**, 구조적 설계 | **성능 최적화**, 처리 속도 향상 |
- **동시성**:
    - 싱글코어에서 여러 네트워크 요청 처리 (Node.js, Python asyncio)
    - UI 스레드는 그리기, 백그라운드는 다운로드
- **병렬성**:
    - 멀티코어 CPU가 이미지 필터를 여러 조각으로 나눠 동시에 처리
    - 과학 계산을 다수의 코어에서 동시에 수행

<aside>
💡

여기서 말하는 동시성: “여러 작업을 번갈아가며 진행하며 동시처럼 보이게 하는 것”

사전적 의미로의 동시가 아님을 주의해야 함.

</aside>

# 스레드와 코루틴 비교

## 스레드 (Thread)

### 정의

JVM에서 병렬성과 동시성을 구현하는 전통적인 방식으로, 각 스레드는 운영체제에서 관리되는 **독립 실행 단위**다.

### 사용 예시

```kotlin
import kotlin.concurrent.thread

fun main() {
    println("I'm on ${Thread.currentThread().name}")
    thread {
        println("And I'm on ${Thread.currentThread().name}")
    }
}
```

### 장점

- 멀티코어 CPU 활용 가능 (병렬성)
- 자바 생태계와 완전 호환

### 단점

- 생성 비용이 크고 수천 개 수준이 한계
- 스레드당 수 MB 메모리 소비
- I/O 대기 중 블로킹 발생 (자원 낭비)
- 스레드 간 전환 비용 큼 (커널 작업)
- 계층적 구조 없음 → 취소/오류 처리 어려움

## 코루틴 (Coroutine)

코틀린이 제공하는 **경량 동시성 추상화**로, 일시 중단이 가능한 코드 블록. 스레드처럼 동시성 작업을 나타내지만 훨씬 가볍고 효율적이다.

### 장점

- 수십만 개의 코루틴 실행 가능
- 생성/관리 비용 매우 낮음
- 블로킹 없이 일시 중단 및 재개 가능
- 구조화된 동시성 제공 (작업 계층, 오류 전파, 자동 취소)
- 짧고 가벼운 작업에도 부담 없이 활용 가능

### 동작 방식

- 실제로는 하나 이상의 JVM 스레드 위에서 실행됨
- 운영체제의 스레드 한계에 구애받지 않음

## 스레드 vs 코루틴 비교

| 항목 | 스레드 | 코루틴 |
| --- | --- | --- |
| 실행 단위 | 운영체제 스레드 | 언어 수준의 경량 단위 |
| 생성 비용 | 큼 | 작음 |
| 실행 수 | 수천 개 제한 | 수십만 개 가능 |
| 블로킹 | 있음 | 없음 |
| 전환 비용 | 운영체제 수준 (무거움) | 언어 수준 (가벼움) |
| 계층 구조 | 없음 | 있음 (구조화된 동시성) |
| 예외 처리 | 복잡하고 분산됨 | 일관적이고 안전함 |

<img width="562" alt="image" src="https://github.com/user-attachments/assets/3acc5458-0b02-4e02-90da-7a199369e118" />




## Project Loom과의 비교

### Project Loom 소개

JVM에 가상 스레드를 도입하여 기존 스레드 API를 유지하면서도 경량 동시성을 제공하려는 실험적 프로젝트.

| 항목 | 코루틴 | Project Loom |
| --- | --- | --- |
| 추상화 위치 | 언어 수준 (네이티브 등 다른 플랫폼 사용 가능) | JVM 수준 |
| 구조화된 동시성 | 기본 설계 요소 | 실험적 도입 중 |
| I/O 작업 처리 | `suspend`로 명확히 표현 | 명시적 구분 없음 |
| 레거시 호환성 | 적음 (새로운 스타일 지향) | 높음 (기존 코드 포팅 유리) |
| 경량성 및 효율 | 매우 우수 | 상대적으로 제한적 |
- 코루틴은 구조화된 동시성과 비용 효율성을 중심으로 설계됨
- 룸은 기존 자바 코드를 가볍게 실행하는 데 집중
- 코루틴은 언어 설계부터 동시성 코드의 안정성을 고려함

# 잠시 멈출 수 있는 함수: 일시 중단 함수

- 코루틴에서 `suspend` 키워드로 선언되는 함수.
- 실행을 **잠시 중단**했다가 나중에 **중단된 지점부터 다시 재개**할 수 있음.
- **스레드를 블로킹하지 않고도** 기다릴 수 있음.
- 코드 구조는 여전히 **순차적으로 보이고 작성 가능**함.

## 일시 중단 함수 사용 전: 블로킹 방식

```kotlin
fun login(credentials: Credentials): UserID
fun loadUserData(userID: UserID): UserData
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
    val userID = login(credentials)
    val userData = loadUserData(userID)
    showData(userData)
}
```

- 이 코드는 **순차적으로 보이지만**, 각 단계에서 **스레드를 블로킹**함.
- 예: `login()`과 `loadUserData()`는 네트워크 요청 대기 중 스레드를 차지함.
- 스레드가 수천 개로 제한된 환경에서, 이런 블로킹은 시스템 자원 낭비와 성능 저하를 초래.
- UI 스레드에서 호출하면 인터페이스가 멈추는 현상이 발생할 수 있음.

## 일시 중단 함수 사용 후: 넌블로킹 방식

```kotlin
suspend fun login(credentials: Credentials): UserID
suspend fun loadUserData(userID: UserID): UserData
fun showData(data: UserData)

suspend fun showUserInfo(credentials: Credentials) {
    val userID = login(credentials)
    val userData = loadUserData(userID)
    showData(userData)
}
```

- `suspend` 키워드를 통해 함수가 **중단 가능한 지점**이 됨.
- 네트워크 요청 도중에 실행이 일시 중단되며, **같은 스레드에서 다른 작업이 실행될 수 있음**.
- UI 스레드에서 실행해도 화면이 멈추지 않음.

## 구조 유지의 장점

- 코드 구조가 여전히 **순차적**임.
- 복잡한 **콜백 지옥이나 상태 머신** 없이도 자연스러운 흐름 구현 가능.
    ![image](https://github.com/user-attachments/assets/0bc6210b-7ba9-41c6-8afb-130fa677d2fd)

    
- 대기 중인 동안에는 **다른 작업(UI 업데이트 등)** 수행 가능.

## 작동 조건

- `login`, `loadUserData` 같은 일시 중단 함수들도 코루틴 친화적으로 구현되어 있어야 함.
- 코틀린 생태계의 주요 라이브러리들은 이를 지원:
    - `ktor`, `Retrofit`, `OkHttp` 등은 코루틴 기반 네트워크 API 제공

# 코루틴을 다른 접근 방법과 비교

## 콜백 기반 접근 방식

```kotlin
fun loginAsync(credentials: Credentials, callback: (UserID) -> Unit)
fun loadUserDataAsync(userID: UserID, callback: (UserData) -> Unit)
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
    loginAsync(credentials) { userID ->
        loadUserDataAsync(userID) { userData ->
            showData(userData)
        }
    }
}
```

- **콜백 중첩 발생**: 코드가 중첩되고 가독성이 떨어짐
- 복잡한 로직일수록 **관리 어려움**
- 일명 **"콜백 지옥(callback hell)"** 현상

## `CompletableFuture`를 사용한 접근 방식

```kotlin
fun loginAsync(credentials: Credentials): CompletableFuture<UserID>
fun loadUserDataAsync(userID: UserID): CompletableFuture<UserData>
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
    loginAsync(credentials)
        .thenCompose { loadUserDataAsync(it) }
        .thenAccept { showData(it) }
}
```

- **콜백 중첩 해결**
- 하지만 `thenCompose`, `thenAccept` 등 **새로운 연산자 학습 필요**
- 반환 타입이 `CompletableFuture`로 **복잡성 증가**

## 반응형 스트림 (`RxJava` 등)

```kotlin
fun login(credentials: Credentials): Single<UserID>
fun loadUserData(userID: UserID): Single<UserData>
fun showData(data: UserData)

fun showUserInfo(credentials: Credentials) {
    login(credentials)
        .flatMap { loadUserData(it) }
        .doOnSuccess { showData(it) }
        .subscribe()
}
```

- **콜백 중첩 없음**
- 연산자 조합 (`flatMap`, `subscribe`)에 대한 **이해와 숙련 필요**
- 반환 타입이 `Single`, `Observable` 등으로 **변형 필요**

## 코루틴 방식의 비교 우위

- 함수 시그니처에 `suspend`만 붙이면 됨
- **기존 코드 구조 유지 가능** (순차적 형태)
- 내부적으로는 넌블로킹 동작 → 스레드 자원 효율적 사용
- 비동기 코드를 **직관적이고 읽기 쉬운 방식으로 표현 가능**

## 일시 중단 함수 호출 규칙

- `suspend` 함수는 **일시 중단 가능한 블록 안**에서만 호출 가능
- 일반 함수에서 직접 호출하면 컴파일 오류 발생

### 예시 (오류 발생)

```kotlin
suspend fun mySuspendingFunction() {}

fun main() {
    mySuspendingFunction() // 오류: suspend 함수는 suspend 블록에서만 호출 가능
}
```

### 해결 방법 1: `main` 함수를 suspend로 만들기 (작은 프로그램에서 사용)

```kotlin
suspend fun main() {
    mySuspendingFunction()
}
```

### 해결 방법 2: 코루틴 빌더 사용 (현실적인 일반적 방법)

```kotlin
fun main() = runBlocking {
    mySuspendingFunction()
}
```

- `runBlocking`, `launch`, `async` 등의 코루틴 빌더로 **코루틴 진입점 생성**

# 코루틴의 세계로 들어가기: 코루틴 빌더

## 주요 코루틴 빌더

| 빌더 | 반환값 | 사용 목적 |
| --- | --- | --- |
| `runBlocking` | 계산된 결과 | 블로킹 코드와 코루틴을 연결할 때 |
| `launch` | `Job` | 값을 반환하지 않는 작업 실행 (부수 효과 중심) |
| `async` | `Deferred<T>` | 비동기적으로 값을 계산하고 나중에 대기 (`await`) |

## runBlocking: 일반 코드에서 진입

```kotlin
fun main() = runBlocking {
    doSomethingSlowly()
}

suspend fun doSomethingSlowly() {
    delay(500)
    println("I'm done")
}
```

- `runBlocking`은 코루틴을 시작하고 **해당 스레드를 블로킹**함.
- 내부에서는 **일시 중단 함수 호출 가능**.
- UI 또는 서버 코드에서는 잘 안 쓰이고, **테스트 코드**나 **작은 유틸리티용**으로 적합.

## launch: 발사 후 망각 (fire-and-forget)

```kotlin
fun main() = runBlocking {
    launch {
        delay(100)
        println("Coroutine A done")
    }
    launch {
        println("Coroutine B starts immediately")
    }
}
```

- **자식 코루틴을 생성**해 실행.
- **결과값을 반환하지 않음**, 대신 `Job` 객체 반환.
- 주로 **로그 남기기, 저장, 전송 등의 부수 효과 작업**에 사용.
- `Job`을 통해 **취소, 완료 대기** 등을 제어할 수 있음.

<aside>
💡

"발사 후 망각” 이름이 해괴망측하지만 놀랍게도 직관적이다.

</aside>

## async: 값을 비동기로 계산하기

```kotlin
fun main() = runBlocking {
    val deferred1 = async { slowlyAdd(2, 2) }
    val deferred2 = async { slowlyAdd(4, 4) }

    println("Result 1: ${deferred1.await()}")
    println("Result 2: ${deferred2.await()}")
}

suspend fun slowlyAdd(a: Int, b: Int): Int {
    delay(100 * a.toLong())
    return a + b
}
```

- `async`는 코루틴을 시작하고, **Deferred** 객체를 반환.
- `await()`를 통해 **결과값을 일시 중단 방식으로 기다림**.
- 여러 계산을 동시에 시작하고, 결과를 나중에 취합할 때 적합.

## async vs 일반 일시 중단 함수 호출

- `async/await`는 **병렬로 수행할 때만 필요**.
- 순차적으로 호출할 경우에는 `suspend` 함수만으로 충분.

```kotlin
// 순차 실행
val x = slowlyAdd(2, 2)
val y = slowlyAdd(4, 4)

// 병렬 실행
val deferredX = async { slowlyAdd(2, 2) }
val deferredY = async { slowlyAdd(4, 4) }
val result = deferredX.await() + deferredY.await()
```

## 일시 중단과 재개는 어떻게 작동할까?

- **컴파일러가 일시 중단 가능한 코드를 상태 머신으로 변환**함.
- 중단 시점의 정보(변수, 위치 등)를 메모리에 저장해두고,
- 나중에 **그 위치부터 재개**할 수 있도록 지원.

| 빌더 | 반환값 | 사용 상황 |
| --- | --- | --- |
| `runBlocking` | 값 | 코루틴 진입점 (main, 테스트 등) |
| `launch` | `Job` | 결과를 기다리지 않는 실행 (부수 효과 중심) |
| `async` | `Deferred<T>` | 비동기 연산 및 결과 수집 |

# 어디서 코드를 실행할지 정하기: 디스패처

## 디스패처란?

- 코루틴을 **어떤 스레드 혹은 스레드 풀에서 실행할지 결정**하는 컴포넌트.
- 코루틴은 특정 스레드에 고정되지 않으며, **실행을 일시 중단한 후 다른 스레드에서 재개**될 수도 있음.
- 디스패처를 통해 **멀티코어 병렬 실행, UI 스레드 고정, IO 최적화** 등 다양한 실행 전략을 적용할 수 있음.

## 주요 디스패처 종류

| 디스패처 | 스레드 수 | 용도 |
| --- | --- | --- |
| `Dispatchers.Default` | CPU 코어 수 | 일반 계산, CPU 집약 작업 |
| `Dispatchers.IO` | 최대 64개 + CPU 수 | 블로킹 IO, 파일/네트워크 작업 |
| `Dispatchers.Main` | 1 (UI 프레임워크 기반) | UI 업데이트 등 메인 스레드 작업 |
| `Dispatchers.Unconfined` | 제한 없음 | 즉시 실행 또는 호출자 스레드에서 실행 (특수한 경우) |
| `limitedParallelism(n)` | 사용자 지정 | 병렬성 제한이 필요한 경우 |

> 코루틴을 명시적으로 디스패처에 지정하지 않으면 부모 코루틴의 디스패처를 상속받음.
> 

## 디스패처 지정 방법

### 코루틴 빌더에 직접 전달

```kotlin
runBlocking {
    launch(Dispatchers.Default) {
        println("Running on background thread")
    }
}
```

### withContext로 전환

```kotlin
launch(Dispatchers.Default) {
    val result = performBackgroundOperation()
    withContext(Dispatchers.Main) {
        updateUI(result)
    }
}
```

- `withContext`는 코루틴 **내부에서 디스패처를 전환**할 수 있게 함.
- UI 업데이트처럼 특정 스레드에서만 동작해야 하는 작업을 분리할 때 유용.

## 디스패처 선택 가이드

```
작업이 UI 관련인가?
 → Dispatchers.Main

블로킹 API 사용 예정인가?
 → Dispatchers.IO

그 외 대부분의 일반 작업인가?
 → Dispatchers.Default

특수 목적 또는 실험적?
 → Dispatchers.Unconfined / limitedParallelism(n)
```

## 스레드 안전성과 디스패처

- **단일 코루틴** 내부에서는 병렬 실행이 없기 때문에 일반적으로 스레드 안전 문제는 발생하지 않음.
- **여러 코루틴이 동일한 데이터에 접근**하는 경우에는 경쟁 조건이 발생할 수 있음.

### 예: 문제 없는 경우 (단일 코루틴)

```kotlin
launch(Dispatchers.Default) {
    var x = 0
    repeat(10000) {
        x++
    }
    println(x) // 항상 10000
}

```

### 예: 문제 발생 (여러 코루틴이 공유 변수 증가)

```kotlin
var x = 0
repeat(10000) {
    launch(Dispatchers.Default) {
        x++ // 경쟁 조건 발생 가능
    }
}
```

### 해결 방법: `Mutex` 사용

```kotlin
val mutex = Mutex()
var x = 0
repeat(10000) {
    launch(Dispatchers.Default) {
        mutex.withLock {
            x++
        }
    }
}
```

### 대안: `AtomicInteger`, `ConcurrentHashMap` 등 **스레드 안전한 구조 사용**도 가능.

# 코루틴은 코루틴 콘텍스트에 추가적인 정보를 담고 있다

## CoroutineContext란?

- **코루틴의 실행에 영향을 미치는 정보들의 집합**.
- `CoroutineContext`는 단순히 디스패처만 포함하는 것이 아니라, 다음과 같은 다양한 요소들을 포함할 수 있음:

| 요소 | 설명 |
| --- | --- |
| `CoroutineDispatcher` | 코루틴이 실행될 스레드를 결정 |
| `Job` | 코루틴의 생명주기와 취소 관리 |
| `CoroutineName` | 디버깅 및 로깅 시 유용한 코루틴 이름 |
| `CoroutineExceptionHandler` | 예외 처리 로직 지정 가능 |

## 현재 콘텍스트 확인하기

```kotlin
import kotlin.coroutines.coroutineContext

suspend fun introspect() {
    log(coroutineContext)
}

fun main() = runBlocking {
    introspect()
}
```

- 일시 중단 함수 안에서 `coroutineContext` 속성에 접근하면, **현재 코루틴의 콘텍스트 정보**를 확인할 수 있음.

```
[CoroutineId(1), "coroutine#1": BlockingCoroutine{Active}, BlockingEventLoop@...]
```

- `coroutineContext`는 특별한 예약어는 아니지만, 컴파일러가 자동으로 처리하는 **빌트인 프리미티브**임.

## 코루틴 콘텍스트 확장

- `runBlocking`, `launch`, `async`, `withContext` 등은 `CoroutineContext`를 파라미터로 받을 수 있음.
- 여러 요소를 함께 지정하려면 `+` 연산자로 연결 가능.

### 예제: 이름 + 디스패처 지정

```kotlin
fun main() = runBlocking(Dispatchers.IO + CoroutineName("Coolroutine")) {
    introspect()
}
```

출력 예시:

```
[CoroutineName(Coolroutine), CoroutineId(1), "Coolroutine#1": BlockingCoroutine{Active}, Dispatchers.IO]
```

- `Dispatchers.IO`: 실행 위치를 IO 디스패처로 설정
- `CoroutineName`: 디버깅 시 코루틴 이름을 명시

## 덮어쓰기 규칙

- 자식 코루틴은 부모 코루틴의 콘텍스트를 **상속**함.
- 명시적으로 전달된 콘텍스트 요소는 상속된 값을 **덮어씀**.
- 예: 부모 코루틴이 `Dispatchers.Default`를 사용하더라도, 자식에서 `Dispatchers.IO`를 지정하면 그 디스패처가 우선됨.
