# 산술 연산자 오버로딩

## 코틀린에서 연산자 오버로딩이 가능한 이유

- 코틀린은 연산자 오버로딩을 **함수 이름 규칙**과 `operator` 키워드를 통해 지원한다.
- 예: `a + b`는 내부적으로 `a.plus(b)`를 호출한다.

## 오버로딩 가능한 이항 연산자 목록

| 연산자 | 함수 이름 |
| --- | --- |
| `a + b` | `plus` |
| `a - b` | `minus` |
| `a * b` | `times` |
| `a / b` | `div` |
| `a % b` | `mod` (또는 `rem`) |

> operator 키워드를 함수 앞에 붙여야 오버로딩 연산자로 인식된다.
> 

## 이항 산술 연산 오버로딩 사용 형태

1. **클래스 멤버 함수** 또는 **확장 함수**로 정의 가능
    
    ```kotlin
    operator fun Point.plus(other: Point): Point
    ```
    
2. **타입이 서로 달라도 가능**
    
    예: `Point * Double`, `Char * Int`
    
3. **교환법칙 지원은 수동으로**
    
    `p * 1.5`는 `Point.times(Double)`
    
    `1.5 * p`는 `Double.times(Point)`를 **따로 정의해야 함**
    
4. **리턴 타입은 자유롭게 정의 가능**
    
    예: `Char * Int` → `String` 리턴 가능
    

### ⚠️ 주의할 점

- 연산자 우선순위는 항상 **표준 연산자 우선순위**에 따라 고정된다.
- 연산자 오버로딩은 **기호 자체를 새로 정의할 수는 없다.** (`*`, `%%` 같은 사용자 지정 연산자는 불가)
- 비트 연산자는 **중위 함수**로 사용하며, 별도 연산자 오버로딩 함수는 없음

### 비트 연산 함수 목록 (숫자 타입 전용)

| 연산 | 함수 이름 |
| --- | --- |
| `<<` | `shl` |
| `>>` | `shr` |
| `>>>` | `ushr` |
| `&` | `and` |
| ` | ` |
| `^` | `xor` |
| `~` | `inv` |

예시:

```kotlin
val result = 0xF0 and 0x0F  // 결과: 0
```

## 복합 대입 연산자 오버로딩

### ### 복합 대입 연산자란?

- `a += b`는 기본적으로 `a = a + b`로 해석된다.
- 이때 `+`는 `plus`, `+=`는 `plusAssign` 함수로 대응된다.

### 코틀린에서 지원하는 복합 대입 연산자 함수들

| 연산자 | 대응 함수 이름 |
| --- | --- |
| `+=` | `plusAssign` |
| `-=` | `minusAssign` |
| `*=` | `timesAssign` |
| `/=` | `divAssign` |
| `%=` | `modAssign` 또는 `remAssign` |

### `plus`만 정의한 경우 (`a = a + b`)

```kotlin
data class Point(val x: Int, val y: Int) {
    operator fun plus(other: Point) = Point(x + other.x, y + other.y)
}
fun main() {
    var p = Point(1, 2)
    p += Point(3, 4)  // 자동으로 p = p + Point(3, 4)
}
```

### `plusAssign`을 정의한 경우 (원본 변경)

```kotlin
operator fun <T> MutableCollection<T>.plusAssign(element: T) {
    this.add(element)
}
fun main() {
    val numbers = mutableListOf<Int>()
    numbers += 42   // 내부에서 add(42) 호출
}
```

### ⚠️ `plus`와 `plusAssign`이 동시에 있으면?

- **컴파일 오류 발생**
    
    → 둘 중 어떤 걸 호출할지 모호하므로 컴파일러는 오류를 낸다.
    

**해결 방법**

- `val`로 선언해서 `plusAssign`이 적용되지 않게 하기
- **둘 중 하나만 정의**해서 일관성 유지하기

### 컬렉션에서는?

- `+`, : **새 컬렉션을 반환** (읽기 전용 컬렉션에도 가능)
- `+=`, `=`: **기존 컬렉션을 변경** (변경 가능한 컬렉션에서만)

```kotlin
val list = mutableListOf(1, 2)
list += 3                    // 리스트에 3 추가
val newList = list + listOf(4, 5)  // 새 리스트 반환
```

## 단항 연산자 오버로딩

### 단항 연산자란?

피연산자가 **하나뿐**인 연산자. 예:

- `+a`, `a`, `!a`, `++a`, `-a`

### 오버로딩 방법

