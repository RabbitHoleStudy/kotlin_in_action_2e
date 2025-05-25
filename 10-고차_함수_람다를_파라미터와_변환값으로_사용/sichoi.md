# 고차 함수란?

- 다른 함수를 **인자로 받거나**, **함수 자체를 반환**하는 함수
- 코틀린에서는 **람다**나 **함수 참조**를 값처럼 다룰 수 있으므로 고차 함수 정의가 자연스럽다

예:

```kotlin
list.filter { it > 0 } // filter는 술어 함수를 받는 고차 함수
```

## 람다를 인자로 받는 함수: 고차 함수의 파마티터 타입과 반환 타입

### 함수 타입

함수 타입은 다음과 같은 형식으로 표현된다:

```
(파라미터 타입들) -> 반환 타입
```

예:

```kotlin
val sum: (Int, Int) -> Int = { x, y -> x + y }
val action: () -> Unit = { println("Hello") }
```

- `Unit`은 반환값이 없는 함수에서 사용된다
- 파라미터가 없으면 `()`로 나타낸다

람다의 파라미터 타입은 명시하지 않아도 변수 타입에서 유추할 수 있다:

```kotlin
val sum: (Int, Int) -> Int = { x, y -> x + y } // x, y 타입 생략 가능
```

### **함수 타입의 널 가능성**

두 가지 구분이 필요하다:

1. 함수의 반환 타입이 `null`일 수 있는 경우:
    
    ```kotlin
    var canReturnNull: (Int, Int) -> Int? = { _, _ -> null }
    ```
    
2. 함수 자체가 `null`일 수 있는 경우:
    
    ```kotlin
    var funOrNull: ((Int, Int) -> Int)? = null
    ```
    

괄호 유무에 따라 의미가 바뀌므로 주의가 필요하다.

## 인자로 전달 받은 함수 호출

- **고차 함수 정의 예시**
    
    ```kotlin
    fun twoAndThree(operation: (Int, Int) -> Int) {
        val result = operation(2, 3)
        println("The result is $result")
    }
    
    fun main() {
        twoAndThree { a, b -> a + b }  // The result is 5
        twoAndThree { a, b -> a * b }  // The result is 6
    }
    ```
    
- **함수 타입 파라미터의 이름 지정**
    
    ```kotlin
    fun twoAndThree(
        operation: (operandA: Int, operandB: Int) -> Int
    ) { ... }
    ```
    
    - 이름은 가독성을 높이고, IDE에서 자동 완성 도움을 준다.
    - 하지만 **함수 타입의 파라미터 이름은 타입 검사에서는 무시된다.**

## 자바에서 코틀린 함수 타입 사용

- 코틀린의 함수 타입은 자바에서 **SAM 변환**을 통해 람다로 전달 가능.
    
    ```kotlin
    fun processTheAnswer(f: (Int) -> Int) {
        println(f(42))
    }
    ```
    
    - 자바 호출 예:
        
        ```java
        processTheAnswer(number -> number + 1);  // 43
        ```
        
- 코틀린 확장 함수도 자바에서 호출 가능하지만:
    
    ```java
    CollectionsKt.forEach(strings, s -> {
        System.out.println(s);
        return Unit.INSTANCE;  // 반드시 반환 필요
    });
    ```
    
- `Unit` 반환 함수는 자바에서 `Unit.INSTANCE`를 명시적으로 반환해야 함.
- 코틀린 내부적으로 함수 타입은 `FunctionN` 인터페이스를 구현.
    
    ```kotlin
    interface Function1<P1, out R> {
        operator fun invoke(p1: P1): R
    }
    ```
    
- 예시: 클래스가 함수 타입을 구현할 수도 있음
    
    ```kotlin
    class Adder : (Int, Int) -> Int {
        override fun invoke(p1: Int, p2: Int): Int = p1 + p2
    }
    ```
    

## 함수 타입의 파라미터에 기본값/nullable 사용

- 기본값을 제공하는 함수:
    
    ```kotlin
    fun <T> Collection<T>.joinToString(
        separator: String = ", ",
        prefix: String = "",
        postfix: String = "",
        transform: (T) -> String = { it.toString() }
    ): String
    ```
    
