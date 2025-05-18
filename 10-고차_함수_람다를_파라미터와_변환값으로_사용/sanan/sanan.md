- 요약
    - **함수 타입**을 사용하면, 함수 자체를 참조하는 **변수, 파라미터, 반환값**을 만들 수 있다.
        - 예: `val f: (Int) -> String` = 정수를 받아 문자열을 반환하는 함수
    - **고차 함수**(Higher-Order Function)란 **다른 함수를 인자로 받거나, 함수를 반환하는 함수**를 말한다.
        - 함수의 파라미터 타입이나 반환 타입으로 함수 타입을 지정하면 고차 함수를 만들 수 있다.
    - **인라인 함수**로 선언된 함수는 컴파일 시에 **함수 본문과 전달된 람다를** 호출 지점에 **직접 코드로 삽입**한다.
        - 결과적으로, **추가적인 람다 객체 생성 없이** 일반 함수처럼 실행된다.
        - 즉, 람다를 사용하는 비용이 사라지며, **성능 최적화**가 가능하다.
    - 고차 함수를 활용하면,
        - **컴포넌트의 코드를 더 잘 재사용**할 수 있으며,
        - **범용적인 제네릭 라이브러리**를 설계할 수 있다.
    - **인라인 함수 내부의 람다**에서는 `non-local return`이 가능하다.
        - 즉, 람다 내부에서 return하면 바깥쪽 인라인 함수 자체를 종료할 수 있다.
    - **익명 함수**는 람다식을 대신하여 사용할 수 있다.
        - `fun(x: Int): Boolean { return x > 0 }` 형태
        - **여러 개의 반환 지점**이 필요한 경우, **람다식보다 익명 함수**가 더 명확하고 적합하다.
        - 익명 함수에서는 return이 **자기 자신만 종료**한다. (non-local return 없음)
- 람다를 제공해줄 수 있다는 것은 상위 컨텍스트가 원하는 행동 양식을 자식이 모르는 상태에서 단순히 인자를 제공하여 호출할 수 있다는 부분.
- 시퀀스는 lambda inlining이 일어나지 않으므로 시퀀스를 이용하더라도 원소 수가 작은 경우에는 일반 collection API를 사용하는 것이 성능에 유효할 수 있다.
- inline을 쓰더라도 람다를 인자로 받는 함수의 경우가 유효할 가능성이 높다. - 난사는 생각해보고 해라.
- 인라이닝 된 람다의 경우 non-local 리턴을 할 수 있고, 이 경우 @label을 지정하여 조절할 수 있다.

- 문제 1
    
    해당 예상 출력에 알맞게 processNumbers를 작성해보자.
    
    ```kotlin
    fun main() {
        val list = listOf(1, 2, 3, 4, 5)
        processNumbers(list) { it * 2 }
        // 예상 출력
        // 2
        // 4
        // 6
        // 8
        // 10
    }
    ```
    
    답:
    
    ```kotlin
    fun processNumbers(numbers: List<Int>, operation: (Int) -> Int) {
        numbers.forEach {
            operation(it).also(::println)
        }
    }
    ```
    
- 문제 2
    
    다음 코드의 출력을 예상해보자.
    
    ```kotlin
    inline fun runBlockingAction(action: () -> Unit) {
        println("Before action")
        action()
        println("After action")
    }
    
    fun main() {
        runBlockingAction {
            println("Inside lambda")
            return
        }
        println("Main ended")
    }
    ```
    
    답:
    
    ```kotlin
    // Before action
    // Inside lambda
    ```
    
- 문제 3
    
    다음 코드에서 발생할 문제를 찾고, 해결법을 제시해보자.
    
    ```kotlin
    fun nonInlineCaller(action: () -> Unit) {
        println("Calling action")
        action()
    }
    
    fun main() {
        nonInlineCaller {
            println("Start")
            return
        }
    }
    ```
    
    - 답:
        
        ```kotlin
        return에 @nonInlineCaller label을 붙여준다.
        ```
