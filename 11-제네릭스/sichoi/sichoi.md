# 타입 인자를 받는 타입 만들기: 제네릭 타입 파라미터

## 제네릭 타입과 함께 동작하는 함수와 프로퍼티

### 제네릭 타입 기본

- **제네릭 타입**은 클래스나 함수에 **타입 파라미터**를 사용하여 다양한 타입에 대응하도록 만든다.
- 예: `List<String>` → 문자열 리스트 / `Map<String, Int>` → 키는 문자열, 값은 정수
- 코틀린 컴파일러는 대부분의 경우 **타입 인자 추론** 가능
    
    ```kotlin
    val authors = listOf("Dmitry", "Svetlana") // List<String>으로 추론
    val readers: MutableList<String> = mutableListOf()
    val readers = mutableListOf<String>() // 둘 다 동일
    ```
    

### raw 타입은 없음

- 자바는 호환성 문제로 **raw type**을 허용하지만, 코틀린은 **항상 타입 인자 필요**
- 자바에서 넘어온 raw type은 `Any!` 플랫폼 타입으로 해석됨

### 제네릭 함수 정의

- 함수 자체에 타입 파라미터를 정의
    
    ```kotlin
    fun <T> List<T>.slice(indices: IntRange): List<T>
    ```
    
- 호출 시 명시도 가능하지만, 보통 **컴파일러가 추론**
    
    ```kotlin
    val letters = ('a'..'z').toList()
    println(letters.slice<Char>(0..2)) // 명시
    println(letters.slice(10..13))     // 추론
    ```
    

### 고차 함수와 제네릭

- 고차 함수 `filter` 예:
    
    ```kotlin
    fun <T> List<T>.filter(predicate: (T) -> Boolean): List<T>
    ```
    
- 타입 인자는 **리스트의 타입에 따라 추론**됨
    
    ```kotlin
    val authors = listOf("Svetlana", "Seb", "Roman", "Dima")
    val readers = mutableListOf("Seb", "Hadi")
    println(readers.filter { it !in authors }) // it은 String으로 추론
    ```
    

### 제네릭 확장 프로퍼티

- `penultimate`: 리스트의 끝에서 두 번째 요소를 반환
    
    ```kotlin
    val <T> List<T>.penultimate: T
        get() = this[size - 2]
    
    println(listOf(1, 2, 3, 4).penultimate) // 3
    ```
    
- **확장 프로퍼티만** 제네릭 가능 (일반 프로퍼티는 불가)

## 제네릭 클래스 선언

- 클래스/인터페이스에 `<T>` 붙이면 제네릭 타입 정의
    
    ```kotlin
    interface List<T> {
        operator fun get(index: Int): T
    }
    ```
    

### 제네릭 클래스 확장 예

```kotlin
class StringList : List<String> {
    override fun get(index: Int): String = TODO()
}

class ArrayList<T> : List<T> {
    override fun get(index: Int): T = TODO()
}
```

- `ArrayList<T>`에서 `T`는 상속받은 `List<T>`와는 별개인 새로운 타입 파라미터

### 타입 인자로 자기 자신 지정

- 예: `Comparable<T>`를 구현할 때, 자기 자신을 타입 인자로 넘기기
    
    ```kotlin
    class String : Comparable<String> {
        override fun compareTo(other: String): Int = TODO()
    }
    ```
    

## 타입 파라미터 제약 (Constraint)

### 단일 상계 (Upper Bound)

- 숫자만 처리하는 함수 정의:
    
    ```kotlin
    fun <T : Number> List<T>.sum(): T
    ```
    
- 예:
    
    ```kotlin
    fun <T : Number> oneHalf(value: T): Double {
        return value.toDouble() / 2.0
    }
    
    println(oneHalf(3)) // 1.5
    ```
    
- 비교 가능한 값에만 작동하는 `max`:
    
    ```kotlin
    fun <T : Comparable<T>> max(first: T, second: T): T {
        return if (first > second) first else second
    }
    
    println(max("kotlin", "java")) // kotlin
    ```
    

### 다중 상계

- `CharSequence`이자 `Appendable`인 경우만 허용:
    
    ```kotlin
    fun <T> ensureTrailingPeriod(seq: T)
        where T : CharSequence, T : Appendable {
        if (!seq.endsWith('.')) {
            seq.append('.')
        }
    }
    
    val helloWorld = StringBuilder("Hello World")
    ensureTrailingPeriod(helloWorld)
    println(helloWorld) // Hello World.
    ```
    

## 널 불허 제약

### 기본 상태

- 아무 제약 없는 `T`는 `T?`로도 대체 가능
    
    ```kotlin
    class Processor<T> {
        fun process(value: T) {
            value?.hashCode() // T? 가능성 있음
        }
    }
    
    val p = Processor<String?>() // 허용
    ```
    

### 널 불허를 명시하려면