- 사용 예:
    
    ```kotlin
    println(letters.joinToString())  // 기본 toString
    println(letters.joinToString { it.lowercase() })  // 람다 전달
    ```
    
- nullable 함수 타입 사용 예:
    
    ```kotlin
    fun <T> Collection<T>.joinToString(
        ...,
        transform: ((T) -> String)? = null
    ): String {
        val str = transform?.invoke(element) ?: element.toString()
    }
    ```
    
- nullable 함수 타입은 **안전 호출 연산자 (`?.invoke()`)** 와 **엘비스 연산자 (`?:`)** 로 처리 가능.

## 함수를 반환하는 함수

- 함수가 **조건에 따라 다른 람다를 반환**하는 구조:
    
    ```kotlin
    enum class Delivery { STANDARD, EXPEDITED }
    
    fun getShippingCostCalculator(delivery: Delivery): (Order) -> Double {
        return if (delivery == Delivery.EXPEDITED)
            { order -> 6 + 2.1 * order.itemCount }
        else
            { order -> 1.2 * order.itemCount }
    }
    ```
    
- 예제:
    
    ```kotlin
    val calculator = getShippingCostCalculator(Delivery.EXPEDITED)
    println("Shipping costs ${calculator(Order(3))}")  // 12.3
    ```
    
- 실용적 예시 – UI 상태에 따라 필터링 함수 반환:
    
    ```kotlin
    class ContactListFilters {
        var prefix: String = ""
        var onlyWithPhoneNumber: Boolean = false
    
        fun getPredicate(): (Person) -> Boolean {
            val startsWithPrefix = { p: Person ->
                p.firstName.startsWith(prefix) || p.lastName.startsWith(prefix)
            }
            return if (onlyWithPhoneNumber)
                { startsWithPrefix(it) && it.phoneNumber != null }
            else
                startsWithPrefix
        }
    }
    ```
    

## 람다로 중복 제거, 코드 재사용

- 고차 함수는 중복 로직을 분리할 수 있게 도와줌.
- 데이터 클래스 예제:
    
    ```kotlin
    data class SiteVisit(val path: String, val duration: Double, val os: OS)
    enum class OS { WINDOWS, MAC, IOS, ANDROID }
    ```
    
- 하드코딩된 평균 계산:
    
    ```kotlin
    val averageWindows = log.filter { it.os == OS.WINDOWS }.map(SiteVisit::duration).average()
    ```
    
- 고차 함수 활용으로 일반화:
    
    ```kotlin
    fun List<SiteVisit>.averageDurationFor(os: OS) =
        filter { it.os == os }.map(SiteVisit::duration).average()
    ```
    
- 더 유연한 고차 함수:
    
    ```kotlin
    fun List<SiteVisit>.averageDurationFor(predicate: (SiteVisit) -> Boolean) =
        filter(predicate).map(SiteVisit::duration).average()
    ```
    
- 사용 예:
    
    ```kotlin
    log.averageDurationFor { it.os == OS.IOS && it.path == "/signup" }
    ```
    

# 인라인 함수를 사용해 람다의 부가 비용 없애기

람다는 간결한 문법을 제공하지만 내부적으로는 **익명 클래스**로 컴파일되며, **객체 생성 비용**과 **함수 호출 비용**이 추가된다. 이런 비용을 줄이기 위해 **`inline` 함수**를 사용하면 컴파일 시점에 코드가 **직접 삽입(inline)** 되어 성능을 개선할 수 있다.

## 인라이닝이 작동하는 방식

```kotlin
inline fun <T> synchronized(lock: Lock, action: () -> T): T {
    lock.lock()
    try {
        return action()
    } finally {
        lock.unlock()
    }
}
```

- `inline` 키워드를 붙이면 이 함수는 **호출 위치에 코드가 복사되어 들어간다.**
- 이때 **람다도 함께 인라이닝**된다.
- 예제:
    
    ```kotlin
    fun foo(lock: Lock) {
        println("Before sync")
        synchronized(lock) {
            println("Action")
        }
        println("After sync")
    }
    ```
    
