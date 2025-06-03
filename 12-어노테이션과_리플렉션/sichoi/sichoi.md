# 어노테이션 선언과 적용

어노테이션은 선언에 메타데이터를 추가할 수 있도록 하며, 이 메타데이터는 컴파일 타임 또는 런타임 도구에서 접근할 수 있다.

## 어노테이션을 적용해 선언에 표지 남기기

### 어노테이션 기본 사용법

- 어노테이션은 `@어노테이션이름` 형식으로 클래스나 함수 선언 앞에 붙인다.
- 예: `@Test`, `@Deprecated`

### @Deprecated 어노테이션 예시

```kotlin
@Deprecated("Use removeAt(index) instead.", ReplaceWith("removeAt(index)"))
fun remove(index: Int) { /* ... */ }
```

- message: 경고 메시지
- replaceWith: 대체 호출 예시 제공
- level: WARN, ERROR, HIDDEN (경고/컴파일 금지/이진 호환 유지용)

### 어노테이션 인자로 가능한 값

- 기본 타입, 문자열, enum, 클래스 참조 (`::class`), 다른 어노테이션, 배열
- 클래스 인자: `@Annotation(MyClass::class)`
- 어노테이션 인자: `@Deprecated(replaceWith = ReplaceWith("..."))`
- 배열 인자: `@RequestMapping(path = ["/a", "/b"])`

### const 상수만 어노테이션 인자로 사용 가능

```kotlin
const val TEST_TIMEOUT = 10L

@Timeout(TEST_TIMEOUT)  // 가능
@Timeout(timeoutValue)  // 컴파일 오류
```

## 어노테이션이 참조할 수 있는 정확한 선언 지정: 어노테이션 타깃

### 사용 지점 타깃 지정

- `@get:JvmName("getFunc")`, `@set:JvmName("setFunc")` 등 사용 위치 지정 가능

### 지원하는 타깃 목록

- property, field, get, set, param, setparam, delegate, receiver, file 등
    - 프로퍼티에 적용할거냐, 필드에 적용할거냐, getter/setter 에 적용할거냐 등등
- 예: `@file:JvmName("MyUtils")`는 파일 전체에 적용
    - 파일에 적용할거냐

### Suppress 사용 예시

```kotlin
@Suppress("UNCHECKED_CAST")
val strings = list as List<String>
```

## 12.1.3 어노테이션을 활용해 JSON 직렬화 제어

### 예제: Person 직렬화

```kotlin
data class Person(val name: String, val age: Int)

val person = Person("Alice", 29)
serialize(person)  // {"name":"Alice", "age":29}
```

### 어노테이션으로 직렬화 제어

```kotlin
data class Person(
  @JsonName("alias") val firstName: String,
  @JsonExclude val age: Int? = null
)
```

- `@JsonName`: JSON 키 이름 변경
- `@JsonExclude`: 직렬화/역직렬화 제외

## 12.1.4 어노테이션 선언

### 어노테이션 클래스 정의

```kotlin
annotation class JsonExclude

annotation class JsonName(val name: String)
```

### 자바와의 비교

```java
public @interface JsonName {
  String value();
}
```

- 코틀린에서는 이름 있는 인자 또는 위치 인자로 사용 가능

## 메타어노테이션: 어노테이션을 처리하는 방법 제어

- `메타어노테이션`: 어노테이션 클래스에 적용할 수 있는 어노테이션

### @Target

```kotlin
@Target(AnnotationTarget.PROPERTY)
annotation class JsonExclude
```

- 사용 가능한 위치를 제한함 (Target 을 지정하지 않으면 모든 선언에 적용가능하게됨.)
    - 프로퍼티에서만 사용가능함.
- `AnnotationTarget` 이라는 enum 으로 어노테이션이 붙을 수 있는 대상이 정의됨.

### @Retention

- `Retention`: 어노테이션 클래스를 소스 수전에서 유지할지, .class 에 저장할지, 런타임에 리플렉션을 통해 접근가능하게 할지를 지정하는 어노테이션
- 코틀린은 기본적으로 RUNTIME
- 자바처럼 SOURCE, BINARY, RUNTIME 중 선택 가능

### 자바 호환용 FIELD 추가

```kotlin
@Target(AnnotationTarget.PROPERTY, AnnotationTarget.FIELD)
```

## 어노테이션 파라미터로 클래스 사용

### 예시: 인터페이스를 구현한 클래스를 명시

```kotlin
annotation class DeserializeInterface(val targetClass: KClass<out Any>)

data class Person(
  val name: String,
  @DeserializeInterface(CompanyImpl::class)
  val company: Company
)
```

### 타입 제약

