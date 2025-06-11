# 13장 - DSL 만들기

생성 일시: 2025년 6월 4일 오후 5:32
스터디일자: 2025년 6월 11일

[책](https://www.notion.so/20fc8304a88c8040aae3fdeeec300378?pvs=21)

- DSL (Domain Specific Language)
도메인 특화 언어

# API 에서 DSL로

- 코틀린이 제공하는 간결한 문법 편의 기능

| 일반 구문 | 간결한 구문 | 사용한 언어 특성 |
| --- | --- | --- |
| `StringUtil.capitalize(s)` | `s.capitalize()` | 확장 함수 |
| `1.to("one")` | `1 to "one"` | 중위 호출 |
| `set.add(2)` | `set += 2` | 연산자 오버로딩 |
| `map.get("key")` | `map["key"]` | get 메서드에 대한 관례 |
| `file.use({ f -> f.read() })` | `file.use { it.read() }` | 람다를 괄호 밖으로 빼내는 관례 |
| `sb.append("yes")`<br>`sb.append("no")` | `with (sb) {`<br>`append("yes")`<br>`append("no")`<br>`}` | 수신 객체 지정 람다 |
| `val m = mutableListOf<Int>()`<br>`m.add(1)`<br>`m.add(2)`<br>`return m.toList()` | `return buildList { add(1)`<br>`add(2)`<br>`}`  | 람다를 받는 빌더 함수 |

- DSL은 컴파일 시점에 타입이 정해진다.

- 예시
    
    ```kotlin
    val yesterday = Clock.System.now() - 1.days
    ```
    
    - 하루전날 반환
    
    ```kotlin
    fun createSimpleTable() = createHTML()
    .table{
    	tr{
    		td { + "cell" }
    	}
    }
    ```
    
    - HTML 표를 생성하는 함수

# 도메인 특화 언어 DSL 이란?

- 범용 프로그래밍 언어 (General-purpose Programming Language) GPL

⇒ 여기에서 특정 도메인 영역에 초점을 맞추고, 필요없는걸 뺀 언어 ⇒ DSL

- 예시
    - SQL ⇒ 데이터베이스를 조작하기 위한 특정 작업 언어
    - 정규식 ⇒ 문자열을 조작하기 위한 특정 작업 언어
- 스스로 제공하는 기능을 제한
    - 클래스나 함수를 선언하는 것부터 시작할 필요가 없다.
    - 첫 키워드가 수행하려는 연산의 종류로 지정
    - 문법이 단순, 간단 (GPL 에 비해서)
- 선언적이다 (declarative)
    - GPL 은 명령적(imperative) ⇒ 각 단계를 순서대로 정확히 기술
    - 원하는 결과를 기술 → 결과를 얻는 세부 실행은 언어를 해석하는 엔진에 맡김

## DSL 과 범용 프로그래밍 언어

- DSL은 선언적(declarative) 이다.
    - 원하는 결과를 기술
    - 세부 실행은 언어를 해석하는 엔진에 맡김
    - 결과를 달성하는 과정은 전체적으로 한꺼번에 최적화 → 더 효율적일 확률이 높다
- 범용 프로그래밍은 명령적 (imperative)
    - 어떤 연산을 완수하기 위해 필요한 각 단계를 순서대로 정확히 기술
- Host 애플리케이션과 DSL 의 조합하기가 어렵다
    - 자체 문법이 있어서, 다른 언어에서 호출하기가 까다롭다.

⇒ 코틀린에서는 이런 문제를 해결하면서, 이점을 살리는 내부 DSL을 만들 수 있게 해준다.

## 예시

- Customer
    - 각자가 사는 나라에 대한 참조 (Country 레코드에 대한 외부 키)
- Country

⇒ 목표 : 가장 많은 고객이 살고 있는 나라 찾기

- 외부 DSL
    - SQL
        
        ```sql
        SELECT Country.name, COUNT (Customer.id)
        	 FROM Country
        	 
        INNER JOIN Customer
        		ON Country.id = Customer.country_id
        	GROUP BY Country.name
        	ORDER BY COUNT (Customer.id) DESC
        	LIMIT 1
        ```
        
- 내부 DSL
    - Exposed ( 코틀린으로 작성된 DB 프레임웍)
    - Exposed 사용
        
        ```kotlin
        (Country innerJoin Customer)
        	.slice(Country.name, Count(Customer. id))
        	.selectAll()
        	.groupBy(Country. name)
        	.orderBy(Count(Customer.id), order = SortOrder.DESC)
        	.limit(1)
        ```
        
        - selectAll
        - groupBy
        - orderBy
        
        등은 일반 코틀린 메서드다.
        
    - 질의를 실행한 결과가 네이티브 코틀린 객체
    - 내부 DSL

# DSL 구조

- 다른 API 에는 존재하지 않는, “구조”와 “문법”이 존재
- 전형적인 라이브러리
    - 여러 메서드로 이뤄진다
    - 클라이언트 → N 개의 메서드를 한번에 하나씩 호출 → 라이브러리 사용
    - 함수 호출 시퀀스에는 내포나 그룹 같은 아무런 구조가 없다.
    - 호출과 다른 호출 사이 context가 유지되지 않는다.
    - Command - query (명령 - 질의)
- DSL 메서드 호출
    - 람다를 내포시키거나, 메서드 호출을 연쇄시키는 방식으로 구조를 만든다
    - 여러 함수 호출을 조합해서 연산을 만든다.
    - 타입 검사기는 여러 함수 호출이 바르게 조합됐는지를 검사

## 비교

```kotlin
dependencies {
	testlmplementation(kotlin("test"))
	implementation("org.jetbrains.exposed:exposed-core:e.40.1")
	implementation("org.jetbrains.exposed:exposed-dao;O.40.1")
}
```

- 람다 Nested 통해 구조를 만든다

```kotlin
project.dependencies.add("testInv)1eaientation ", kotlin("test"))
project.dependencies.add( "implementation",
	"org.jetbrains.exposed:exposed-core:e.40.1")
project.dependencies.add("impleeentation",
	"org.jetbrains.exposed: exposed-dao:O.40.1")
```

- 일단 dependencies

### 코테스트

```kotlin
str should startWith("kot")
```

- 메서드 호출을 연쇄시켜 구조를 만든다

```kotlin
assertTrue(str.startsWith("kot"))
```

- 일반 Junit 스타일 API

---

# 내부 DSL로 HTLM 만들기 예제

- kotlinx.html
- 셀이 하나인 표를 만드는 코드
    
    ```kotlin
    import kotlinx.html.stream.createHTML
    import kotlinx.html.*
    
    fun createSimplelable() = createHTML().
    	table {
    		tr {
    			td { + "cell" }
    		}
    	}
    ```
    

- createSimpleTable
    - 아래의 HTML 조각이 들어있는 문자열을 반환한다.
    
    ```html
    <table>
    	<tr>
    		<td>cell</td>
    	</tr>
    </table>
    ```
    
    - td 를 tr 안에서만 써야한다. ⇒ 아니면 컴파일 되지 않는다.
    - 이런 룰을 지키면서 생성 가능
    - 표를 정의하면서, 동적으로 표의 셀을 생성
        
        ```kotlin
        import kotlinx.html.stream.createHTML
        import kotlinx.html.*
        
        fun createAnotherTable() = createHTML().table {
        	val numbers = mapOf(1 to "one", 2 to "two")
        	for ((num, string) in numbers) {
        		tr {
        			td { "$num" }
        			td { +string }
        		}
        	}
        }
        ```
        
        ```html
        //결과
        <tr>
        	<td>1</td>
        	<td>one</td>
        </tr>
        <tr>
        	<td>2</td>
        	<td>two</td>
        </tr>
        </table>
        ```
        

# DSL 에서 수신 객체 지정 람다 사용

## 수신 객체 지정 람다와 확장 함수 타입

### buildString 예제1

```kotlin
fun buildString( builderAction: (StringBuilder) -> Unit //함수 타입 파라미터 정의
): String {
	val sb = StringBuilder()
	builderAction(sb) // 람다 인자로 StringBuilder 클래스를 넘긴다.
	return sb.toString()
}

fun main() {
	val s = buildString {
			it.append("Hello, ") //it 는 StringBuilder 인스턴스
			it.append("World! ")
		}
	println(s) // Hello, World!
}
```

- 람다 본문에서 매번 `it` 로 `StringBuilder` 인스턴스를 참조해야한다.
- append 처럼 더 간단하게 호출하고 싶다 ⇒ 수신 객체 지정 람다
    
    ```kotlin
    fun buildString( builderAction: StringBuilder.() -> Unit //수신객체 지정 함수 타입
    ): String {
    	val sb = StringBuilder()
    	sb.builderAction() // StringBuilder 인스턴스를 람다의 수신객체로
    	return sb.toString()
    }
    
    fun main() {
    	val s = buildString {
    			this.append("Hello, ") //this 생략 가능
    			append("World! ")
    		}
    	println(s) // Hello, World!
    }
    ```
    
    - `(StringBuilder) → Unit`
    - `StringBuilder.()→ Unit`
        
        ![image.png](image.png)
        
        - 수신 객체 타입
        
        ![image.png](image%201.png)
        
        - 동작 구조

### 예제2

- 함수 타입의 변수를 정의할 수 있다.
- (함수가 담긴)변수를 마치 확장 함수처럼 호출하거나,
- 수신 객체 지정 람다를 요구하는 전달인자로 넣을 수 있다.

```kotlin
fun buildString( builderAction: StringBuilder.() -> Unit //수신객체 지정 함수 타입
): String {
	val sb = StringBuilder()
	sb.builderAction() // StringBuilder 인스턴스를 람다의 수신객체로
	return sb.toString()
}

val appendExcl: StringBuilder.() -> Unit = { this.append("!") } 
// appendExcl은 확장 함수 타입

fun main() {
	val stringBuilder = StringBuilder("Hi")
	stringBuilder.appendExcl() // 확장 함수처럼 호출
	println(stringBuilder) // Hi !
	println(buildString(appendExcl))// appendExcl 을 인자로 넘길 수 있다. 
	// ! 
}
```

### 예제3 - 표준 라이브러리의 buildString

```kotlin
fun buildString(builderAction: StringBuilder.() -> Unit): String =
	StringBuilder().apply(builderAction).toString()
```

- apply : 인자로 받은 람다나 함수(여기서는 builderAction)를 호출하면서, 자신의 수신 객체를 람다나 함수의 암시적 수신객체로 사용한다.
    
    ![image.png](image%202.png)
    
    - with 와 apply 는 모두 자신이 제공받은 수신 객체를 갖고 확장 함수 타입의 람다를 호출
    - apply는 수신 객체를 암시적 인자(this)로 받는다 → 수신 객체를 return
    - with는 수신 객체를 첫 번째 파라미터로 명시적으로 받는다. → 람다를 호출해 얻은 결과를 return

⇒ 결과를 받아서 쓰는 경우가 아니면 두 함수를 바꿔 쓸 수 있다

```kotlin
fun main() {
	val map = mutableMapOf(1 to "one")
	map.apply { this[2] = "two"}
	with (map) { this[3] = "three" }
	println(map)
	// {1=one, 2=two, 3=three}
}
```

# HTML 빌더

```kotlin
import kotlinx.html.stream.createHTML
import kotlinx.html.*

fun createSimplelable() = createHTML().
	table {
		tr {
			td { + "cell" }
		}
	}
```

- table
- tr
- td

모두 평범한 함수다. 

- +”cell” ⇒ 함수호출이다.
- 각 수신 객체 지정 람다가 이름 결정 규칙을 바꾼다
    - tr은 table 람다 안에서만 작동된다
    - td 는 tr 안에서만 접근 가능하다
    - 단항 덧셈 연산자는 td 태그 안에서만 접근 가능
    
    ⇒ 이런 HTML 언어의 문법을 따르는 코드만 작성할 수 있다.
    

## 구조

```kotlin
open class Tag

class TABLE : Tag {
	fun tr(init : TR.() -> Unit)
}

class TR : Tag {
	fun td(init : TD.() -> Unit)
}

class TD : Tag
```

- 일반 람다로 표현한다면?
    
    ```kotlin
    fun createSimplelable() = createHTML().table {
    	this@table.tr {
    		(this@tr).td { + "cell" }
    		}
    	}
    }
    ```
    

### @DslMarker

- 내포된 람다에서 외부 람다의 수신 객체에 접근하지 못하게 제한
- 메타어노테이션으로, 어노테이션 클래스에 적용하는 어노테이션

```kotlin
@DslMarker
annotation class HtmllagMarker
```

- 수신객체가 2개가 될 수 없다.
- 모두 Tag 클래스의 하위 클래스이므로, @HtmlTagMarker 어노테이션을 Tag 클래스에 추가할 수 없다.

---

# Invoke 관례를 사용해 유연하게 블록 내포시키기

```kotlin
class Greeter(val greeting: String) {
	operator fun invoke(name: String){ 
		println("$greeting, $name")
	}
}

fun main() {
	val koreanGreeter = Greeter("Do you know kimchi?")
	koreanGreeter("Dmitry")
	// Do you know kimchi?, Dimitry
}
```

- Greeter 안에 inovke 메서드를 정의
- Greeter 인스턴스를 함수처럼 호출 할 수 있다.
- `koreanGreeter(”Dmitry”)` 는 내부적으로 `koreanGreeter.inovke(”Dmitry”)` 로 컴파일된다.
- invoke 시그니처는 요구사항이 없다.
    - 파라미터 개수, 타입 맘대로
    - 파라미터 타입을 지원하기 위해 inovke 오버로딩 가능

⇒ 람다 호출 (lambda() 처럼 람다 뒤에 괄호 붙이는 방식)이 실제로는 invoke 관례를 적용한 것

 

```kotlin
interface Function2<in P1, in P2, out R> {
	operator fun invoke(p1: P1, p2: P2): R
}
```

## DSL 의 invoke 관례

```kotlin
dependencies.implementation( "org.jetbrains.exposed: exposed-core :0.40.1")
dependencies {
	inp1ementation( "org.jetbrains.exposed: exposed-core :0.40.1")
}
```

⇒ 내포된 블록 구조 허용 + 평평한 함수 호출 구조 
두 방식 다 사용하고싶다

- DSL 사용자가 설정할 항목이 많으면, 내포된 블록 구조 사용
    - dependencies 변수에 대해 implementation 메서드를 호출
    - dependencies 안에 람다를 받는 inovke 메서드를 정의 → 두 번째 방식 호출 사용 가능
    - `dependencies. invoke({. . . })`
    
    ```kotlin
    class DependencyHandler {
    	fun implementation(coordinate: String) {
    		println("Added dependency on $coordinat&")
    	}
    	
    	operator fun invoke(
    		body: DependencyHandler.() -> Unit) {
    		body()
    	}
    }
    
    fun main() {
    	val dependencies = DendencyHandler()
    	dependencies.implementation( "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.O")
    //Added dependency on org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.0
    	
    	dependencies {
    			implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.5.0")
    		}
    	// Added dependency on org.jetbrains.kotlinx:kotlinx-datetime:0.5.O
    }
    ```
    
    - 첫번째 dependencies 추가시 compile 메서드를 직접 호출
    - 두번째는 이렇게 변환된다
        
        ```kotlin
        dependencies.invoke({
        	this.implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.5.0")
        })
        ```
        
    - dependencies 를 함수처럼 호출하면서 람다 인자로 넘긴다.
    

# 실전 코틀린 DSL

## 중위 호출 연쇄시키기

- 테스트 프레임워크의 should
    
    ```kotlin
    import io.kotest.matchers.should
    import io.kotest.matchers.string.startWith
    import org.junit.jupiter.api.Test
    
    class Prefixlest {
    	@Test
    	fun testKPrefix() {
    		val s."kotlin".uppercase()
    		s should startWith("K")
    	}
    }
    ```
    

```kotlin
infix fun <T> T.should(matcher: Matcher<T>) = matcher.test(this)
```

- should 함수는 Matcher의 인스턴스를 요구
- Matcher 는 값에 대한 단언문을 표현하는 제네릭 인터페이스
- startWith 는 Matcher를 구현→ 문자열이 뭐로 시작하는지 검사(실제 구현체)

```kotlin
interface Matcher<T> { fun test(value: T) }

fun startWith(prefix: String): Matcher<String> {
	return object : Matcher<String> {
		override fun test(value: String) {
			if( !value.startsWith(prefix)) {
					throw AssertionError("$value does not start with $prefix")
				}
			}
		}
}
```

(단순화)⇒실제 test 에 사용되는건 MatcherResult 반환

- DSL 문맥에서 중위 연산자를 호출

## 원시 타입에 대해 확장 함수 정의 → 날짜 처리

```kotlin
val now = Clock.System.now()
val yesterday = now - 1.days
val later = now + 5.hours
```

- 날짜와 시간을 다루기 위한 DSL

```kotlin
import kotlin.time.DurationUnit

val Int.days: Duration
	get() = this.toDuration(DurationUnit.DAYS)
val Int.hours: Duration
	get() = this.toOuration(DurationUnit.HOURS)
	
	
val Int.fortnights: Duration get() =
	(this * 14) .toDuration(DurationUnit.DAVS)
```

- days 와 hours 는 Int 타입의 확장 프로퍼티

## 멤버 확장 함수

- 클래스 안에서 확장 함수와 확장 프로퍼티를 선언
- 멤버 확장

```kotlin
object Country : Table() {
	val id = integer("id").autoIncrement()
	val name = varchar("name", 50)
	override val primaryKey = PrimaryKey(id)
}
```

```kotlin
fun main () {
	val db = Database.connect("jdbc:h2:mem:test", driver = "org.h2.Driver")
	
	transaction(db) {
		SchemaUtils.create(Country)
	}
	
}
/*
CREATE TABLE IF NOT EXISTS Country (
	id INT AUTO_INCREMENT NOT NULL,
	name VARCHAR(50) NOT NULL,
	CONSTRAINT pk_Country PRIMARY KEY (id)
)
*/
```

## 구조

```kotlin
class Table {
	fun integer(name: String): Column<Int>
	fun varchar(name: String, length: Int): Column<String>
	//...
}
```

- 각 컬럼의 속성을 지정하는 방법을 사용하면 멤버 확장이 쓰인다
    
    ```kotlin
    val id = integer("id").autoIncrement().primaryKey()
    ```
    
    - 컬럼의 속성 지정
        
        ```kotlin
        class Table {
        	fun Column<Int>. autolncrement(): Column<Int>
        	//..
        }
        ```
        
    - autoIncrement는 Table 클래스의 멤버지만, Column 의 확장함수이기도 하다.
    - 메서드가 적용되는 범위를 제한하기 위함
    - Table 이 없으면, Column 의 프로퍼티를 정의해도 아무 의미가 없다.
    
    - `autoincrement()`는 오직 `Table` 클래스 안에서만 사용 가능하고,
    - 그 안에서도 `Int` 타입의 컬럼(`Column<Int>`)에만 사용할 수 있다는 뜻.
        
        ```kotlin
        val id = integer("id").autoincrement()
        
        val name = varchar("name", 50).autoincrement() // 불가능
        
        ```
        

# 문제