- 위 코드는 컴파일 시 다음과 유사하게 변환됨:
    
    ```kotlin
    println("Before sync")
    lock.lock()
    try {
        println("Action")
    } finally {
        lock.unlock()
    }
    println("After sync")
    ```
    
- 람다가 **익명 클래스 객체로 감싸지지 않음** → 객체 생성 X, 함수 호출 X
- 다만 **람다를 변수에 저장**하는 경우 인라이닝이 불가능:
    
    ```kotlin
    class LockOwner(val lock: Lock) {
        fun runUnderLock(body: () -> Unit) {
            synchronized(lock, body) // 람다가 인라인되지 않음
        }
    }
    ```
    

## 인라인 함수의 제약

- 람다를 **변수에 저장하거나, 나중에 호출하는 용도**로 사용하면 인라인 불가
    
    ```kotlin
    class FunctionStorage {
        var myStoredFunction: ((Int) -> Unit)? = null
    
        inline fun storeFunction(f: (Int) -> Unit) {
            myStoredFunction = f  // 오류: 인라인 파라미터 저장 불가
        }
    }
    ```
    
- 인라인 가능:
    - 즉시 호출
    - 다른 인라인 함수에 전달
- 일부 파라미터만 인라인하려면 `noinline` 키워드 사용:
    
    ```kotlin
    inline fun foo(inlined: () -> Unit, noinline notInlined: () -> Unit) { ... }
    ```
    
- `noinline`으로 선언된 람다는 일반 함수처럼 처리되며 익명 클래스가 생성됨

## 컬렉션 연산 인라이닝

- 예:
    
    ```kotlin
    val people = listOf(Person("Alice", 29), Person("Bob", 31))
    val result = people.filter { it.age < 30 }
    println(result)  // [Person(name=Alice, age=29)]
    ```
    
- 위 `filter` 함수는 **inline 함수**이므로 내부 코드가 직접 삽입됨
- 동일 로직을 일반 for-loop로 작성한 코드와 **바이트코드 거의 동일**
- 즉, **표준 컬렉션 연산을 써도 성능 저하 없음**
- 그러나 `filter → map` 처럼 **연쇄 호출**은 중간 리스트를 생성하기 때문에 비용 증가
    
    ```kotlin
    people.filter { it.age > 30 }.map(Person::name)
    ```
    
- 이를 **지연 계산(lazy)** 하려면 `asSequence()` 사용:
    
    ```kotlin
    people.asSequence()
        .filter { it.age > 30 }
        .map(Person::name)
        .toList()
    ```
    
- 단, **시퀀스는 람다 인라이닝 불가** → 많은 요소 처리 시만 이점 있음
- **작은 컬렉션은 일반 컬렉션 연산이 오히려 빠를 수 있음**

## 언제 함수를 인라인으로 선언할지 결정

- **람다를 인자로 받는 경우** 인라이닝의 효과가 크다:
    - 객체 생성 생략
    - 함수 호출 비용 제거
    - 비로컬 return 등 람다 관련 기능 활용 가능
- 일반 함수는 JVM이 이미 JIT 인라이닝 최적화 처리함
    - 수동 인라이닝은 **코드 크기 증가**만 유발할 수 있음
- **인라인 함수 사용 가이드:**
    - 람다 인자가 있는 함수
    - 함수 본문이 작을 때
    - 호출 빈도가 높을 때
    - 성능을 실측/분석한 경우

## `withLock`, `use`, `useLines`로 자원 관리 최적화

### `withLock`: Lock 자원을 자동으로 해제

```kotlin
val lock = ReentrantLock()
lock.withLock {
    // 락 보호 하에 수행할 작업
}
```

- `Lock.withLock()` 정의:
    
    ```kotlin
    inline fun <T> Lock.withLock(action: () -> T): T {
        lock()
        try { return action() } finally { unlock() }
    }
    ```
    

### `use`: `Closeable` 자원 자동 닫기

```kotlin
fun readFirstLineFromFile(fileName: String): String {
    return BufferedReader(FileReader(fileName)).use { br ->
        br.readLine()
    }
}
```