- `T : Any` 로 상계 설정
    
    ```kotlin
    class Processor<T : Any> {
        fun process(value: T) {
            value.hashCode() // non-null 보장됨
        }
    }
    
    // val p = Processor<String?>() // 컴파일 오류
    ```
    

### 자바와 상호운용 시 제약 맞추기

- 자바에서는 다음 인터페이스처럼 일부 메서드만 널 불허할 수 있음:
    
    ```java
    public interface JBox<T> {
        void put(@NotNull T t);
        void putIfNotNull(T t);
    }
    ```
    
- 코틀린에서는 `T & Any`를 사용해 메서드 수준에서 제약 가능
    
    ```kotlin
    class KBox<T> : JBox<T> {
        override fun put(t: T & Any) { /* ... */ }
        override fun putIfNotNull(t: T) { /* ... */ }
    }
    ```
    

## 요약

- 제네릭 타입은 타입 안전성과 재사용성을 높인다.
- 함수/클래스/프로퍼티에 타입 파라미터를 정의할 수 있다.
- `T : 상위타입` 구문으로 제약을 설정할 수 있으며, 여러 제약은 `where` 절로 지정
- 널이 될 수 없는 타입을 강제하려면 `T : Any` 사용
- 자바 상호운용성을 위해 `T & Any`와 같은 고급 기법도 활용 가능

# 실행 시점 제네릭스 동작: 소거된 타입 파라미터와 실체화된 타입 파라미터

## 실행 시점 제네릭의 한계: 타입 검사와 캐스팅

### 타입 소거란?

- **JVM 제네릭**은 **타입 소거 (type erasure)** 방식으로 구현됨
- 실행 시점에는 타입 인자 정보가 존재하지 않음
- 예:
    
    ```kotlin
    val list1: List<String> = listOf("a", "b")
    val list2: List<Int> = listOf(1, 2, 3)
    ```
    
    → 런타임에는 둘 다 단순히 `List`일 뿐, 타입 차이는 소거됨
    

### 타입 소거의 문제

- `is List<String>` 같은 검사는 불가능
- 예외:
    - `value is List<*>`처럼 **스타 프로젝션**을 사용해 리스트 여부는 확인 가능
    - **`하지만  자리에 들어간 타입은 알 수 없음`**

### 캐스팅의 위험

- `as? List<Int>`와 같은 캐스팅은 런타임에 항상 성공 (단, 원소가 보장되지 않음)
- 예:
    
    ```kotlin
    fun printSum(c: Collection<*>) {
        val intList = c as? List<Int> ?: throw IllegalArgumentException("List is expected")
        println(intList.sum())
    }
    
    printSum(listOf(1, 2, 3)) // OK
    printSum(setOf(1, 2, 3)) // IllegalArgumentException
    printSum(listOf("a", "b")) // ClassCastException
    ```
    

### 컴파일러의 역할

- 컴파일 시점에 타입이 명확할 경우 `is` 검사는 허용
    
    ```kotlin
    fun printSum(c: Collection<Int>) {
        when (c) {
            is List<Int> -> println("List sum: ${c.sum()}")
            is Set<Int> -> println("Set sum: ${c.sum()}")
        }
    }
    ```
    

## 인라인 함수에서 타입 인자를 실체화하기

### 일반 함수에서는 타입 검사 불가

- 예:
    
    ```kotlin
    fun <T> isA(value: Any) = value is T // 컴파일 오류
    ```
    

### inline + reified 조합

- 해결 방법: 인라인 함수로 만들고 `reified` 키워드를 붙인다
    
    ```kotlin
    inline fun <reified T> isA(value: Any) = value is T
    
    println(isA<String>("abc")) // true
    println(isA<String>(123))   // false
    ```
    

### filterIsInstance

- 표준 함수 `filterIsInstance<T>()`: 타입에 맞는 원소만 걸러냄
    
    ```kotlin
    val items = listOf("one", 2, "three")
    println(items.filterIsInstance<String>()) // [one, three]
    ```
    
- 내부 구현:
    
    ```kotlin
    inline fun <reified T> Iterable<*>.filterIsInstance(): List<T> {
        val destination = mutableListOf<T>()
        for (element in this) {
            if (element is T) {
                destination.add(element)
            }
        }
        return destination
    }
    ```
    

### 왜 일반 함수에선 안 되고 인라인 함수에선 되는가?

- 인라인 함수는 호출 지점에 타입이 명확하므로 컴파일 시점에 `is String` 같은 구체적인 검사로 대체 가능

## 클래스 참조에 실체화된 타입 사용

### 기존 방식

- 자바식:
    
    ```kotlin
    ServiceLoader.load(MyService::class.java)
    ```
    

### 코틀린식: 실체화된 타입 파라미터 사용

- 더 간단하고 명확한 표현
    
    ```kotlin
    inline fun <reified T> loadService(): T {
        return ServiceLoader.load(T::class.java)
    }
    
    val serviceImpl = loadService<MyService>()
    ```
    