- `KClass<out Any>`로 선언하여 하위 타입 수용
    - KClass 타입의 파라미터를 사용할 때, out 변경자 없이 `KClass<Any>` 를 쓰면, `Any::class` 만 인자를 넘길 수 있음.

## 어노테이션 파라미터로 제네릭 클래스 받기

### 예시: ValueSerializer 사용

```kotlin
interface ValueSerializer<T> {
  fun toJsonValue(value: T): Any?
  fun fromJsonValue(jsonValue: Any?): T
}

annotation class CustomSerializer(
  val serializerClass: KClass<out ValueSerializer<*>>
)

data class Person(
  val name: String,
  @CustomSerializer(DateSerializer::class) val birthDate: Date
)
```

### KClass<out ValueSerializer<*>>의 의미

- * 은 모든 타입 인자를 포괄하는 스타 프로젝션
    - 제네릭 타입을 구체적으로 알 수 없을 때 사용하는 와일드카드 같은 키워드
- `ValueSerializer<Date>`를 구현한 클래스한 모든 클래스만을 허용

<aside>
💡

- 특정 타입을 JSON으로 어떻게 바꿀지 제어하고 싶을 때 어노테이션으로 직렬화 클래스를 지정할 수 있음
- `KClass<out ValueSerializer<*>>`는 그런 클래스만 받도록 타입을 제한하는 안전장치
- 제네릭이 들어간 클래스는  (스타 프로젝션)으로 받아야 구체적인 타입을 몰라도 허용됨
</aside>

# 리플렉션: 실행 시점에 코틀린 객체 내부 관찰

## 12.2.1 코틀린 리플렉션 API: KClass, KCallable, KFunction, KProperty

### KClass

- `MyClass::class` 형태로 KClass 인스턴스를 얻을 수 있다.
- 클래스 이름, 프로퍼티, 함수, 생성자 등에 접근할 수 있다.

```kotlin
val kClass = person::class
kClass.simpleName // 클래스 이름
kClass.memberProperties // 모든 멤버 프로퍼티 접근
```

## KCallable

- 함수나 프로퍼티를 호출할 수 있는 공통 인터페이스
- `call(vararg args: Any?): R` 메서드를 통해 실행 가능

```kotlin
val kFunction = ::foo
kFunction.call(42) // 함수 foo(x: Int) 실행
```

## KFunctionN

- `KFunction2<Int, Int, Int>` 등 함수 파라미터 개수에 따라 인터페이스가 분화됨
- `invoke(a, b)` 형태로 타입 안정적으로 호출 가능

## KProperty와 KMutableProperty

- 프로퍼티를 표현하는 리플렉션 타입
- `get(obj)` 메서드로 값 조회
- `setter.call(value)`로 값 설정 (가변일 경우)

```kotlin
val kProperty = ::counter
kProperty.setter.call(21)
println(kProperty.get()) // 21
```

## 12.2.2 리플렉션을 사용한 JSON 직렬화 구현

- `serialize(obj: Any): String` 함수가 진입점
- 내부에서는 `StringBuilder`를 확장한 `serializeObject(obj: Any)` 함수를 사용
- KClass, memberProperties, get 메서드를 활용해 객체의 각 프로퍼티를 JSON key-value 쌍으로 직렬화

```kotlin
private fun StringBuilder.serializeObject(obj: Any) {
    val kClass = obj::class as KClass<Any>
    val properties = kClass.memberProperties
    properties.joinTo(this, prefix = "{", postfix = "}") { prop ->
        serializeString(prop.name)
        append(": ")
        serializePropertyValue(prop.get(obj))
    }
}
```

## 어노테이션을 활용한 직렬화 제어

### @JsonExclude

- 어노테이션이 붙은 프로퍼티는 직렬화 대상에서 제외

```kotlin
.filter { it.findAnnotation<JsonExclude>() == null }
```

### @JsonName

- 어노테이션의 인자를 JSON 키 이름으로 사용

```kotlin
val jsonNameAnn = prop.findAnnotation<JsonName>()
val propName = jsonNameAnn?.name ?: prop.name
```

### @CustomSerializer

- 지정한 `ValueSerializer` 클래스를 사용해 값 변환
    - 프로퍼티에 대해 정의된 커스텀 직렬화기가 있으면 그걸 사용
    - 없으면 일반적인 방법을 따라 프로퍼티 직렬화

```kotlin
val jsonValue = prop.getSerializer()?.toJsonValue(value) ?: value
```

## 12.2.4 JSON 파싱과 객체 역직렬화

### deserialize 함수 정의

- 타입 파라미터에 `reified`를 붙여야 런타임에 KClass로 접근 가능
    - 그로 인해 함수도 inline 으로 선언