- `operator` 키워드를 붙인 **멤버 함수 또는 확장 함수**로 정의
- 인자는 없음 (피연산자 하나 → 함수 내에서 `this` 사용)

### 예시: 좌표 부호 반전

```kotlin
data class Point(val x: Int, val y: Int)

operator fun Point.unaryMinus(): Point {
    return Point(-x, -y)
}

fun main() {
    val p = Point(10, 20)
    println(-p) // Point(x=-10, y=-20)
}
```

### 오버로딩 가능한 단항 연산자 표

| 연산자 | 함수 이름 | 설명 |
| --- | --- | --- |
| `+a` | `unaryPlus()` | 단항 양수 (그대로 반환) |
| `-a` | `unaryMinus()` | 부호 반전 |
| `!a` | `not()` | 논리 부정 |
| `++a`, `a++` | `inc()` | 증가 (전위, 후위 모두 지원) |
| `--a`, `a--` | `dec()` | 감소 (전위, 후위 모두 지원) |

### 예시: 증가 연산자 `++`

```kotlin
import java.math.BigDecimal

operator fun BigDecimal.inc(): BigDecimal = this + BigDecimal.ONE

fun main() {
    var num = BigDecimal.ZERO

    println(num++) // 0 (출력 후 증가)
    println(num)   // 1
    println(++num) // 2 (증가 후 출력)
}
```

- **후위 `a++`** → 값 반환 후 증가
- **전위 `++a`** → 증가 후 값 반환

코틀린이 **자동으로 후위/전위 차이를 구현**해준다.

### ⚠️ 주의사항

- `++`와 `-`는 **반드시 `var` 변수**에만 사용 가능 (`val` 불가)
- `inc()` / `dec()`는 **새 인스턴스를 반환**해야 하며, 원본 수정은 하지 않음

# 비교 연산자 오버로딩

## 동등성 연산자: `==`, `!=`

- `a == b`는 내부적으로 `a?.equals(b) ?: (b == null)` 로 컴파일됨
- 즉, `null` 안전하고 equals 오버라이딩에 기반
- `!=`는 위 결과를 **반전**
- `equals()`는 `Any`에 정의되어 있으므로 `override fun equals(...)`로 오버라이딩
    - `operator`는 생략 가능 (상위 클래스에 붙어 있음)

### 예시

```kotlin
class Point(val x: Int, val y: Int) {
    override fun equals(other: Any?): Boolean {
        if (other === this) return true
        if (other !is Point) return false
        return x == other.x && y == other.y
    }
}
```

## 순서 비교 연산자: `<`, `>`, `<=`, `>=`

- 내부적으로 `compareTo()` 함수로 컴파일됨
    - e.g. `a < b` → `a.compareTo(b) < 0`
- `compareTo()`는 `Comparable<T>` 인터페이스에서 제공됨
- `operator`는 상위 인터페이스에 이미 붙어 있으므로 생략 가능

### 예시

```kotlin
class Person(val firstName: String, val lastName: String) : Comparable<Person> {
    override fun compareTo(other: Person): Int {
        return compareValuesBy(this, other, Person::lastName, Person::firstName)
    }
}
```

### 코틀린에서 비교 연산자는 **간결하고 안전하게 작성 가능**

- `==` / `!=`은 `null` 안전
- `<`, `>`는 자바보다 훨씬 간결
- `compareValuesBy(...)` 사용 시 가독성 높은 비교 가능

# 컬렉션과 범위에 대해 쓸 수 있는 관례

## `get`과 `set` – 인덱스 접근 연산자

- `a[i]` → `a.get(i)`
- `a[i] = v` → `a.set(i, v)`
- 여러 인자를 받을 수도 있음 (예: `a[row, col]`)
- **사용 예:**
    
    ```kotlin
    val p = Point(10, 20)
    println(p[0]) // 10
    ```
    

## `in` – 포함 여부 검사

- `x in c` → `c.contains(x)`
- `x !in c` → `!c.contains(x)`
- 컬렉션뿐 아니라 사용자 정의 클래스에서도 가능
- **사용 예:**
    
    ```kotlin
    val rect = Rectangle(Point(0, 0), Point(100, 100))
    if (Point(10, 20) in rect) println("포함됨")
    ```
    

## `rangeTo` / `rangeUntil` – 범위 생성

- `a..b` → `a.rangeTo(b)` (닫힌 범위, `b` 포함)
- `a..<b` → `a.rangeUntil(b)` (열린 범위, `b` 미포함)
- `Comparable<T>`만 구현하면 자동 지원
- **주의:** 산술 연산자보다 우선순위가 낮기 때문에 괄호 필요할 수 있음
- **사용 예:**
    
    ```kotlin
    (0..9).forEach { print(it) } // 0123456789
    ```
    