### 안드로이드 예제

- `startActivity()` 호출도 간단하게 변형 가능
    
    ```kotlin
    inline fun <reified T : Activity> Context.startActivity() {
        val intent = Intent(this, T::class.java)
        startActivity(intent)
    }
    
    startActivity<DetailActivity>()
    ```
    

## 실체화된 타입 파라미터를 프로퍼티 접근자에서 사용

- 실체화된 타입 파라미터를 사용한 **게터 정의**:
    
    ```kotlin
    inline val <reified T> T.canonical: String
        get() = T::class.java.canonicalName
    
    println(listOf(1, 2, 3).canonical) // java.util.List
    println(1.canonical)              // java.lang.Integer
    ```
    

## 실체화된 타입 파라미터의 제약

### 가능한 작업

- `is`, `!is`, `as`, `as?` 등의 **타입 검사 및 캐스팅**
- `::class`, `::class.java` 사용
- 다른 함수의 타입 인자로 사용 가능

### 불가능한 작업

- 실체화된 타입 파라미터의 **인스턴스 생성**
- 실체화된 타입 파라미터의 **companion object 호출**
- 실체화된 타입 파라미터로 넘길 타입을 **다른 타입 파라미터로 전달**
- **클래스/프로퍼티/일반 함수**의 타입 파라미터에는 `reified` 사용 불가

### 인라인 함수의 부작용

- 인라인 함수에 `reified`를 사용하면 모든 **람다 파라미터도 인라이닝됨**
- 필요 없는 인라이닝을 막고 싶다면 `noinline` 키워드로 방지 가능

## 요약

- JVM의 타입 소거로 인해 **제네릭 타입 인자는 런타임에 알 수 없음**
- `inline` + `reified` 조합으로 타입 인자를 **실행 시점에 확인** 가능
- 대표 활용 예:
    - `isA<T>()`
    - `filterIsInstance<T>()`
    - `loadService<T>()`
    - `Activity` 클래스 지정 없이 `startActivity<T>()`
- 일반 함수나 클래스에는 `reified`를 사용할 수 없으며, 인라인 함수로 한정됨
- 성능 최적화를 고려해 **함수 크기를 관리**해야 하며, 필요시 일반 함수로 분리하거나 `noinline`을 사용해야 함

알겠습니다. 핵심 개념을 더 잘 드러내고, 코드와 개념의 연결이 명확히 되도록 구체적으로 다시 정리하겠습니다.

# 타입 인자를 받는 타입 만들기: 제네릭 타입 파라미터

## 제네릭 타입과 함께 동작하는 함수와 프로퍼티

### 제네릭 함수의 정의와 동작 방식

- 함수에 `<T>` 형식의 타입 파라미터를 선언하여 **다양한 타입에 대응 가능**하게 만든다.
- 타입 파라미터는 함수의 **수신 객체**, **파라미터**, **반환 타입** 등에 사용된다.
- `List<T>.slice()` 함수는 대표적인 예시로, 리스트의 일부 범위를 반환하는 제네릭 함수이다.

```kotlin
fun <T> List<T>.slice(indices: IntRange): List<T>
```

- 사용 시 타입 인자를 명시할 수도 있지만, **컴파일러가 타입을 추론**하기 때문에 생략 가능하다.

```kotlin
val letters = ('a'..'z').toList()

println(letters.slice<Char>(0..2)) // 명시적 타입: [a, b, c]
println(letters.slice(10..13))     // 타입 추론: [k, l, m, n]
```

### 고차 함수와 제네릭의 결합

- 고차 함수인 `filter`는 `(T) -> Boolean` 형태의 함수 타입 파라미터를 받는다.
- 람다의 파라미터 타입은 호출된 리스트의 타입으로부터 **컴파일러가 T를 추론**한다.

```kotlin
val authors = listOf("Sveta", "Seb", "Roman", "Dima")
val readers = mutableListOf("Seb", "Hadi")

println(readers.filter { it !in authors }) // [Hadi]
```

- 이 경우 `readers`가 `List<String>`이므로, T는 `String`으로 추론된다.

### 제네릭 확장 프로퍼티

- 확장 프로퍼티에서도 타입 파라미터 사용 가능.
- 예: 리스트의 마지막 전 원소를 반환하는 `penultimate` 확장 프로퍼티

```kotlin
val <T> List<T>.penultimate: T
    get() = this[size - 2]

println(listOf(1, 2, 3, 4).penultimate) // 3
```

- **일반 클래스 프로퍼티**는 타입 파라미터를 사용할 수 없음. → 수신 객체 타입에 타입 파라미터가 쓰이지 않기 때문

```kotlin
val <T> x: T = TODO()
// ERROR: type parameter of a property must be used in its receiver type
```

## 제네릭 클래스 선언