- `use`는 **람다 종료 후 자원 자동 close**
- 람다 안의 `return`은 **비로컬 return** (호출 함수로 반환)

### `useLines`: 파일 라인을 `Sequence<String>`으로 사용

```kotlin
fun readFirstLineFromFile(fileName: String): String {
    return Path(fileName).useLines { it.first() }
}
```

### Java와의 차이

- Java의 `try-with-resources` 구문과 기능적으로 동일:
    
    ```java
    try (BufferedReader br = new BufferedReader(new FileReader(fileName))) {
        return br.readLine();
    }
    
    ```
    
- Kotlin은 **함수형 방식 (`use`)** 으로 자원 관리를 간결하게 처리

### 비로컬 return

- `inline` 함수에서 람다 안에서의 `return`은 **람다를 넘긴 함수에서 반환** 가능
- 이는 일반 람다에서는 불가능하며, **인라인 함수 + 람다 조합**일 때만 동작함

## 요약

| 개념 | 설명 |
| --- | --- |
| `inline fun` | 함수 호출 대신 본문을 코드에 직접 삽입 |
| 이점 | 객체 생성 X, 함수 호출 비용 X, 비로컬 `return` 사용 가능 |
| 제약 | 람다를 변수에 저장하면 인라인 불가 → `noinline` 사용 필요 |
| `filter/map` | inline이라 성능 좋음, 단 중간 리스트 주의 |
| `asSequence()` | 중간 리스트 제거 가능하지만 람다 인라인 불가 |
| `withLock`, `use` | 자원 획득/해제 코드 중복 제거. 안전한 자원 관리 |
| 사용 시점 | 람다 인자가 있는 짧은 함수, 반복 호출될 함수 등 성능 요구 상황 |

# 람다에서 반환: 고차 함수에서 흐름 제어

람다로 명령형 루프를 대체하다 보면 `return`의 의미에 대해 혼란이 생길 수 있다. 특히 람다 안에서의 `return`은 **람다 자신을 종료**하는지, **둘러싼 함수를 종료**하는지에 따라 의미가 달라지며, 이를 **비로컬 return**, **로컬 return**, **익명 함수 return**으로 구분해 이해해야 한다.

## 람다 안의 return 문: 람다를 둘러싼 함수에서 반환

### 개념 설명

- 람다 안의 `return`은 **람다를 감싸고 있는 함수 전체를 반환**시킬 수 있다.
- 단, **인라인 함수로 전달된 람다일 때만** 가능하다.
- 이를 **비로컬 return**이라고 한다.

### 예시 1: 일반 for-loop에서의 return

```kotlin
fun lookForAlice(people: List<Person>) {
    for (person in people) {
        if (person.name == "Alice") {
            println("Found!")
            return
        }
    }
    println("Alice is not found")
}
```

### 예시 2: `forEach` 안의 람다에서 return

```kotlin
fun lookForAlice(people: List<Person>) {
    people.forEach {
        if (it.name == "Alice") {
            println("Found!")
            return
        }
    }
    println("Alice is not found")
}
```

- `forEach`는 인라인 함수이므로 `return`은 `lookForAlice` 함수를 종료시킴.

## 람다로부터 반환: 레이블을 사용한 return

### 개념 설명

- 람다 안에서 `return`을 사용하면 비로컬 return이 발생하므로,
- 단순히 람다 실행을 중단하고 싶을 땐 **레이블을 붙여 로컬 return**을 사용해야 한다.

### 예시: `return@forEach` 사용

```kotlin
fun lookForAlice(people: List<Person>) {
    people.forEach label@{
        if (it.name != "Alice") return@label
        println("Found Alice!")
    }
}
```

- `return@label` 은 **해당 람다 블록만 종료**하고 다음 반복을 계속함.

### 람다 이름을 레이블로 사용하는 방법

```kotlin
fun lookForAlice(people: List<Person>) {
    people.forEach {
        if (it.name != "Alice") return@forEach
        println("Found Alice!")
    }
}
```

