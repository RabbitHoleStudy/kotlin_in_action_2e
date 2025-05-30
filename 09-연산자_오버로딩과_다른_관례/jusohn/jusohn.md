# 09장 - (p.385 - p.429)

## 관례

- 어떤 언어 기능과 미리 정해진 이름의 함수를 연결해 주는 기법
- 코틀린은 관례에 의존
  - 이유: 자바 클래스를 코틀린 언어에 적용하기 위함

## 산술 연산자를 오버로딩해 임의의 클래스에 대한 연산을 더 편리하게 만들기

### `plus`, `times`, `divide` 등: 이항 산술 오버로딩

```kotlin
data class Point(val x: Int, val y: Int) {
	operator fun plus(other: Point): Point {
		return Point(x + other.x, y + other.y)
	}
}
```

- 연산자를 오버로딩하는 함수 앞에는 반드시 `operator` 키워드가 있어야 함
  - 키워드를 붙임으로써 어떤 함수가 관례를 따르는 함수임을 명확히 할 수 있고, 실수로 관례에서 사용
    하는 함수 이름을 사용하는 경우를 막아줌
- 코틀린에서는 프로그래머가 직접 연산자를 만들어 사용할 수 없고, 언어에서 미리 정해둔 연산자만 오버로딩할 수 있으며, 관례에 따르기 위해 클래스에서 정의해야 하는 이름이 연산자별로 정해져 있음
- 직접 정의한 함수를 통해 구현하더라도, 연산자 우선순위는 언제나 표준 숫자 타입에 대한 연산자 우선순위와 같음
- 일반 함수와 마찬가지로, `operator` 함수도 오버로딩 가능
- 오버로딩 가능한 이항 산술 연산자
  - `times`
  - `div`
  - `mod`
  - `plus`
  - `minus`

### 연산을 적용한 다음에 그 결과를 바로 대입: 복합 대입 연산자 오버로딩

- `plus` 와 같은 연산자를 오버로딩하면 코틀린은 `+` 연산자뿐 아니라 그와 관련 있는 연산자인 `+=` 도 자동으로 함께 지원
- `+=`, `-=` 등의 연산자를 복합 대입 연산자라고 함

- 반환 타입이 `Unit`인 `plusAssign` 함수를 정의하면서 `operator` 로 표시하면 코틀린은 `+=` 연산자에 그 함수를 사용
- 다른 복합 대입 연산자 함수도 비슷하게 `minusAssign`, `timesAssign` 등의 이름올 사용

```kotlin
operator fun <T> MutableCollection<T>.plusAssign(element: T) {
	this.add(element)
}
```

- 일반적으로, 새로운 클래스를 일관성 있게 설계하는 것이 가장 좋음
- `plus` 와 `plusAssign` 을 동시에 정의핮지 말 것

- 코틀린 표준 라이브러리는 컬렉션에 대해 2가지 접근 방법을 함께 제공
- `+` 와 `-` 는 항상 새로운 컬렉션을 반환하며
- `+=` 와 `-=` 연산자는 항상 변경 가능한 컬렉션에 작용해 메모리에 있는 객쳬 상태를 변화시킴
- 또한 읽기 전용 컬렉션에서 `+=` 와 `-=` 는 변경을 적용한 복사본을 반환

### 피연산자가 1개뿐인 연산자: 단항 연산자 오버로딩

```kotlin
operator fun Point.unaryMinus(): Point {
	return Point(-x, -y)
}

fun main() {
	val p = Point(10, 20)
	println(-p)
	// Point(x=-10, y=-20)
}
```

- 오버로딩 가능한 단항 연산자
  - `unaryPlus`
  - `unaryMinus`
  - `not`
  - `inc`
  - `dec`

## 비교 연산자를 오버로딩해서 객체들 사이의 관계를 쉽게 검사

- 자바와 달리, 코틀린에서는 `==` 비교 연산자를 직접 사용할 수 있어 비교 코드가 `equals` 나 `compareTo` 를 사용한 코드보다 더 간결하며 이해하기 쉬움