### 기본 선언 방식

- 클래스/인터페이스 이름 뒤에 `<T>`를 붙여 제네릭 클래스 선언
- 내부에서는 일반 타입처럼 T를 사용할 수 있음

```kotlin
interface List<T> {
    operator fun get(index: Int): T
}
```

### 타입 인자 명시 및 확장

- 제네릭 클래스를 확장할 때는 상위 타입의 타입 인자를 **구체 타입** 또는 **타입 파라미터**로 지정 가능

```kotlin
class StringList: List<String> {
    override fun get(index: Int): String = ...
}

class ArrayList<T>: List<T> {
    override fun get(index: Int): T = ...
}
```

- 이름이 같더라도 서로 다른 클래스의 타입 파라미터는 **독립적**임

### 자기 자신을 타입 인자로 참조

- `Comparable<T>` 패턴: 자신과 동일한 타입을 비교 대상으로 명시

```kotlin
class String : Comparable<String> {
    override fun compareTo(other: String): Int = ...
}
```

## 타입 파라미터 제약 (Constraints)

### 상계(bound) 설정

- 타입 파라미터는 특정 상위 타입을 **상계(bound)** 로 가질 수 있음
- 예: 숫자만 가능한 리스트 합산 함수

```kotlin
fun <T : Number> List<T>.sum(): T
```

- `T`는 반드시 `Number` 혹은 그 하위 타입이어야 하므로 `List<String>`과 같이 부적절한 타입은 컴파일 시 거부됨

### 상계 타입의 메서드 호출 가능

- 상계를 지정하면 해당 타입의 메서드를 안전하게 호출 가능

```kotlin
fun <T : Number> oneHalf(value: T): Double {
    return value.toDouble() / 2.0
}
```

### `Comparable<T>`을 활용한 정렬 가능 타입 제약

- `max()` 함수 정의 시, 비교 가능한 타입만 받도록 제약 가능

```kotlin
fun <T : Comparable<T>> max(first: T, second: T): T {
    return if (first > second) first else second
}

println(max("kotlin", "java")) // kotlin
```

### 다중 제약 조건 (복수 상계)

- `where` 절을 사용해 타입 파라미터에 여러 제약 설정 가능

```kotlin
fun <T> ensureTrailingPeriod(seq: T)
    where T : CharSequence, T : Appendable {
    if (!seq.endsWith(".")) {
        seq.append(".")
    }
}

```

- `StringBuilder`는 `CharSequence`이자 `Appendable`이므로 인자로 사용 가능

## 널 가능성 제약

### 널 불가능 타입 제약 (`T : Any`)

- 타입 파라미터가 **널 불가능한 타입만** 받도록 제약

```kotlin
class Processor<T : Any> {
    fun process(value: T) {
        value.hashCode() // null 안전 호출 필요 없음
    }
}
```

- `Processor<String?>`와 같이 널 타입은 사용할 수 없음 → 컴파일 오류 발생

### `T & Any`를 통한 자바 인터페이스 상호 운용성 확보

- 자바 인터페이스가 일부 메서드에서만 `@NotNull`을 명시한 경우, 해당 메서드에서만 `T & Any` 사용

```kotlin
class KBox<T>: JBox<T> {
    override fun put(t: T & Any) { /* ... */ }
    override fun putIfNotNull(t: T) { /* ... */ }
}
```

- 전체 타입이 `T : Any`일 경우 `putIfNotNull`처럼 null을 허용하는 메서드는 구현 불가
- `T & Any`를 사용하면 특정 메서드만 null 불가로 제한 가능 → 자바 인터페이스와 완전 호환

# 실행 시점 제네릭스 동작: 소거된 타입 파라미터와 실체화된 타입 파라미터

## 실행 시점에 제네릭 타입 정보를 사용할 수 없는 이유

### JVM의 타입 소거

- JVM의 제네릭스는 **타입 소거(type erasure)** 기반으로 동작.
- 제네릭 클래스나 함수의 타입 인자 정보는 **실행 시점에 제거**되어 유지되지 않음.

```kotlin
val list1: List<String> = listOf("a", "b")
val list2: List<Int> = listOf(1, 2, 3)
```

- 컴파일 시점에는 `List<String>`과 `List<Int>`가 구별되지만, 런타임에는 둘 다 `List`로만 식별됨.

### 타입 검사 및 캐스팅의 한계

- 실행 시점에는 다음과 같은 **타입 검사가 불가능**함:

```kotlin
when (list) {
    is List<String> -> ... // 컴파일 오류: Cannot check for instance of erased type
}
```

- 해결 방법 중 하나는 `List<*>` 형태로 **스타 프로젝션** 사용:

```kotlin
if (value is List<*>) { ... } // 원소 타입은 알 수 없음
```

### 캐스팅은 가능하지만 안전하지 않음

