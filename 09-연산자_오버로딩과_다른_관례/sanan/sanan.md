- 요약
    - 코틀린은 **정해진 이름의 함수를 정의함으로써** 표준적인 **수학 연산을 오버로딩**할 수 있도록 한다.
        
        → 자신만의 연산자를 정의할 수는 없지만, **중위 함수(infix function)**를 통해 더 표현력 있게 작성할 수 있다.
        
    - 비교 연산자 (==, !=, >, < 등)는 **모든 객체에 사용 가능**하며,
        
        → 내부적으로 equals() 또는 compareTo() 호출로 변환된다.
        
    - get, set, contains 함수를 정의하면,
        
        → 클래스 인스턴스에 대해 [ ], in 연산자를 **컬렉션처럼** 사용할 수 있다.
        
    - **정해진 관례**를 따르면, 범위를 만들거나 컬렉션/배열의 원소를 **이터레이션(반복)** 할 수 있다.
    - *구조 분해 선언(destructuring declaration)**을 통해
        
        → 한 객체의 상태를 **여러 변수로 분해해서 대입**할 수 있다.
        
        → 함수가 **여러 값을 반환**할 때 유용하며,
        
        → data class에는 자동으로 구조 분해가 제공된다.
        
        → 일반 클래스의 경우 componentN 함수를 정의하면 구조 분해를 지원할 수 있다.
        
    - *위임 프로퍼티(delegated property)**를 사용하면
        
        → 프로퍼티 값을 저장, 초기화, 읽기, 변경할 때의 로직을 **재활용**할 수 있다.
        
        → **프레임워크나 DSL 설계**에서 매우 강력하게 활용된다.
        
    - 표준 라이브러리의 lazy 함수를 이용해
        
        → **지연 초기화(lazy initialization)** 프로퍼티를 쉽게 구현할 수 있다.
        
    - Delegates.observable 함수를 사용하면
        
        → 프로퍼티 변경을 감지하고 **옵저버 패턴**처럼 처리할 수 있다.
        
    - **맵을 위임 객체로 사용하는 위임 프로퍼티**를 정의하면
        
        → 동적으로 다양한 속성을 제공하는 객체를 **유연하게 구성**할 수 있다.
        
- 이쁘게 짜고 느려지면 개선해라
- Pair, Triple은 간편하긴 하지만 의미를 담을 수 없으므로 아쉬움이 있다.
- 디스트럭처링 할당 시에 프로퍼티의 순서를 유의하자 (named로 사용할 수 없기 때문 - K2 컴파일러 이후에 named destructuring 논의 중)
- val에 대한 lazy 선언을 이용하면 캐싱 전략처럼 사용할 수 있음.
- `Delegates.observable(INITIAL) {property, old, new → }` 은 `setValue`, `getValue`에 대한 override를 통해 afterChange 콜백 실행으로 옵저빙을 제공한다.
    - ReadWriteProperty, ReadOnlyProperty 등을
- 근데 위임을 이용해서 set, get등을 사용하는데 DB를 이용해서 get-set을 query로 대체하는 건 오히려 추상화를 위해 명확성을 떨어뜨리는 부분처럼 느껴짐.
- BackginMap 등을 이용해서 map으로 미리 config을 조정한다든지 하는 방법은 알겠는데, 굳이 싶긴하다. `[]` 참조자를 굳이 안 써도 가시적으로 표현가능하지 않나.. 그냥 default value를 지정해주는게 더 명료한 느낌(개인적)

- 문제 1
    
    `config` 프로퍼티를 프로그램 내에서 단 한 번만 초기화시키고, 읽을 때마다 값이 계산되지 않고 **캐시**되도록 작성해보자.
    
    ```kotlin
    class AppConfigLoader {
        fun load(): String {
            println("Loading config...")
            return "config.yaml"
        }
    }
    
    class App {
        val loader = AppConfigLoader()
    
        // TODO: config 프로퍼티를 lazy로 선언
        val config by ...
    }
    
    fun main() {
        val app = App()
        println(app.config)
        println(app.config)
    }
    
    // 예상 출력
    // Loading config...
    // config.yaml
    // config.yaml
    ```
    
    - 답
        
        ```kotlin
        val config by lazy { loader.load() }
        ```
        

- 문제 2
    
    중위 연산 함수를 구현해서 다음 프로그램이 동작하게 해보자.
    
    ```kotlin
    fun main() {
        val presiCandi1 = ("이제는" space "이재명").dot()
        val presiCandi2 = ("기호 2번" space "김문수").dot()
        println(presiCandi1) // 이제는 이재명.
        println(presiCandi2) // 기호 2번 김문수.
    }
    ```
    
    - 답
        
        ```kotlin
        infix fun String.space(other: String): String = "$this $other"
        fun String.dot(): String = "$this."
        ```
        

- 문제 3
    
    `ReadWriteProperty<RECEIVER, PROPERTY_TYPE>`을 이용해서 `LogginProperty<T>` 를 구현해보자.
    
    ```kotlin
    import kotlin.properties.ReadWriteProperty
    import kotlin.reflect.KProperty
    
    class LoggingProperty<T>(private var value: T) : ReadWriteProperty<Any?, T> {
        override fun getValue(thisRef: Any?, property: KProperty<*>): T {
    			// todo
        }
    
        override fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        // todo
        }
    }
    
    class Person {
        var name: String by LoggingProperty("Unknown")
        var age: Int by LoggingProperty(0)
    }
    
    fun main() {
        val person = Person()
        person.name = "Alice"
        person.age = 25
        person.name
        person.age
        
        // name = Alice (was Unknown)
        // age = 25 (was 0)
        // name = Alice
        // age = 25
    }
    ```
    
    - 답
        
        ```kotlin
        class LoggingProperty<T>(private var value: T) : ReadWriteProperty<Any?, T> {
            override fun getValue(thisRef: Any?, property: KProperty<*>): T {
                println("${property.name} = $value")
                return value
            }
        
            override fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
                println("${property.name} = $newValue (was $value)")
                value = newValue
            }
        }
        ```
