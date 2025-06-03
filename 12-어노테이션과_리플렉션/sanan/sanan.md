- 요약
    
    ## **📌 어노테이션(annotation)**
    
    - 어노테이션을 적용할 때는 @Annotation(parameters) 구문을 사용한다.
    - 코틀린에서는 **파일, 식(expression)** 등 넓은 범위의 **타깃**에 어노테이션을 붙일 수 있다.
    - 어노테이션의 인자로는 다음을 사용할 수 있다:
        - 기본 타입 값 (Int, Boolean 등)
        - 문자열
        - enum 상수
        - 클래스 참조 (::class)
        - 다른 어노테이션 인스턴스
        - 배열
    
    ---
    
    ### **🎯 어노테이션 사용 위치 지정:**
    
    ### **@target**
    
    - 하나의 코틀린 선언이 여러 JVM 바이트코드 요소를 만들어낼 수 있기 때문에,
        
        @target을 사용하여 정확히 어떤 **타깃 요소(field, method 등)**에 어노테이션을 붙일지 지정할 수 있다.
        
    
    ```
    @get:JvmName("getCustomName")
    val name: String = "Kotlin"
    ```
    
    ---
    
    ### **✨ 어노테이션 클래스 정의**
    
    - annotation class 키워드를 사용해 정의한다.
    - 모든 파라미터는 **val 프로퍼티로 주 생성자에 선언**해야 하며, **본문은 없어야 한다.**
    
    ```
    annotation class JsonName(val name: String)
    ```
    
    ---
    
    ### **🔧 메타 어노테이션**
    
    - 어노테이션에 붙는 어노테이션.
    - 예:
        - @Target → 적용 가능한 위치 지정 (예: 클래스, 함수, 프로퍼티 등)
        - @Retention → 어노테이션의 유지 범위 (SOURCE, BINARY, RUNTIME)
        - @Repeatable → 중복 적용 허용 여부
    
    ---
    
    ## **🔍 리플렉션(Reflection)**
    
    - **리플렉션 API**를 사용하면 실행 시점에 **클래스, 함수, 프로퍼티** 등을 열거하거나 호출할 수 있다.
    - 주요 타입:
        - KClass: 클래스 표현
        - KFunction: 함수 표현
        - KProperty: 프로퍼티 표현
        - 이들은 모두 KCallable을 확장한다 → call, callBy 메서드 제공
    
    ---
    
    ### **🔑 클래스 참조와 함수/프로퍼티 참조**
    
    - 클래스의 KClass 인스턴스:
        
        MyClass::class
        
    - 객체 인스턴스의 클래스 참조:
        
        myObject::class
        
    
    ---
    
    ### **⚙️ 함수 호출**
    
    - KFunction0, KFunction1 등은 파라미터 수에 따라 함수 타입을 표현한다.
    - invoke()로 호출하거나,
        
        call() / callBy()로 파라미터를 넣어 실행 가능하다.
        
    - callBy()는 **디폴트 파라미터**를 자동으로 사용해 호출 가능.
    
    ---
    
    ### **🏷️ 프로퍼티 접근**
    
    - KProperty0, KProperty1 등은 프로퍼티를 표현한다.
    - .get()으로 값 읽기, .set()으로 값 쓰기 (KMutableProperty 계열에서 가능)
    
    ---
    
    ### **📐 타입 정보 가져오기**
    
    - typeOf<T>() 함수를 사용하면 실행 시점의 KType을 얻을 수 있다.
        - 단, 사용하려면 함수에 reified가 붙은 인라인 함수여야 한다.
- 어노테이션은 컴파일 타임에 컴파일러와 프레임워크(라이브러리) 등에 특정 뉘앙스나, 의미, 또는 제약을 부여한다.
- 리플렉션은 런타임에 클래스 - 인스턴스에 대한 제어 수단으로 사용할 수 있다.
- 문제 1
    
    아래 두 클래스의 함수 각각을 reflection을 이용하여 호출해보자.
    
    `Person`의 경우, 서로 다른 프로퍼티 name을 갖는 두 인스턴스를 이용해 호출해보자.
    
    ```kotlin
    class HelloMachine {
        fun sayHello(name: String): String = "Hello, $name!"
    }
    
    class Person(val name: String) {
        fun introduce() = println("Hello, I'm $name!")
    }
    ```
    
    ---
    
    - 답
        
        ```kotlin
        fun main() {
            val instance = HelloMachine()
            val clazz = instance::class
            val hello = clazz.functions.find { it.name == "sayHello" }?.also { println("$it <- exists.") }
            println(hello?.call(instance, "쑤암제"))
        
            val ljm = Person("이재명")
            val kms = Person("김문수")
            val personClass = ljm::class
            personClass.functions.find { it.name == "introduce" }?.let {
                it.call(ljm)
                it.call(kms)
            }
        }
        ```
        