- `as` 또는 `as?` 캐스팅은 수행되지만 타입 정보가 없기 때문에 **경고만 출력**됨:

```kotlin
val intList = c as? List<Int> ?: throw IllegalArgumentException("List is expected")
```

- 실제로 잘못된 타입이 섞인 경우엔 `ClassCastException` 발생 가능:

```kotlin
printSum(listOf("a", "b", "c")) // 런타임 오류
```

### 타입이 고정된 경우에는 검사 가능

- 함수 파라미터에 **고정된 타입 인자**가 명시된 경우, `is` 검사가 가능함:

```kotlin
fun printSum(c: Collection<Int>) {
    when (c) {
        is List<Int> -> println("List sum: ${c.sum()}")
        is Set<Int> -> println("Set sum: ${c.sum()}")
    }
}
```

## 인라인 함수에서의 실체화된 타입 파라미터

### 일반 함수에서는 타입 인자 참조 불가능

- 제네릭 함수 내에서는 `value is T` 같은 **타입 검사가 불가능**함:

```kotlin
fun <T> isA(value: Any): Boolean = value is T // 오류 발생
```

### 인라인 함수에서만 `reified` 사용 가능

- `inline` + `reified`를 사용하면 컴파일 타임에 타입 정보를 주입할 수 있음:

```kotlin
inline fun <reified T> isA(value: Any) = value is T
```

- 호출 예:

```kotlin
println(isA<String>("abc")) // true
println(isA<String>(123))   // false
```

- 이유: 인라인 함수는 호출 시점에 **함수 본문이 복사**되므로 타입 인자를 **구체적인 타입**으로 치환 가능

### 표준 라이브러리 함수 `filterIsInstance`

- `reified`의 대표적인 활용 예

```kotlin
val items = listOf("one", 2, "three")
println(items.filterIsInstance<String>()) // [one, three]
```

- 정의 구조 (단순화된 예):

```kotlin
inline fun <reified T> Iterable<*>.filterIsInstance(): List<T> {
    val destination = mutableListOf<T>()
    for (element in this) {
        if (element is T) {
            destination.add(element)
        }
    }
    return destination
}
```

## 실체화된 타입 파라미터로 `Class<T>` 대체

### Java Class 기반 API 어댑터 예: `ServiceLoader`

```kotlin
val serviceImpl = ServiceLoader.load(Service::class.java)
```

- 위 코드를 다음처럼 간결하게 바꿀 수 있음:

```kotlin
val serviceImpl = loadService<Service>()
```

- `loadService` 정의:

```kotlin
inline fun <reified T> loadService(): T {
    return ServiceLoader.load(T::class.java)
}
```

### Android 예제: `startActivity`

- 액티비티 클래스를 직접 넘기지 않고, 타입 인자로 전달

```kotlin
inline fun <reified T : Activity> Context.startActivity() {
    val intent = Intent(this, T::class.java)
    startActivity(intent)
}

startActivity<DetailActivity>()
```

## 실체화된 타입 파라미터를 사용하는 프로퍼티 접근자

- 프로퍼티에서도 `inline val`과 `reified` 조합 가능

```kotlin
inline val <reified T> T.canonical: String
    get() = T::class.java.canonicalName
```

- 예제:

```kotlin
println(listOf(1, 2, 3).canonical) // java.util.List
println(1.canonical)              // java.lang.Integer
```

## 실체화된 타입 파라미터의 제약

### 가능한 사용 예

- `is`, `!is`, `as`, `as?` 같은 **타입 검사 및 캐스팅**
- `::class` (KClass)
- `::class.java` (Java Class)
- 다른 제네릭 함수의 **타입 인자**

### 불가능한 사용 예

- 타입 인자 클래스의 **인스턴스 생성**
- **동반 객체 메서드** 호출
- **실체화하지 않은 타입 파라미터**로 `reified` 함수 호출
- `class`, `property`, **비인라인 함수**에서 `reified` 사용

### 관련 추가 사항

- `reified` 함수 내부에서 사용하는 **람다는 무조건 인라이닝**됨
- 람다를 인라이닝하지 않도록 하고 싶다면 `noinline` 키워드 사용

# 변성은 제네릭과 타입 인자 사이의 하위 타입 관계를 기술

**`변성(variance)`**은 기저 타입은 같지만 타입 인자가 다른 제네릭 타입들 간의 하위 타입 관계를 설명하는 개념이다. 예를 들어 `List<String>`과 `List<Any>`는 둘 다 `List`라는 기저 타입을 공유하지만, 타입 인자가 다르기 때문에 이들 간의 하위 타입 관계가 자동으로 성립하지는 않는다.

## 변성은 인자를 함수에 넘겨도 안전한지 판단하게 해준다

타입 `String`은 `Any`의 하위 타입이므로, `String` 값을 `Any` 타입을 받는 함수에 넘기는 건 항상 안전하다. 하지만 `List<String>`을 `List<Any>`에 넘길 수 있을까?

