# 설계에서 DSL로: 표현력이 좋은 커스텀 코드 구조 만들기

- 소프트웨어의 **가독성과 유지 보수성**을 높이기 위해서는 클래스 자체보다 **클래스 간의 상호작용(API)** 설계가 중요하다.
- 라이브러리뿐 아니라 모든 애플리케이션 구성 요소도 다른 객체와의 인터페이스를 **명확하고 이해하기 쉬운 API**로 제공해야 한다.

<aside>
💡

- **읽기 쉬움**: 이름과 개념이 명확하여 어떤 동작이 일어날지 쉽게 이해할 수 있어야 한다.
- **간결함**: 불필요한 구문이나 준비 코드가 없어야 한다
</aside>

## DSL(Domain-Specific Language)이란?

- **특정 도메인**에 최적화된 언어
- 일반적인 프로그래밍 언어(GPL: General Purpose Language)보다 **제한된 기능**을 갖지만, 특정 목적에 더 적합
- 예시: **SQL, 정규식**

## DSL의 장점

- **선언적 표현**이 가능 (무엇을 할지 표현, 어떻게 할지는 DSL 엔진이 처리)
- **간결하고 직관적**
- **도메인 문제에 집중 가능**

## DSL의 단점

- 호스트 언어(GPL)와의 **통합 어려움**
- IDE 지원 부족, **디버깅 불편**
- 별도의 문법 학습 필요

## Kotlin의 internal DSL

- Kotlin은 DSL을 **코드 내부에 직접 포함할 수 있게 해주는 문법적 특징들**을 제공
- Kotlin DSL은 **정적 타입 시스템** 위에서 동작 → IDE 자동완성, 컴파일 시점 오류 탐지, 리팩터링 지원 가능
- Kotlin DSL은 `메서드 체이닝`이나 `수신 객체 지정 람다` 등을 활용하여, **마치 언어 자체처럼 보이게 할 수 있다**

```kotlin
val yesterday = clock.System.now() - 1.days

fun createSimpleTable() = createHTML {
    table {
        tr {
            td { +"cell" }
        }
    }
}
```

# 구조화된 API 구축: DSL 에서 수신 객체 지정 람다 사용

- **수신 객체 지정 람다**는 DSL에서 **구조화된 코드 작성**을 가능하게 하는 핵심 기능.
- 일반 람다는 인자(`it`)로 수신 객체를 참조해야 하지만, 수신 객체 지정 람다는 **`this` 생략 가능**.
- 수신 객체 지정 람다는 **확장 함수 타입**으로 표현됨

```kotlin
fun buildString(builderAction: StringBuilder.() -> Unit): String {
    val sb = StringBuilder()
    sb.builderAction() // 수신 객체 지정 방식
    return sb.toString()
}
```

- `StringBuilder.() -> Unit`는 `StringBuilder`를 수신 객체로 가지는 **확장 함수 타입**

### 수신 객체 지정 람다를 변수에 저장

```kotlin
val appendExcl: StringBuilder.() -> Unit = {
    append("!")
}
```

- 마치 `StringBuilder`의 메서드처럼 `appendExcl()` 호출 가능.

## HTML 빌더 예제: DSL에서 수신 객체 지정 람다 활용

### 구조화된 코드의 예

```kotlin
createHTML().table {
    tr {
        td { +"cell" }
    }
}
```

- `table`, `tr`, `td`는 전부 **수신 객체 지정 람다를 받는 함수**임.
- `+"cell"`은 `unaryPlus` 연산자 오버로딩으로 구현됨.
- 각 람다 안에서는 **해당 수신 객체의 멤버 함수만 사용 가능**.

### 수신 객체 명시 예

```kotlin
table {
    this@table.tr {
        this@tr.td {
            +"cell"
        }
    }
}
```

### 스코프 충돌 방지: `@DslMarker`

```kotlin
@DslMarker
annotation class HtmlTagMarker

@HtmlTagMarker
open class Tag
```

- 동일 마커가 붙은 수신 객체가 **동시에 두 개 이상 접근 불가**.
- 예: `<a>` 안의 `<img>` 람다에서 `a.href = ...` 사용 불가 (컴파일 오류 발생).

## 코틀린 빌더: 추상화와 재사용

### 중복 DSL 코드 함수화

