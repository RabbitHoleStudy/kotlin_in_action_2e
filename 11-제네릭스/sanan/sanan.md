![image](https://github.com/user-attachments/assets/078f79f4-9cbf-480f-80c1-206a800401c2)

- 요약
    - 코틀린의 제네릭은 자바와 매우 비슷하며, 제네릭 함수와 클래스를 자바와 유사하게 선언할 수 있다.
    - 자바와 마찬가지로 **제네릭 타입의 타입 인자**는 **컴파일 시점에만 존재**한다.
        - 즉, **런타임에는 타입 정보가 지워지기 때문에** 제네릭 타입을 is 연산자로 검사할 수 없다.
    
    ```kotlin
    fun <T> check(value: Any): Boolean {
        return value is T // ❌ 오류 발생
    }
    ```
    
    - 단, **인라인 함수**에서 타입 파라미터에 `reified` 키워드를 붙이면,
        - **실행 시점**에도 타입을 검사할 수 있고, `java.lang.Class` 객체도 얻을 수 있다.
    
    ```kotlin
    inline fun <reified T> check(value: Any): Boolean {
        return value is T // ✅ 가능
    }
    ```
    
    - 변성(variance)은 **기저 클래스는 같지만 타입 인자가 다른 두 제네릭 타입 간**의 **상하위 관계를 설명**하기 위한 개념이다.
    
    ---
    
    ### **✅ 공변성(Producer, out)**
    
    - 제네릭 클래스의 타입 파라미터가 **out 위치(반환 전용)**에서만 사용된다면,
    - out 키워드를 붙여서 **공변성(covariance)**을 선언할 수 있다.
    
    ```kotlin
    interface Source<out T> {
        fun next(): T
    }
    ```
    
    - 예: List<String>은 List<Any>의 **하위 타입**이다. (List는 공변적임)
    
    ---
    
    ### **✅ 반공변성(Consumer, in)**
    
    - 타입 파라미터가 **in 위치(매개변수 전용)**에서만 사용되는 경우,
    - in 키워드를 붙여 **반공변성(contravariance)**을 선언할 수 있다.
    
    ```kotlin
    interface Consumer<in T> {
        fun consume(item: T)
    }
    ```
    
    ---
    
    ### **✅ 함수 타입의 변성**
    
    - 함수 타입 (A) -> B는
        - **입력 타입 A에 대해 반공변성**,
        - **출력 타입 B에 대해 공변성**을 가진다.
    - 예:
        
        (Animal) -> Int는 (Cat) -> Number의 하위 타입이다.
        
        → Cat은 Animal의 하위, Number는 Int의 상위
        
    
    ---
    
    ### **✅ 사용 지점 변성(Site variance)**
    
    - 제네릭 클래스 선언부에 변성을 지정하지 않고, **사용하는 위치에서 in 또는 out을 지정**할 수 있다.
    
    ```kotlin
    fun copy(from: Array<out Any>, to: Array<Any>) { ... }
    ```
    
    ---
    
    ### **✅ 스타 프로젝션(*)**
    
    - 타입 인자를 정확히 모르거나 중요하지 않을 경우 * 기호로 대체할 수 있다.
    
    ```kotlin
    val list: List<*> = listOf("A", 1, true)
    ```
    
    ---
    
    ### **✅ 타입 별명(Type alias)**
    
    - 복잡한 타입 이름에 **별칭을 부여**하여 더 간단하게 표현할 수 있다.
    
    ```kotlin
    typealias StringMap = Map<String, String>
    val headers: StringMap = mapOf("Content-Type" to "application/json")
    ```
    
    ---
    
- `in`, `out` 키워드와 위치 - 소비와 반환
- ![image](https://github.com/user-attachments/assets/bd051ba6-dfe7-470d-9270-c5a133001e48)

    
    ```kotlin
    // ─────────────────────────────────────────────────────────────────
    // 1. 공통 클래스 정의
    // ─────────────────────────────────────────────────────────────────
    
    /** 모든 과일의 최상위 타입 */
    open class Fruit
    
    /** Fruit의 하위 타입: 바나나 */
    open class Banana : Fruit()
    
    /** Banana의 하위 타입: 더 달콤한 바나나 */
    class SweetBanana : Banana()
    
    /** Fruit의 또 다른 하위 타입: 두리안 */
    class Durian : Fruit()
    
    // ─────────────────────────────────────────────────────────────────
    // 2. 제네릭 박스 클래스 정의
    // ─────────────────────────────────────────────────────────────────
    
    /**
     * 1) 불변성(Invariance)
     * - T 자리에 들어온 타입 그대로만 사용.
     * - Box<Banana>는 Box<Fruit>로 업/다운캐스팅 불가.
     */
    class Box<T>(val item: T)
    
    /**
     * 2) 공변성(Covariance)
     * - out T 키워드를 사용해 “읽기 전용”으로만 T를 사용함을 명시.
     * - 공급자(Producer) 역할: 내부에서 T를 꺼내서 반환만 함.
     * - CovariantBox<Banana>를 CovariantBox<Fruit>로 업캐스팅 가능.
     */
    class CovariantBox<out T>(val item: T)
    
    /**
     * 3) 반공변성(Contravariance)
     * - in T 키워드를 사용해 “쓰기 전용”으로만 T를 사용함을 명시.
     * - 소비자(Consumer) 역할: 내부에서 T를 매개변수로만 받음.
     * - ContravariantBox<Fruit>를 ContravariantBox<Banana>로 다운캐스팅 가능.
     */
    class ContravariantBox<in T> {
        fun put(item: T) { /* 아이템을 받기만 함 */ }
    }
    
    // ─────────────────────────────────────────────────────────────────
    // 3. 메인 함수에서 사용 예시
    // ─────────────────────────────────────────────────────────────────
    
    fun main() {
        // 1) 불변성: 같은 타입끼리만 대입 가능
        val fruitBox: Box<Fruit>   = Box(Fruit())   // OK: Fruit 박스에 Fruit
        val bananaBox: Box<Banana> = Box(Banana())  // OK: Banana 박스에 Banana
        //  val cantUpcast: Box<Fruit> = bananaBox
        //  val cantDownCast: Box<SweetBanana> = bananaBox
        // → 컴파일 오류: Box<Banana>는 Box<Fruit>에서 사용하는 타입 파라미터와 아예 다른 것으로 간주(불변성)되기 때문.
    
        // 2) 공변성: 공급자 역할(읽기 전용)
        val covBanana: CovariantBox<Banana> = CovariantBox(Banana())
        // Banana를 공급하는 박스이므로 Fruit를 기대하는 곳(Banana는 Fruit이므로)에 넣어도 안전
        val covFruit: CovariantBox<Fruit>   = covBanana
        val someFruit: Fruit               = covFruit.item
        // covFruit.item은 실제로 Banana를 반환하지만,
        // 반환 타입이 상위 타입(Fruit)이기 때문에 타입 안정성이 보장됨.
    
        // SweetBanana > Banana > Fruit이므로 당연히 가능
        val covSweetBananaAsFruit: CovariantBox<Fruit> = CovariantBox(SweetBanana())
    
        // val covSweetBananaWithBanana: CovariantBox<SweetBanana> = CovariantBox(Banana())
        // → 컴파일 오류: SweetBanana에 대한 공급을 기대하지만 Banana를 공급? - Banana가 SweetBanana말고 뭐가 있는 줄 알고?
    
        // 3) 반공변성: 소비자 역할(쓰기 전용)
        val contraFruit: ContravariantBox<Fruit>   = ContravariantBox<Fruit>()
        // Fruit를 받을 수 있는 박스를 Banana용으로 취급해도 안전!
        val contraBanana: ContravariantBox<Banana> = contraFruit
        contraBanana.put(Banana())     // OK: Banana는 Fruit로 처리 가능
        contraBanana.put(SweetBanana())// OK: SweetBanana 역시 Banana의 서브타입
    
        // contraBanana.put(Durian())
        // → 컴파일 오류: Durian은 Banana나 그 서브타입이 아니므로 넣을 수 없음
    
        // ※ contraFruit.put(Banana())도 당연히 OK,
        //    왜냐면 Fruit 박스에는 어떠한 Fruit도 안전하게 넣을 수 있으니까.
    }
    ```
    
- 생성자 파라미터에서 사용되는 제네릭은 공변/반공변성을 띄지 않는다
    - 왜 일까? - out 키워드에 따라 T 제네릭은 공변성을 띄게 된다. 즉 producer로서 - readonly로서 작동하게되는데, 이 경우 vararg의 경우에는 단순한 인자인 `animals: Array<out T>`로 변환되고, val의 경우에도 불변 값이므로 readonly로 작동한다. 따라서, `<out T : Animal>`이더라도 생성자에서 사용되는 인자가 Animal의 하위타입이기만 하면 사용 가능하다. 한편, var의 경우 setter, 즉 in 위치에 해당할 수 있기 떄문에 T를 out으로 사용할 수 없다.
    - 한편, 클래스 내부(private)로 선언해서 사용하면 var로도 선언할 수 있어지는데, 이는 **외부에서 소비하게 될 때의 문제에 영향이 없기 때문**이다 - 결국 외부에 반환하더라도 T로 반환하게 될 것이기 때문에.
- 무변성을 띄는 제네릭의 경우에 구체적인 타입 제네릭을 기대하는 것임과  동시에 그것이 어떤 것이어도 상관없다는 선언이므로, `MutableList<*>`는 `MutableList<Any?>`와 같지 않다. 후자가 전자에 대해서 할당될 수는 있지만, ‘같다’는 표현에서 구체적으로 타입이 지정되지 않은 전자와 타입이 지정된 후자는 다른 제네릭 타입으로 구분되기 때문이다.
    - 즉, `MutableList<*>`는 ‘뭐든지 읽을 수 있지만, 어떻게든 쓸 수는 없다’는 의미이고, `MutableList<Any?>`의 경우는 ‘뭐든지 읽고, 쓸 수 있다’의 의미이다.
    - `MutableList<*>`는 내부적으로는 `MutableList<out Any?>`로 읽기는 가능하지만, 쓰기는 아예 할 수 없도록(`in Nothing`) 막는다.
        
        ```kotlin
        val anythingList: MutableList<Any?> = mutableListOf("이것도 되고", 0, null, 1.0, "저것도 되고")
        val startProjectedList: MutableList<*> = anythingList
        val thisIsAny: Any? = null
        //startProjectedList.add(thisIsAny) << Receiver type 'MutableList<*>' contains star projection which prohibits the use of 'fun add(element: E): Boolean'.
        val thisIsNothing: Nothing = throw IllegalArgumentException("이것만 된다.")
        startProjectedList.add(thisIsNothing) // <- 가능
        ```
        
- 내가 사용한 코드
    - 제네릭 : `TurningViewModel`
    - `out` : 로컬 DB 스키마 - 마이그레이션의 제한
    - `typealias` : `Debouncer`의 `ExecutedMillis`

- 문제1
    
    다음 위의 코드를 보고`Fruit.filterBySpecies(): List<원하는 종>` 을 구현해보자.
    
    ```kotlin
    sealed interface Fruit {
        class Banana(val length: Int) : Fruit
        class Apple(val color: String) : Fruit
        class Orange(val size: Int) : Fruit
    }
    
    // 확장 함수 구현
    
    fun main() {
        val banana = Fruit.Banana(10)
        val apple = Fruit.Apple("red")
        val orange = Fruit.Orange(5)
    
        val fruits: List<Fruit> = listOf(banana, apple, orange)
        fruits.filterBySpecies<Fruit.Banana>().forEach {
            println("Found a banana with length: ${(it as Fruit.Banana).length}")
        }
    }
    ```
    
    - 답
        
        ```kotlin
        inline fun <reified T> List<Fruit>.filterBySpecies(): List<T> {
            return this.filter { it is T }.map { it as T  }
        }
        // 한편, 해당 함수는 타입 인자에 뭐든 받을 수 있으므로, Fruit에 대해서만 사용할 수 있도록 바꾼다면 
        // 상계(Upper Bound)를 이용할 수 있다.
        inline fun <reified T**: Fruit**> List<Fruit>.filterBySpecies(): List<T> {
            return this.filter { it is T }.map { it as T  }
        }
        
        // 한편, 해당 filter와 map은 Iterable의 API이므로 더 확장성을 가진 함수로 바꾼다면
        inline fun <reified T: Fruit> Iterable<Fruit>.filterBySpecies(): List<T> {
            return this.filter { it is T }.map { it as T  }
        }
        
        // 한편, Array와 Iterable은 다르므로, Array까지 지원해준다면 더 확장적일 것이다.
        inline fun <reified T: Fruit> Array<Fruit>.filterBySpeciesArray(): List<T> {
            return this.filter { it is T }.map { it as T }
        }
        ```
        
- 문제 2
    
    제네릭을 이용하여 `String`인 경우 그 끝에 `.`을 붙이고, `Number`인 경우 `*2` 를 하여 반환하고, 그 외의 타입인 경우 error를 throw하는 함수를 작성해보자.
    
    ```kotlin
    fun main() {
        2.stringAndIntegerOnly().also { println(it) }
        "Hello".stringAndIntegerOnly().also { println(it) }
        0.3f.stringAndIntegerOnly().also { println(it) } // This will throw an exception
    }
    ```
    
    - 답
        
        ```kotlin
        inline fun <reified T> T.stringAndIntegerOnly(): Any {
            return when (this) {
                is String -> "$this."
                is Int -> this * 2
                else -> throw IllegalArgumentException("Only String and Int types are allowed")
            }
        }
        
        // 왜 반환을 Any로 하게 되었을까? - 저 상태에서 반환을 T로 두면 컴파일 에러가 난다.
        inline fun <reified T> T.stringAndIntegerOnly(): T {
            return when (this) {
                is String -> "$this."
                is Int -> this * 2
                else -> throw IllegalArgumentException("Only String and Int types are allowed")
            }
            // Return type mismatch: expected 'T (of fun <T> T.stringAndIntegerOnly)', actual 'Comparable<*> & Serializable'.
            // 왜 Comparable<*> & Serializable을 기대하는 걸까?
            // 그 이유는, String, Int의 가장 가까운 공통 조상이기 때문이다.
        		// when 식의 타입이 바로 이 intersection 타입으로 추론된다.
        }
        
        // 한편, as T를 이용해 직접 캐스팅하여 return하면 컴파일 에러가 사라진다.
        inline fun <reified T> T.stringAndIntegerOnly(): T {
            return when (this) {
                is String -> "$this."
                is Int -> this * 2
                else -> throw IllegalArgumentException("Only String and Int types are allowed")
            } as T
        }
        
        // reified를 없애도 compile error는 없으나, unchecked cast에 대한 warning이 나타난다
        // 당연하다. 컴파일 타임에 해당 형변환이 가능한지 알 수 없기 때문이다(reified)
        fun <T> T.stringAndIntegerOnly(): T {
            return when (this) {
                is String -> "$this."
                is Int -> this * 2
                else -> throw IllegalArgumentException("Only String and Int types are allowed")
            } as T // Unchecked cast of 'Comparable<*> & Serializable' to 'T (of fun <T> T.stringAndIntegerOnly)'.
        }
        
        // else -> throw, else -> this ... as T로 리턴이 가능한 이유는 무엇일까?
        inline fun <reified T> T.stringAndIntegerOnly(): T {
            return when (this) {
                is String -> "$this."
                is Int -> this * 2
                else -> throw IllegalArgumentException("Only String and Int types are allowed")
                // 혹은
        				// else -> this
            } as T
        }
        
        // 그 이유는, 구체화된 타입 T에 대해서 throw Exception의 Nothing은 해당 타입 T의 최하위 타입이므로 T로 캐스팅이 가능하다(조상이므로)
        // 한편, this는 당연히 컴파일 타임에 전달(캡처)된 타입인자 그대로이므로 문제 없이 형변환이 가능하다.
        
        // 근데 아래 코드는 왜 컴파일될까?
        inline fun <reified T> T.stringAndIntegerOnly(): T {
            return when (this) {
                is String -> "$this."
                is Int -> this * 2
                else -> 3
                // 혹은
                // else -> this
            } as T
        }
        
        fun main() {
            2.stringAndIntegerOnly().also { println(it) }
            "Hello".stringAndIntegerOnly().also { println(it) }
            0.3f.stringAndIntegerOnly<Float>().also { println(it) } // This will throw an exception
        }
        // 그 이유는, inline을 통해서 as T로 cast의 실행문이 같이 적혀져, 런타임에서 Unchecked Cast에 대해서만 확인하면 되기 때문이다.
        // 그러므로 컴파일 타임에서 문제가 없는 것 처럼 작동하게 된다.
        // 코틀린 컴파일러는 제네릭 함수 내부에서 as T 캐스팅이 있을 때, 실제 타입이 무엇이든 컴파일 자체는 허용한다.
        // 즉, 컴파일 타임에는 T가 무엇인지 모르기 때문에, as T가 항상 성공할지 실패할지 판단하지 않는다.
        // 따라서, 만약 런타임에 타입이 맞지 않으면 ClassCastException이 발생한다.
        
        // 위 문제를 해결하여 안전하게 만든다면 다음과 같이 시도해볼 수 있겠다.
        inline fun <reified T> T.stringAndIntegerOnly(): T {
            return when (this) {
                is String -> "$this."
                is Int -> this * 2
                else -> 3
                // 혹은
                // else -> this
            } as? T ?: this
        }
        
        fun main() {
            2.stringAndIntegerOnly().also { println(it) }
            "Hello".stringAndIntegerOnly().also { println(it) }
            0.3f.stringAndIntegerOnly<Float>().also { println(it) }
        }
        ```