예제:

```kotlin
fun printContents(list: List<Any>) {
    println(list.joinToString())
}

fun main() {
    printContents(listOf("abc", "bac")) // 안전
}
```

위 코드는 안전하다. `List<Any>`는 읽기 전용이며, 내부 원소는 `Any`로만 읽히기 때문에 `String` 값도 문제 없다.

그러나 아래는 위험하다:

```kotlin
fun addAnswer(list: MutableList<Any>) {
    list.add(42)
}

fun main() {
    val strings = mutableListOf("abc", "bac")
    addAnswer(strings) // 만약 허용된다면, Int가 String 리스트에 추가됨
}
```

실행 시점에는 `ClassCastException`이 발생한다. `MutableList<Any>`는 쓰기가 가능하므로, `List<String>`을 넘기면 타입 안정성이 깨진다.

### 결론

- 읽기 전용 컬렉션은 `List<String>` → `List<Any>`로 안전하게 대입 가능
- 변경 가능한 컬렉션은 불가능
- 코틀린 컴파일러는 타입 불일치를 방지하기 위해 `MutableList<String>`을 `MutableList<Any>`로 넘기는 것을 **금지**한다

## 클래스, 타입, 하위 타입

### 타입과 클래스의 차이

- **클래스**는 일종의 템플릿 (e.g. `List`)
- **타입**은 클래스에 구체적인 타입 인자를 채운 결과 (e.g. `List<Int>`)

예:

- `String` → 타입이기도 하고 클래스이기도 함
- `String?` → 널이 될 수 있는 타입이지만 같은 클래스 `String` 기반
- `List`는 클래스지만 타입은 아님
- `List<Int>`는 타입

### 하위 타입(Subtyping)의 정의

어떤 타입 B가 타입 A의 하위 타입이라는 것은:

> A가 필요한 모든 위치에 B를 넣어도 안전하다면, B는 A의 하위 타입이다
> 

예:

```kotlin
val x: Number = 3 // OK: Int는 Number의 하위 타입
val s: String = "abc"
val t: String? = s // OK: String은 String?의 하위 타입
```

- `Int` → `Number`의 하위 타입
- `String` → `CharSequence`의 하위 타입
- `T` → `T?`의 하위 타입 (널이 될 수 없는 타입이 널이 될 수 있는 타입보다 더 구체적)

### 무공변(Invariant) 제네릭 타입

`MutableList<A>`와 `MutableList<B>`는 A와 B가 아무 관계가 없어도 서로 **하위 타입 관계가 아님**

- 코틀린에서는 `MutableList<T>`는 무공변 → 변성 키워드가 없다면 하위 타입 관계 없음
- 이는 API 사용 시 안전성을 보장하기 위한 설계

## 공변성은 하위 타입 관계를 유지한다

**`공변성(covariance)`**은 타입 인자의 하위 타입 관계가 제네릭 타입에도 그대로 반영되는 성질이다. 즉, `T1`이 `T2`의 하위 타입일 때, `Generic<T1>`이 `Generic<T2>`의 하위 타입이 되면 공변적이라 한다.

예를 들어:

- `Cat`은 `Animal`의 하위 타입
- `Producer<Cat>`이 `Producer<Animal>`의 하위 타입이면, `Producer<T>`는 공변적

### Kotlin에서 공변성 선언 방법

타입 파라미터 앞에 `out` 키워드를 붙여 선언한다:

```kotlin
interface Producer<out T> {
    fun produce(): T
}
```

- `T`는 클래스 외부로만 나가는 방향(생산)으로 사용됨
- 즉, T를 함수 반환 타입 등 **아웃 위치(out position)** 에서만 사용 가능

### 예제: 공변적인 Herd 클래스

```kotlin
open class Animal {
    fun feed() {}
}
class Cat : Animal() {
    fun cleanLitter() {}
}

fun feedAll(animals: Herd<Animal>) {
    for (i in 0 until animals.size) {
        animals[i].feed()
    }
}

// 공변적으로 선언하지 않은 경우
class Herd<T : Animal> {
    val size: Int get() = ...
    operator fun get(i: Int): T = ...
}

// 공변적으로 선언한 경우
class Herd<out T : Animal> {
    val size: Int get() = ...
    operator fun get(i: Int): T = ...
}
```

- `Herd<Cat>`은 `Herd<Animal>`의 하위 타입
- `T`는 `get()` 함수의 반환값으로만 사용되므로 `out` 사용이 안전함

```kotlin
fun feedAll(animals: Herd<Animal>) {
    for (i in 0 until animals.size) {
        animals[i].feed()
    }
}

fun takeCareOfCats(cats: Herd<Cat>) {
    feedAll(cats)
}
```