- HTML, SQL 등 내부 DSL도 일반 함수처럼 재사용 가능
- 예: 책 목차 리스트를 생성하는 공통 DSL 코드 블록을 함수로 추출 가능

# invoke 관례를 사용해 더 유연하게 블록 내포시키기

`invoke` 관례를 사용하면 클래스의 인스턴스를 **함수처럼 호출**할 수 있게 되어 DSL에서 간결하고 유연한 구문을 구성할 수 있다. 

이 관례는 `operator fun invoke(...)` 메서드를 통해 동작하며, 수신 객체 지정 람다와 함께 사용할 때 DSL 작성에 특히 유용하다.

## invoke 관례란?

- `invoke`는 **특별한 이름을 가진 메서드**이며, 클래스에 `operator fun invoke(...)`로 정의하면 객체를 함수처럼 호출할 수 있다.

```kotlin
class Greeter(val greeting: String) {
    operator fun invoke(name: String) {
        println("$greeting, $name!")
    }
}

fun main() {
    val greeter = Greeter("Hello")
    greeter("Kotlin")  // → greeter.invoke("Kotlin") 으로 변환됨
}
```

### 일반 함수 관례와의 비교

- `get(index)` → `obj[index]`와 같이 단축 호출 가능.
- `invoke(args...)` → `obj(args...)`로 호출 가능.
- `invoke`는 파라미터 시그니처에 제한이 없어 오버로딩도 가능하다.

### 람다와의 관계

- 람다는 사실상 `FunctionN` 인터페이스를 구현한 객체이며, 내부적으로 `invoke` 메서드를 사용.
- 예: `lambda(1, 2)`는 `lambda.invoke(1, 2)`로 컴파일된다.

## DSL에서 invoke 관례 활용

### DSL 유연성 확장

Gradle의 `dependencies { ... }` 문법처럼 **함수 호출형**과 **블록 내포형** 두 가지 사용 방식을 모두 지원하고자 할 때 유용하다.

```kotlin
dependencies.implementation("lib:version")  // 일반 함수 호출
dependencies {
    implementation("lib:version")           // invoke 블록
}
```

### DSL 예제 구현

```kotlin
class DependencyHandler {
    fun implementation(coordinate: String) {
        println("Added dependency on $coordinate")
    }

    operator fun invoke(body: DependencyHandler.() -> Unit) {
        body()  // this는 DependencyHandler
    }
}

fun main() {
    val dependencies = DependencyHandler()

    // 함수 스타일 호출
    dependencies.implementation("org.example:lib:1.0")

    // DSL 스타일 호출
    dependencies {
        implementation("org.example:lib:2.0")
    }
}
```

### 작동 방식

- `dependencies { ... }` → `dependencies.invoke { ... }`로 변환됨
- 람다의 수신 객체가 `DependencyHandler`이므로, 내부에서 `implementation(...)`을 바로 호출 가능
- 코드 요약 흐름:
    
    ```kotlin
    dependencies {
        implementation(...)  // this = DependencyHandler
    }
    → dependencies.invoke {
        this.implementation(...)
    }
    ```
    

### 효과

- **코드 간결성** 증가
- **DSL 표현력** 향상
- **기존 API와 병행 사용 가능** (일반 함수 호출 & DSL 블록 호출)

# 실전 코틀린 DSL

## 중위 호출 연쇄시키기: 테스트 DSL 에서 `should`

- **DSL의 핵심**은 읽기 쉬운 문법 구성이다.
- 중위 함수(`infix`)는 연쇄적인 함수 호출을 자연스러운 문장처럼 보이게 한다.

```kotlin
val s = "kotlin".uppercase()
s should startWith("K")
```

<aside>
💡

이거 보고 코틀린 테스팅 프레임워크를 써보고 싶어졌네요

</aside>

- `should`는 중위 함수로 정의:
    
    ```kotlin
    infix fun <T> T.should(matcher: Matcher<T>) = matcher.test(this)
    ```
    
- `Matcher<T>`는 특정 조건을 검사하는 인터페이스:
    
    ```kotlin
    interface Matcher<T> {
        fun test(value: T)
    }
    ```
    
- `startWith()`는 `Matcher<String>` 구현 반환:
    
    ```kotlin
    fun startWith(prefix: String): Matcher<String> = object : Matcher<String> {
        override fun test(value: String) {
            if (!value.startsWith(prefix)) throw AssertionError("$value does not start with $prefix")
        }
    }
    ```
    