- 문제 2
    
    다음 중 코틀린의 어노테이션(annotation) 에 대한 설명으로 틀린 것은?
    
    **1.**  어노테이션의 유지 범위(`Retention`)는 `SOURCE`, `BINARY`, `RUNTIME` 중 선택할 수 있다.
    
    **2.** 어노테이션은 선언에 의미를 부여하고, 컴파일러나 프레임워크가 이를 해석하여 기능을 확장할 수 있다.
    
    **3.**`@Retention(BINARY)`로 선언된 어노테이션은 바이트코드에 포함되지만, 실행 시점에 리플렉션으로 접근할 수 있다.
    
    **4.** 리플렉션으로 어노테이션 정보를 조회하려면 `Retention`이 `RUNTIME`이어야 한다.
    
    **5.** 어노테이션은 클래스, 함수, 프로퍼티뿐 아니라 식(expression), 파일 수준에도 적용할 수 있다.
    
    ---
    
    - 답
        
        ### **1. ✅**
        
        - 코틀린 어노테이션의 `Retention`은 **`SOURCE`, `BINARY`, `RUNTIME`** 세 가지 중 선택 가능함
        
        ---
        
        ### **2. ✅**
        
        - 어노테이션은 컴파일러 플러그인, kapt, 프레임워크(DI, 직렬화 등)에서 의미를 해석하는 **메타 정보**로 널리 사용됨
        
        ---
        
        ### **3. ❌**
        
        - 어노테이션이 **항상 런타임에 존재하는 것은 아님**
        - `@Retention(SOURCE)`로 선언된 어노테이션은 **컴파일 타임에만 존재**하고, **바이트코드에조차 포함되지 않음**
        - `@Retention(BINARY)`는 바이트코드에는 남지만, **리플렉션으로는 접근 불가**
        - 오직 `@Retention(RUNTIME)`일 때만 **리플렉션으로 접근 가능**
        
        ---
        
        ### **4. ✅**
        
        - `@Retention(RUNTIME)`으로 선언된 어노테이션만 `clazz.annotations` 같은 **리플렉션 API로 조회**할 수 있음
        
        ---
        
        ### **5. ✅**
        
        - 코틀린은 `@file:`, `@get:`, `@set:`, `@receiver:` 등 **특정 타깃 지정 어노테이션 문법**을 제공하여 파일, 식, 생성자 등 넓은 범위에 어노테이션을 적용할 수 있음
        
- 문제 3
    
    **다음 중 코틀린 리플렉션 API에 대한 설명으로 틀린 것은?**
    
    **A.** `KClass`는 클래스의 타입 메타정보를 담는 타입이며, `MyClass::class` 문법으로 얻을 수 있다.
    
    **B.** `KFunction.call()`을 통해 private 함수도 호출할 수 있다.
    
    **C.** `KProperty`는 프로퍼티를 표현하며, `.get()`으로 값을 가져올 수 있다.
    
    **D.** `KFunction.callBy()`를 사용하면 디폴트 파라미터가 있는 함수를 일부 인자만으로 호출할 수 있다.
    
    **E.** `typeOf<T>()`를 사용하면 실행 시점의 `KType` 정보를 얻을 수 있다.
    
    - 답
        
        ### **1. ✅**
        
        - KClass는 클래스의 메타 정보를 표현하는 타입이며, ::class 키워드를 통해 얻는다.
            
            예: MyClass::class
            
        - 이는 java.lang.Class가 아니라 코틀린 전용 타입이며, .simpleName, .members 등을 통해 다양한 정보를 조회할 수 있다.
        
        ---
        
        ### **2. ❌**
        
        - **`KFunction.call()`은 private 함수에는 접근할 수 없다.**
        - 컴파일러는 접근 제한을 지키며, **`IllegalCallableAccessException` 예외**가 발생한다.
        - `Java Reflection`의 `setAccessible(true)`를 쓰지 않는 이상 private 멤버는 호출할 수 없다.
        
        ---
        
        ### **3. ✅**
        
        - `KProperty`는 코틀린의 프로퍼티를 표현하며, `.get()` 메서드로 값을 조회할 수 있다.
        - 예: `val nameProp = MyClass::name; nameProp.get(instance)`
        
        ---
        
        ### **4. ✅**
        
        - `KFunction.callBy()`는 파라미터에 **디폴트 값이 정의되어 있을 때**, 전달하지 않은 인자는 자동으로 디폴트 값이 사용된다.
        - **`Map<KParameter, Any?>`** 형태로 인자를 넘기며, 부분적 호출이 가능하다.
        
        ---
        
        ### **5. ✅**
        
        - `typeOf<T>()`는 `inline + reified` 키워드와 함께 사용되며, **실행 시점의 타입 표현`(KType)`** 을 반환한다.
        - 이 기능을 통해 복잡한 제네릭 타입도 런타임에 타입 정보로 추적 가능하다.
