- 요약
    - 내부 DSL은 여러 메서드 호출로 구성된 구조를 더 쉽게 표현할 수 있게 해주는 API 설계 패턴이다.
    - 수신 객체 지정 람다는 람다 본문 안에서 메서드 호출 방식을 재정의함으로써, 여러 요소를 중첩해서 표현할 수 있는 구조를 만들 수 있다.
    - 수신 객체 지정 람다가 함수의 파라미터로 전달될 경우, 해당 람다의 타입은 확장 함수 타입이며, 함수는 람다를 호출할 때 수신 객체를 함께 제공한다.
    - 외부 템플릿이나 마크업 언어 대신 코틀린 내부 DSL을 사용하면 코드의 추상화와 재사용성을 높일 수 있다.
    - 원시 타입에 대한 확장을 정의하면, 기간 등의 상수를 더욱 읽기 쉽게 표현할 수 있다.
    - invoke 관례를 사용하면 임의의 객체를 마치 함수처럼 호출할 수 있어 DSL 표현력이 향상된다.
    - kotlinx.html, kotest, Exposed 라이브러리 내부 DSL을 제공한다.
- DSL(Domain Specific Language)는 이름 그대로 도메인 한정 언어이다. 즉, 한정적인(일부러 닫아놓은) 범위 내에서 자유롭게, 가독성과 생산성 좋게 사용할 수 있게끔 구성하는 언어이다.
- 선언적으로 표현할 수 있다는 점에서, 큰 메리트를 가진다. (특히 보일러 플레이트를 많이 줄일 수 있다)
- 수신 객체 지정 람다를 이용하면 DSL을 구성하기 편리해진다 - `apply`, `with`가 사용하는 방식이다.
- `@DslMarker`는 해당 람다 스코프의 수신 객체 참조 범위를 딱 1개(현재의 this)로 한정해준다.
- `operator fun invoke()`의 구현을 통해 `해당_객체()` 처럼 사용할 수 있다. (주로 유즈케이스)

- 문제 1
    
    다음 중 **수신 객체 지정 람다**에 대한 설명으로 올바른 것은?
    
    1. 람다 내부에서 this는 항상 최상위 클래스의 인스턴스를 가리킨다.
    2. 수신 객체 지정 람다는 일반 함수 타입과 동일하게 선언된다.
    3. 수신 객체 지정 람다의 타입은 T.() -> R 형식의 **확장 함수 타입**이다.
    4. 수신 객체 지정 람다는 반드시 클래스 내부에서만 사용할 수 있다.
    5. 수신 객체 지정 람다는 invoke() 연산자와 함께 쓸 수 없다.
    - 답
        
        
        | 1번 | this는 **람다의 수신 객체**를 가리키며, 반드시 최상위 클래스는 아님. 클래스 외부에서도 사용 가능 | ❌ |
        | --- | --- | --- |
        | 2번 | 일반 함수 타입은 (T) -> R 형식이며, 수신 객체 지정 람다는 T.() -> R로 다름 | ❌ |
        | 3번 | 맞는 설명. 수신 객체 지정 람다는 수신 객체의 컨텍스트 내에서 코드 블록을 작성할 수 있게 함 | ✅ |
        | 4번 | 클래스 외부 함수에서도 충분히 사용 가능 (with, apply, 사용자 정의 DSL 등) | ❌ |
        | 5번 | invoke()는 수신 객체 람다와는 무관하게 오버로딩 가능. 둘은 동시에 사용 가능 | ❌ |
- 문제 2
    
    다음 Person 객체의 민감정보(name, age, gender)에 대한 DSL을 구성해서, 민감정보만 print하는 DSL을 만들어보자.
    
    ```kotlin
    data class Person(val name: String, val age: Int, val gender: String, val isSleeping: Boolean)
    ```
    
    - 답
        
        ```kotlin
        class SensitivePrinter(private val person: Person) {
            fun name() = println("name: ${person.name}")
            fun age() = println("age: ${person.age}")
            fun gender() = println("gender: ${person.gender}")
        }
        
        fun printSensitive(person: Person, block: SensitivePrinter.() -> Unit) {
            SensitivePrinter(person).block()
        }
        
        fun Person.printSensitiveExtended(block: SensitivePrinter.() -> Unit) {
            SensitivePrinter(this).block()
        }
        
        fun main() {
              val alice = Person(
                  name = "Alice",
                  age = 29,
                  gender = "Female",
                  isSleeping = true
              )
        
              printSensitive(alice) {
                  name()
                  gender()
              }
        
              alice.printSensitiveExtended {
                  name()
                  gender()
                  age()
              }
        }
        ```
        
- 문제 3
    
    `operator fun invoke` 관례를 통해, 책을 생성하는 `UseCase`를 설계해보자.
    
    ```kotlin
    fun main() {
        val createUseCase = BookCreateUseCase()
        createUseCase("이것은 책의 이름이다").also {
            println(it)
        }
        // Book(name=이것은 책의 이름이다)
    }
    ```
    
    - 답
        
        ```kotlin
        interface UseCase<in PARAMS, out DATA> where DATA : Any {
            fun run(params: PARAMS): DATA
        
            operator fun invoke(params: PARAMS): DATA {
                return run(params)
            }
        }
        
        data class Book(
            val name: String,
        )
        
        class BookCreateUseCase : UseCase<String, Book> {
            override fun run(params: String): Book {
                return Book(params)
            }
        }
        
        fun main() {
            val createUseCase = BookCreateUseCase()
            createUseCase("이것은 책의 이름이다").also {
                println(it)
            }
        }
        ```