### H2.4 효과

- 테스트 코드가 **일반 영어 문장처럼 읽힘**
- **가독성 높은 DSL** 제공
- **중위 호출 + 객체식 표현**은 깔끔한 문법 구성의 핵심

---

## 13.4.2 원시 타입에 대한 확장 함수: 날짜 DSL

### H2.1 목표

- `1.days`, `5.hours`와 같은 **직관적인 시간 표현** DSL 제공

### H2.2 구현 예시

```kotlin
val Int.days: Duration
    get() = this.toDuration(DurationUnit.DAYS)

val Int.hours: Duration
    get() = this.toDuration(DurationUnit.HOURS)

```

- 추가 단위 정의도 가능:
    
    ```kotlin
    val Int.fortnights: Duration
        get() = (this * 14).toDuration(DurationUnit.DAYS)
    
    ```
    

### 효과

- 숫자 상수를 **자연어처럼** 사용할 수 있음
- 날짜/시간 연산을 **간결하게 표현**

## 멤버 확장 함수: SQL DSL

- 멤버 확장 함수는 DSL에서 **문맥에 맞는 함수 호출만 허용**하게 하며 **의미 없는 오용을 방지**함

### 테이블 선언 예시 (Exposed)

```kotlin
object Country : Table() {
    val id = integer("id").autoIncrement().primaryKey()
    val name = varchar("name", 50)
}
```

- `integer`, `varchar`: 테이블 정의용 메서드
- `autoIncrement`, `primaryKey`: `Column`에 대한 멤버 확장

```kotlin
fun Column<Int>.autoIncrement(): Column<Int> { ... }  // Table 멤버 확장
fun Column<T>.primaryKey(): Column<T> { ... }
```

### 멤버 확장의 장점

- `Column<Int>`만 `autoIncrement()` 가능하게 제한
- 함수 사용 범위를 `Table` 문맥 안으로 제한 → **API 안전성 향상**

### H한계와 대응

- **확장성 제약**: 외부에서 `Table`에 새로운 멤버 확장을 추가 불가
- **해결 방향**: [KEEP](http://mng.bz/J67Ge)에서 제안한 **콘텍스트 수신 객체(context receiver)**

```kotlin
context(Table)
fun Column<Int>.autoIncrement(): Column<Int> { ... }
```

## SQL 쿼리 DSL: `select` + `eq`

```kotlin
val result = (Country innerJoin Customer)
    .select { Country.name eq "USA" }

result.forEach { println(it[Customer.name]) }
```

### 내부 구조

- `select` 함수:
    
    ```kotlin
    fun Table.select(where: SqlExpressionBuilder.() -> Op<Boolean>): Query
    ```
    
- `eq` 함수: `SqlExpressionBuilder`의 멤버 확장
    
    ```kotlin
    object SqlExpressionBuilder {
        infix fun <T> Column<T>.eq(value: T): Op<Boolean> { ... }
    }
    ```
    

### 작동 방식

- `select` 내부에서 `SqlExpressionBuilder`가 **암시적 수신 객체**
- 그 안에서 `eq`, `isNull` 등 다양한 조건 조합 가능

### 멤버 확장의 활용

- `Table` 문맥에서만 동작하는 조건 정의
- **문맥 기반 조건 표현** 가능

# 오늘의 문제

```kotlin
fun main() {
    MyForm {
        Title("회원가입")
        Input("이메일")
        Input("비밀번호")
        SubmitButton("제출")
    }
}
```

```kotlin
<form>
  <h2>회원가입</h2>
  <label>이메일</label><input />
  <label>비밀번호</label><input />
  <button>제출</button>
</form>
```

위와 같은 DSL 사용 코드를 가능하게 하는 Kotlin DSL 함수와 빌더 클래스를 구현하시오.

- `MyForm` 함수는 `MyFormBuilder`를 수신 객체로 받는 확장 함수여야 한다.
- `Title(text: String)`, `Input(label: String)`, `SubmitButton(text: String)` 함수를 `MyFormBuilder` 내부에 `infix`가 아닌 일반 함수로 정의한다.
- 각 함수는 호출될 때 콘솔에 HTML 태그 형태로 출력해야 한다.
- `MyForm` 블록의 시작과 끝에 `<form>`과 `</form>`을 출력해야 한다.