- in 키워드를 붙인 경우 -> OK. 정상 동작
- in 키워드를 붙이지 않은 경우 -> FAIL. 컴파일 에러 (고냥이들이 굶어버림 ㅠㅠ)
    
    ![image](https://github.com/user-attachments/assets/2cdfd365-e0f8-4b96-b5bc-21548e67f713)

    

## 공변성의 제약

- `out T`로 선언하면, `T`는 **반환 전용**이므로 소비(함수 파라미터 등)에는 사용 불가
- 이를 통해 타입 안정성을 유지

## 반공변성은 하위 타입 관계를 뒤집는다

## 개념 설명

**`반공변성(contravariance)`**은 **하위 타입 관계가 역전되는 것**이다.

즉, `T1`이 `T2`의 하위 타입일 때, `Generic<T2>`가 `Generic<T1>`의 하위 타입이면 반공변적이다.

예:

- `Cat` <: `Animal`
- `Consumer<Animal>` <: `Consumer<Cat>`

## Kotlin에서 반공변성 선언 방법

타입 파라미터 앞에 `in` 키워드를 붙여 선언한다:

```kotlin
interface Comparator<in T> {
    fun compare(e1: T, e2: T): Int
}
```

- `T`는 함수 파라미터 타입(즉, **인 위치(in position)**)에서만 사용됨

## 예제: Fruit 비교기

```kotlin
sealed class Fruit { abstract val weight: Int }
data class Apple(...) : Fruit()
data class Orange(...) : Fruit()

val comparator: Comparator<Fruit> = Comparator { a, b -> a.weight - b.weight }

val apples = listOf(Apple(...), Apple(...))
println(apples.sortedWith(comparator)) // OK
```

- `Comparator<Fruit>`는 `Comparator<Apple>`로 사용할 수 있음
- 소비만 하는 타입이므로 상위 타입을 넘겨도 안전

## 사용 지점 변성을 사용해 타입이 언급되는 지점에서 변성 지정

### 선언 지점 변성과 사용 지점 변성

- **선언 지점 변성(declaration-site variance)**: 클래스나 인터페이스 선언 시 `in`, `out`을 지정
- **사용 지점 변성(use-site variance)**: 타입을 사용할 때 `in`, `out`을 지정 (자바의 `? extends`, `? super`와 유사)

### 예제: 복사 함수에서 사용 지점 변성 활용

**문제**

```kotlin
fun <T> copyData(source: MutableList<T>, destination: MutableList<T>)
```

- `MutableList`는 무공변이므로 `MutableList<Int>` → `MutableList<Any>` 불가능

**해결책 1: 두 개의 타입 파라미터 도입**

```kotlin
fun <T : R, R> copyData(source: MutableList<T>, destination: MutableList<R>)
```

- `Int <: Any`이면 호출 가능

**해결책 2: 사용 지점 변성 적용**

```kotlin
fun <T> copyData(source: MutableList<out T>, destination: MutableList<T>)
```

- `source`는 읽기 전용
- `out`은 인자 위치에서 `T` 사용을 금지

```kotlin
val list: MutableList<out Number> = mutableListOf()
list.add(42) // 컴파일 오류
```

### in 프로젝션

```kotlin
fun <T> copyData(source: MutableList<T>, destination: MutableList<in T>)
```

- `destination`은 소비자 역할 → `T`의 상위 타입을 받을 수 있음

## 요약

| 변성 종류 | 예시 | 하위 타입 관계 | 사용 위치 |
| --- | --- | --- | --- |
| 공변 (`out`) | `Producer<out T>` | 그대로 유지됨 | 반환 위치만 가능 |
| 반공변 (`in`) | `Consumer<in T>` | 뒤집힘 | 파라미터(소비) 위치만 가능 |
| 무공변 | `MutableList<T>` | 없음 | 어디든 가능 |

## 스타 프로젝션: 제네릭 타입 인자에 대한 정보가 없음을 표현하고자  사용

## 개념 설명

스타 프로젝션(`*`)은 타입 인자에 대한 **정확한 정보가 없거나 중요하지 않을 때** 사용하는 표기이다.

```kotlin
val unknownElements: MutableList<*> = ...
```

이 표현은 다음과 같은 의미를 갖는다:

- 리스트가 특정 타입의 원소를 가진다는 사실은 알고 있지만, **그 타입이 무엇인지 모른다**
- 따라서 리스트에서 **읽기(get)는 가능**하지만, **추가(add)는 불가능**

### `MutableList<*>` vs `MutableList<Any?>`

- `MutableList<Any?>`: 어떤 타입이든 원소를 담을 수 있는 리스트
- `MutableList<*>`: 어떤 특정 타입의 원소만 담을 수 있지만, 우리는 **그 타입을 알 수 없다**

예제:

```kotlin
val chars = mutableListOf('a', 'b', 'c')
val mixed = mutableListOf<Any?>('a', 1, null)

val unknown: MutableList<*> = if (Random.nextBoolean()) chars else mixed

println(unknown.first()) // 읽기는 가능
unknown.add(42)          // 컴파일 오류
```

### 내부 동작 원리

- `MutableList<*>`는 내부적으로 `MutableList<out Any?>`처럼 취급된다 (아웃 프로젝션)
- 읽기는 안전 (반환 타입은 항상 `Any?`)
- 쓰기는 위험 (타입 안정성이 보장되지 않음)

### 반공변 타입에서의 스타 프로젝션

- `Consumer<*>`는 내부적으로 `Consumer<in Nothing>`과 동일하게 처리된다
- 이 경우는 어떤 값을 소비하는지 알 수 없기 때문에, **모든 소비 메서드 호출이 금지된다**

### 사용 시기

- 타입 파라미터의 **구체적인 타입이 중요하지 않은 경우**
- 데이터를 읽기만 하고 **구체적인 타입 로직이 필요하지 않은 경우**

예제:

```kotlin
fun printFirst(list: List<*>) {
    if (list.isNotEmpty()) {
        println(list.first()) // 타입은 Any?
    }
}
```

### 대체 방안

- 타입이 중요하다면 **제네릭 함수로 직접 선언**

```kotlin
fun <T> printFirst(list: List<T>) {
    println(list.first()) // 타입은 T
}
```

## 타입 별칭

`타입 별칭(typealias)`은 **복잡한 타입 표현식을 간결한 이름으로 치환**하는 기능이다.

```kotlin
typealias NameCombiner = (String, String, String, String) -> String
```

- 긴 타입 시그니처에 의미 있는 이름을 붙일 수 있음
- 타입 자체의 **의미 부여** 및 **가독성 향상** 목적

### 예제: 저자 이름 조합 함수

```kotlin
val authorsCombiner: NameCombiner = { a, b, c, d -> "$a et al." }
val bandCombiner: NameCombiner = { a, b, c, d -> "$a, $b & The Gang" }

fun combineAuthors(combiner: NameCombiner) {
    println(combiner("Sveta", "Seb", "Dima", "Roman"))
}
```

- `combineAuthors`는 `NameCombiner`를 매개변수로 받아 다양한 방식으로 이름을 조합한다

### 타입 안전성과의 관계

타입 별칭은 **기존 타입에 단순한 별칭**만 부여하며, 컴파일러는 내부적으로 이를 원래 타입으로 치환한다.

→ **타입 안전성 강화 효과는 없다. (강제성은 없음)**

예:

```kotlin
typealias ValidatedInput = String

fun save(v: ValidatedInput) {}

val rawInput = "needs validating!"
save(rawInput) // OK — 아무 String이나 전달 가능
```

- 타입이 여전히 `String`이기 때문에, 안전성 검증을 강제할 수 없음

### 인라인 클래스와 비교

타입 안전성이 중요한 경우에는 `value class`를 사용하라:

```kotlin
@JvmInline
value class ValidatedInput(val s: String)

fun save(v: ValidatedInput) {}

val rawInput = "needs validating!"
save(rawInput) // 컴파일 오류: 명시적 변환 필요
```

- 인라인 클래스는 **다른 타입과 구분됨** → 컴파일 타임 타입 안전성 보장

## 요약

| 구분 | 타입 별명 (`typealias`) | 인라인 클래스 (`value class`) |
| --- | --- | --- |
| 목적 | 간결한 이름 제공 | 타입 안전성 강화 |
| 컴파일 시 처리 | 원래 타입으로 치환 | 별개의 타입으로 인식 |
| 타입 구분 | 불가능 | 가능 |
| 사용 예 | 읽기 쉬운 함수 시그니처 | 검증된 입력 타입 정의 등 |

# 오늘의 깜짝 퀴즈

## 문제 1번

```kotlin
interface Transformer<in T, out R> {
    fun transform(value: T): R
}

open class Animal
class Dog : Animal()
class Cat : Animal()

fun useTransformer(transformer: Transformer<Animal, Animal>) {
    val result = transformer.transform(Cat())
    println(result)
}

fun main() {
    val t: Transformer<Dog, Cat> = object : Transformer<Dog, Cat> {
        override fun transform(value: Dog): Cat = Cat()
    }
    
    val result = useTransformer(t, Dog())

    println(result) // ?
}
```

- `Dog` 가 들어오면 `Cat` 으로 마개조(?)시키는 코드이다.
- 이 코드는 예상처럼 동작할까? **최종 출력 결과**는 무엇일까?


## 2번

```kotlin
typealias Username = String

@JvmInline
value class ValidatedUsername(val value: String)

fun saveUsername(username: ValidatedUsername) {
    println("Saving $username")
}

fun main() {
    val raw: Username = "식호이"
    saveUsername(raw)
}
```

- `Username` 이라는 타입 별칭을 사용한 코드이다.
- 이 코드는 예상처럼 동작할까?