- 코틀린이 `==` 연산자 호출을 `equals` 메서드 호출로 컴파일한다는 사실을 배움
  - 사실 이는 특별한 경우는 아니고 지금껏 설명한 관례라는 원칙을 적용한 것에 불과
- `≠` 연산자를 사용하는 식도 `equals` 호출로 컴파일됨
- `==` 와 `≠` 는 내부에서 인자가 `null` 인지 검사하므로, 다른 연산과 달리 널이 될 수 있는 값에도 적용 가능

```kotlin
class Point(val x: Int, val y: Int) {
	override fun equals(obj: Any?): Boolean {
		if (obj === this) return true
		if (obj !is Point) reutrn false
		return obj.x == x && obj.y == y
	}
}
// 완전한 구현은 hashCode 구현도 함께 제공해야 함
```

- `equals` 함수에는 `override` 가 붙어있음

  - 다른 연산자 오버로딩 관례와 달리 `equals` 는 `Any` 에 적용된 메서드이므로 `override` 가 필요
  - `equals`를 구현할 때는 `===` 를 사용해 자신과의 비교를 최적화하는 경우가 많음
    - `===` 를 오버로딩할 수는 없다는 사실올 기억

- 자바에서 정렬이나 최댓값 , 최솟값 등 값을 비교해야 하는 알고리듬에 사용할 클래스는 `Comparable` 인터페이스를 구현해야 함
- `Comparable` 에 들어있는 `compareTo` 메서드는 한 객체와 다른 객체의 크기를 비교해 정수로 나타냄
- 코틀린도 똑같은 Comparable 을 지원
- `p1 < p2` 는 `p1.comapreTo(p2) < 0` 와 같음

- 비교 연산자를 자바 클래스에 대해 사용하기 위해 특별히 확장 메서드를 만들거나 할 필요는 없음

## 컬렉션과 범위에 대해 쓸 수 있는 관례

- 인덱스 접근 연산자를 사용해 원소를 읽는 연산은 `get`
- 원소를 쓰는 연산은 `set` 연산자 메소드로 변환됨
- Map 과 MutableMap 인터페이스에는 이미 그 두 메서드가 들어있음

```kotlin
operator fun Point.get(index: Int): Int {
	return when(index) {
		0 -> x
		1 -> y
		else ->
			throw IndexOutOfBoundsException("Invalid coordinate $index")
	}
}
```

- `get`이라는 메서드를 만들고 `operator` 변경자를 붙이기만 하면 됨

- 비슷하게 인덱스에 해당하는 컬렉션 원소를 각괄호를 사용해 쓰는 함수를 정의할 수도 있음
- Point 클래스는 불변 클래스이므로 이런 메서드를 정의하는 것이 의미가 없음

### 어떤 객체가 컬렉션에 들어거있는지 검사: `in` 관례

- `in` 은 객체가 컬렉션에 들어있는지 검사 (맴버십 검사)
  - 대응하는 함수는 `contains`

```kotlin
data class Rectangle(val upperLeft: Point, val lowerRight: Point)

operator fun Rectangle.contains(p: Point): Boolean {
	return p.x in upperLeft.x..<lowerRight.x &&
		p.y in uppserLeft.y..<lowerRight.y
}
```

### 객체로부터 범위 만들기: `rangeTo` 와 `rangeUntil` 관례

- `rangeTo` 함수는 범위를 반환
- 여러분의 클래스에 대해서도 이 연산자를 정의할 수 있음
- 하지만 어떤 클래스가 `Comparable` 인터페이스를 구현하면 `rangeTo`를 정의할 필요가 없음

### 자신의 타입에 대해 루프 수행: `iterator` 관례

- `iterator` 메서드를 확장 함수로 정의할 수 있음
- 이런 성질로 인해 일반 자바 문자열에 대한 `for` 루프가 가능

### `component` 함수를 사용해 구조 분해 선언 제공