```kotlin
inline fun <reified T: Any> deserialize(json: String): T
```

### 파서 구조

- `렉서(Lexer)`: JSON 문자열을 문자열을 토큰으로 나눔
- `파서(Parser)`: 토큰을 해석하여 JSON 구조 생성
- `역직렬화기`: 필요한 클래스의 인스턴스로 생성하여 반환
- JSON 객체 구조는 `JsonObject` 인터페이스로 표현

### Seed 인터페이스

- JSON으로부터 데이터를 모아 객체를 생성하는 구조
    - 일종의 빌더 역할을 수행
- `createCompositeProperty()` 메서드를 통해 내포된 객체나 내포된 리스트를 만들 때 사용
- `spawn()` 메서드를 통해 최종 객체 생성

```kotlin
interface Seed : JsonObject {
    fun spawn(): Any?
    fun createCompositeProperty(name: String, isList: Boolean): JsonObject
}
```

### ObjectSeed 구현

- 간단 값은 `valueArguments`, 복합 객체는 `seedArguments`에 저장
- `spawn()` 호출 시 두 맵을 합쳐 인자 맵을 생성자에 넘김

```kotlin
override fun spawn(): T = classInfo.createInstance(arguments)
```

## 12.2.5 callBy를 이용한 객체 생성

### call vs callBy

- `call(args...)`: 순서 맞춰야 하고 디폴트 인자 지원 안 함
- `callBy(mapOf(param to value))`: 순서 상관없고, 디폴트 인자 자동 적용

```kotlin
constructor.callBy(mapOf(param1 to value1, param2 to value2))
```

### ClassInfo 클래스

- 생성자 파라미터와 `@JsonName`, `@CustomSerializer` 등을 매핑
- `createInstance()`에서 callBy로 최종 인스턴스 생성

```kotlin
fun createInstance(args: Map<KParameter, Any?>): T {
    ensureAllParametersPresent(args)
    return constructor.callBy(args)
}
```

### 캐시 구조

- `ClassInfoCache`는 KClass를 키로 캐시된 ClassInfo를 저장
- `get(cls)` 호출 시 존재하면 반환, 없으면 새로 생성

## 요약

- KClass, KFunction, KProperty 등을 사용하면 실행 시점에 구조 정보를 활용할 수 있음
- 직렬화는 객체를 순회하며 각 프로퍼티를 문자열로 변환
- 역직렬화는 JSON을 파싱하여 객체의 생성자 인자로 맵핑
- 어노테이션(`@JsonName`, `@JsonExclude`, `@CustomSerializer`)을 통해 동작을 제어
- callBy로 생성자를 호출함으로써 디폴트 인자와 타입 안정성을 확보

# 오늘의 깜짝 퀴즈

## 문제 1

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

interface ValueSerializer<T> {
    fun toJsonValue(value: T): Any?
    fun fromJsonValue(jsonValue: Any?): T
}

class Date(val value: String)

object DateSerializer : ValueSerializer<Date> {
    override fun toJsonValue(value: Date): Any? = value.value
    override fun fromJsonValue(jsonValue: Any?): Date = Date(jsonValue as String)
}

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class CustomSerializer(
    val serializerClass: KClass<out ValueSerializer<*>>
)

data class Event(
    val title: String,
    @CustomSerializer(DateSerializer::class)
    val date: Date
)

fun KProperty1<*, *>.getSerializer(): ValueSerializer<Any?>? {
    val ann = findAnnotation<CustomSerializer>() ?: return null
    val klass = ann.serializerClass
    val instance = klass.objectInstance ?: klass.createInstance()
    @Suppress("UNCHECKED_CAST")
    return instance as ValueSerializer<Any?>
}

fun main() {
    val prop = Event::class.memberProperties
        .first { it.name == "date" } as KProperty1<Any, *>
    val serializer = prop.getSerializer()
    val value = serializer?.toJsonValue(Date("2025-06-03"))
    println(value)  // 출력: 2025-06-03
}

```

- 이 코드는 예상대로 동작할까?
- 문제가 있다면 어느 부분이 문제일까?



## 문제 2

다음은 코틀린 리플렉션을 사용해 객체를 생성하려는 코드이다.

```kotlin
import kotlin.reflect.KParameter
import kotlin.reflect.full.primaryConstructor

data class User(val name: String, val age: Int = 30)

fun main() {
    val kClass = User::class
    val constructor = kClass.primaryConstructor!!

    val args = mapOf<KParameter, Any?>(
        constructor.parameters.first { it.name == "name" } to "Alice"
    )

    val user = constructor.call(args)
    println(user)
}
```

- 이 코드는 정상적으로 컴파일될까?