- `@forEach`는 명시적인 레이블 없이도 사용 가능 (함수 이름 기반)

### 레이블이 붙은 `this` 사용 예

```kotlin
fun main() {
    println(
        StringBuilder().apply sb@ {
            listOf(1, 2, 3).apply {
                this@sb.append(this.toString())
            }
        }.toString()
    )
}
```

- `this@sb`를 통해 바깥쪽 수신 객체에 접근 가능

## 10.3.3 익명 함수: 기본적으로 로컬 return

### 개념 설명

- **익명 함수**는 `fun` 키워드를 사용하는 람다의 또 다른 형태
- 익명 함수에서의 `return`은 **항상 로컬 return** → **람다 자신만 종료**
- 따라서 **비로컬 return 불가능**

### 예시: 익명 함수에서 return

```kotlin
fun lookForAlice(people: List<Person>) {
    people.forEach(fun(person) {
        if (person.name == "Alice") return
        println("${person.name} is not Alice")
    })
}

```

- 출력 결과:
    
    `Bob is not Alice`
    

### 예시: 익명 함수와 `filter`

```kotlin
val result = people.filter(fun(person): Boolean {
    return person.age < 30
})
```

- 또는 식 본문으로:
    
    ```kotlin
    val result = people.filter(fun(person) = person.age < 30)
    ```
    

### 정리 비교

| 표현 방식 | `return` 작동 방식 | 레이블 필요 여부 |
| --- | --- | --- |
| 일반 람다 | 비로컬 return (감싼 함수 종료) | 필요함 |
| 람다 + 레이블 | 로컬 return (람다만 종료) | 사용 권장 |
| 익명 함수 | 로컬 return (함수 자신만 종료) | 불필요 |

## 결론

- 인라인 함수에 전달된 **람다 안의 return은 비로컬 return**을 가능하게 한다.
- **람다 자신만 종료**하고 싶다면 `return@람다이름` 또는 `return@레이블` 형태로 작성해야 한다.
- **익명 함수**는 항상 로컬 return만 허용하므로 복잡한 흐름 제어 시 **대체 수단으로 유용**하다.
- 흐름 제어를 더 명확하게 하고 싶다면, **비로컬 return 대신 레이블 사용** 또는 **익명 함수 사용**을 고려하는 것이 좋다.

## 요약 정리

### 함수 타입과 고차 함수

- **함수 타입**은 `(파라미터) -> 반환값` 형태로 함수에 대한 참조를 변수, 파라미터, 반환값으로 저장할 수 있게 해준다.
- **고차 함수**는 함수를 **인자로 받거나**, 함수를 **반환하는 함수**를 말하며, 이를 통해 함수의 동작을 매개변수로 추상화할 수 있다.

### 인라인 함수와 성능 최적화

- **inline 함수**는 함수 본문과 람다 본문이 호출 지점에 직접 삽입되므로, 익명 클래스 및 함수 호출에 따른 **부가 비용이 제거된다**.
- 람다를 인라인하면 **함수 호출 오버헤드 제거**, **객체 생성 최소화**, **성능 향상**을 기대할 수 있다.
- 인라인 함수에서만 가능한 특별 기능: **비로컬 return**

### 람다의 흐름 제어

- 일반 람다에서 `return`을 사용하면 **함수를 감싸고 있는 외부 함수 전체가 종료**된다. 이를 **비로컬 return**이라고 한다.
- 이를 제어하려면:
    - `return@레이블` 사용 → **람다 자체만 종료**
    - 또는 **익명 함수** 사용 → 기본적으로 **로컬 return**

### 고차 함수의 장점

- **코드 재사용성 증가**: 반복되는 동작을 추상화 가능
- **범용 라이브러리 구현**: 제네릭 기반의 연산 구현에 적합
- **명확한 컨텍스트 분리**: 수신 객체 지정 람다 등으로 DSL 구현 가능

### 실전 예시를 통해 익힌 주요 함수

- `filter`, `map`, `forEach`, `withLock`, `use`, `useLines` 등 고차 함수는 람다를 인자로 받아 코드 재사용을 극대화하며, inline을 통해 성능도 확보