## `iterator` – for 루프 지원

- `for (x in c)` → `c.iterator()` 호출
- 이터레이터는 `hasNext()`와 `next()`를 가진 객체
- 사용자 정의 클래스에도 확장 함수로 iterator 제공 가능
- **사용 예:**
    
    ```kotlin
    operator fun ClosedRange<LocalDate>.iterator(): Iterator<LocalDate> = ...
    
    ```
    

### 핵심 요점 정리

| 연산자 | 대응 함수 | 예시 |
| --- | --- | --- |
| `a[i]` | `get(i)` | `p[0]` → `p.get(0)` |
| `a[i] = v` | `set(i, v)` | `p[0] = 10` |
| `x in c` | `contains(x)` | `if (x in set)` |
| `a..b` | `rangeTo(b)` | `1..10` |
| `a..<b` | `rangeUntil(b)` | `1..<10` |
| `for (x in c)` | `iterator()` | `for (x in list)` |

# component 함수를 사용한 구조 분해 선언

코틀린에서는 **구조 분해 선언(destructuring declaration)** 을 통해 복합 객체에서 값을 **편리하게 여러 변수로 나눠** 받을 수 있다. 이를 가능하게 하는 것은 `componentN()` 함수이다.

## 구조 분해 선언 사용 방법

```kotlin
val p = Point(10, 20)
val (x, y) = p  // p.component1(), p.component2() 호출됨
```

- 구조 분해 선언은 내부적으로 `component1()`, `component2()` 등의 함수 호출로 변환됨
- `data class`는 생성자 파라미터에 대해 자동으로 `componentN()` 함수 생성

## `componentN()` 직접 정의하기

```kotlin
class Point(val x: Int, val y: Int) {
    operator fun component1() = x
    operator fun component2() = y
}
```

## 구조 분해를 활용한 다중 반환

- 함수가 여러 값을 반환할 때 `data class`를 사용

```kotlin
data class NameComponents(val name: String, val extension: String)

fun splitFilename(fullName: String): NameComponents {
    val parts = fullName.split('.', limit = 2)
    return NameComponents(parts[0], parts[1])
}

val (name, ext) = splitFilename("example.kt")
```

## 컬렉션에서 구조 분해 사용

- `List`, `Array`, `Map.Entry`는 이미 `componentN()` 제공

```kotlin
val list = listOf("A", "B")
val (first, second) = list

val map = mapOf("JetBrains" to "Kotlin")
for ((key, value) in map) {
    println("$key -> $value")
}
```

## 람다에서 구조 분해 사용

```kotlin
map.forEach { (key, value) -> println("$key -> $value") }
```

### 사용하지 않는 변수 무시 (`_`)

```kotlin
val (firstName, _, age) = person
```

### ⚠️ 구조 분해의 한계

- **순서 기반**이므로 `data class`의 **프로퍼티 순서 변경** 시 **오류 없이 잘못된 데이터가 할당**될 수 있음
- 구조 분해는 **작고 안정적인 클래스에만 사용하는 것이 권장**
- 향후 **이름 기반 구조 분해**가 Kotlin에 도입될 가능성 있음

### 핵심 요약

| 구조 분해 대상 | 자동 제공 여부 | 활용 |
| --- | --- | --- |
| `data class` | ✅ (자동 생성) | 함수 반환값, 루프 등 |
| 일반 클래스 | ❌ (직접 구현) | `operator fun componentN()` 정의 필요 |
| `Map.Entry`, `List` 등 | ✅ (표준 지원) | 루프나 람다에서 사용 가능 |
| `Pair`, `Triple` | ✅ | 이름 없는 반환값 처리 가능 |
| 사용하지 않는 값 | ✅ `_` 사용 | `val (_, _, age) = ...` |

# 프로퍼티 접근자 로직 재활용: 위임 프로퍼티

**위임 프로퍼티(delegated property)** 는 프로퍼티의 getter/setter 로직을 **다른 객체(위임 객체)** 에 위임하는 Kotlin의 기능이다. `by` 키워드를 사용하며, **중복 코드 제거**, **로직 재활용**, **유연한 저장 위치 지정** 등의 장점이 있다.

```kotlin
var p: Type by Delegate()
```

## 내부 동작

컴파일러는 `get()` / `set()` 호출을 위임 객체의 다음 함수로 변환함:

```kotlin
get() = delegate.getValue(thisRef, property)
set(value) = delegate.setValue(thisRef, property, value)
```

- `thisRef`: 프로퍼티가 포함된 클래스의 인스턴스
- `property`: 프로퍼티 자체 (`KProperty`)

## 기본 시그니처

```kotlin
operator fun getValue(thisRef: Any?, property: KProperty<*>): T
operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T)
```

## 대표 예시: `by lazy` (지연 초기화)

```kotlin
val emails by lazy { loadEmails(this) }
```

- 최초 접근 시 한 번만 초기화
- 기본적으로 **thread-safe**

## `Delegates.observable` 사용 예

```kotlin
var age by Delegates.observable(0) { prop, old, new ->
    println("${prop.name} changed from $old to $new")
}
```

- 값 변경 시 알림 가능
- 변경 감지 로직 중복 제거

## 사용자 정의 위임 객체 만들기

```kotlin
class ObservableProperty(var value: Int, val onChange: (String, Int, Int) -> Unit) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>) = value
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: Int) {
        val oldValue = value
        value = newValue
        onChange(property.name, oldValue, newValue)
    }
}
```

사용 예:

```kotlin
var salary by ObservableProperty(1000) { name, old, new ->
    println("$name changed from $old to $new")
}
```

## 표준 제공 확장: Map 기반 위임

```kotlin
class Person {
    private val attributes = mutableMapOf<String, String>()
    var name: String by attributes
}
```

- 표준 라이브러리가 `Map<K, V>`에 대해 `getValue` / `setValue` 제공
- 프로퍼티 이름이 key로 사용됨

## 프레임워크 활용: Exposed 예

```kotlin
class User(id: EntityID) : Entity(id) {
    var name: String by Users.name  // 컬럼 값 접근
    var age: Int by Users.age
}
```

- `Users.name`은 `Column<String>` 객체이며, 내부적으로 `getValue`, `setValue` 구현
- 데이터베이스 칼럼과 객체 프로퍼티가 자동으로 연결됨

### 정리

| 사용처 | 이점 |
| --- | --- |
| `by lazy` | 비용 큰 연산의 지연 초기화 |
| `Delegates.observable` | 변경 통지 기능 (예: UI 바인딩) |
| `Map` 위임 | 동적 프로퍼티 (key → property 이름 사용) |
| DB 컬럼 (`Exposed`) | 엔티티-테이블 매핑 자동화 |
| 사용자 정의 (`ObservableProperty`) | 공통 getter/setter 추출로 중복 제거 |

# 챕터 요약

### 🔢 연산자 오버로딩

- Kotlin은 **정해진 이름의 함수**를 정의함으로써 `+`, , , `+=` 등 **수학 연산자**를 오버로딩할 수 있음.
- 새로운 연산자를 정의할 수는 없지만 **중위 함수**를 대안으로 사용 가능 (`infix` 키워드).

---

### ⚖️ 비교 연산자

- `==`, `!=` → `equals`로 변환됨
- `<`, `>`, `<=`, `>=` → `compareTo`로 변환됨
- 데이터 클래스는 자동으로 `equals`/`compareTo` 구현 가능

---

### 📦 컬렉션 관련 관례

- `get(index)` / `set(index, value)` → `[]` 연산자 사용
- `contains()` → `in` 연산자 사용
- `iterator()` → `for (item in collection)` 루프 지원

---

### 📐 범위와 이터레이션

- `rangeTo` 함수 (`..`) 로 범위 생성
- `until` (`..<`)로 열린 범위 생성
- `iterator()` 관례로 사용자 정의 타입도 for 루프 지원

---

### 🧩 구조 분해 선언 (Destructuring Declaration)

- `componentN()` 함수를 통해 객체를 여러 변수로 분해 가능
- 데이터 클래스는 자동으로 `componentN()` 지원
- `_` 문자를 사용해 특정 값 무시 가능
- 맵, 리스트, 람다 인자에도 구조 분해 적용 가능

---

### 🪝 위임 프로퍼티 (Delegated Property)

- `by` 키워드를 통해 **getter/setter 로직을 다른 객체에 위임**
- 주요 사용 사례:
    - `by lazy { ... }`: 지연 초기화
    - `by Delegates.observable(...)`: 값 변경 감지
    - `by map`: 맵 기반 속성 저장
- 직접 `getValue` / `setValue`를 구현하여 커스텀 위임 객체 생성 가능
- 프레임워크(예: ORM, DSL 등)에서 강력한 도구로 활용 가능