- 구조 분해 선언은 일반 변수 선언과 비슷해 보이지만, `=` 의 왼쪽에 여러 변수를 괄호로 묶었다는 점이 다름
- 내부에서 구조 분해 선언은 다시 관례를 사용
- 구조 분해 선언의 각 변수를 초기화하고자 `componentN` 라는 함수를 호출

```kotlin
class Point(val x: Int, val y: Int) {
	operator fun cornponentl() = x
	operator fun coniponent2() = y
}
```

- 코틀린 표준 라이브러리에서는 맨 앞의 다섯 원소에 대한 componentN 을 제공

- 함수에서 여러 값을 반환하는 더 단순한 방법은 표준 라이브러리의 Pair 나 Triple 클래스를 사용하는 것
- 이들을 사용하면 여러분이 클래스를 정의하지 않아도 되므로 코드를 적게 작성해도 되지만, Pair 나 Triple 은 그 안에 담겨있는 원소의 의미를 말해주지 않음

### 구조 분해 선언과 루프

### `_` 문자를 사용해 구조 분해 값 무시

## 프로퍼티 접근자 로직 재활용: 위임 프로퍼티

### 위임 프로퍼티의 기본 문법과 내부 동작

### 위임 프로퍼티 사용: by Lazy() 를 사용한 지연 초기화

### 위임 프로퍼티 구현

### 위임 프로퍼티는 커스텀 접근자가 있는 감춰진 프로퍼티로 변환된다

### 맵에 위임해서 동적으로 애트리뷰트 잡근

## 요약

- 코틀린은 정해친 이름의 함수를 정의함으로써 표준적인 수학 연산을 오버로드할 수 있게 해준다. 자신만의 연산자를 정의할 수는 없지만 중위 함수를 더 표현력이 좋은 대안으로 사용할 수 있다.
- 비교 연산자를 모든 객체에 사용할 수 있다. 비교 연산자는 equals 와 compareTo 메서드 호출로 변환된다.
- `get`, `set`, `contains` 라는 함수를 정의하면 코틀린 컬렉션과 비슷하게 여러분 클래스의 인스턴스에 대해 `[]`와 `in` 연산을 사용할 수 있다.
- 미리 정해진 관례를 따라 범위를 만들거나 컬렉션과 배열의 원소를 이터레이션할 수 있다.
- 구조 분해 선언을 통해 한 객쳬의 상태를 분해해서 여러 변수에 대입할 수 있다. 함수가 여러 값을 한꺼번에 반환해야 하는 경우 구조 분해가 유용하다. 데이터 클래스에 대해 구조 분해를 거저 사용할 수 있지만, 자신의 클래스에 `componetnN` 함수를 정의하면 구조 분해를 지원할 수 있다.
- 위임 프로퍼티를 통해 프로퍼티 값을 저장하거나 초기화하거나 읽거나 변경할 때 사용하는 로직을 재활용할 수 있다. 위임 프로퍼티는 프레임워크를 만들 때 아주 강력한 도구로 쓰인다.
- 표준 라이브러리 함수인 `lazy`를 통해 지연 초기화 프로퍼티를 쉽게 구현할 수 있다.
- `Delegates.observable` 함수를 사용하면 프로퍼티 변경올 관찰할 수 있는 옵저버를 쉽게 추가할 수 있다.
- 맵올 위임 객쳬로 사용하는 위임 프로퍼티를 통해 다양한 속성을 제공하는 객체를 유연하게 다룰 수 있다.

## 문제

### 1번

- 복소수를 나타내는 클래스 Complex 에 대해 +, -, \*, / 연산자 오버로딩을 구현해주쇼

```kotlin
data class Complex(val real: Double, val imag: Double) {
}
```

### 2번

- 크기가 다른 Box(val volume: Int) 클래스에 대해 >, <, >=, <= 비교가 가능하도록 compareTo를 구현해주세요

```kotlin
data class Box(val volume: Int): Comparable<Box> {
}
```
