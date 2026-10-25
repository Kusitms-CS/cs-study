# Generic

## 제네릭이 필요한 이유

제네릭은 **클래스나 메서드에서 사용할 타입을 외부에서 지정**하는 기능이다. Java 5에서 도입되었다.

제네릭 이전에는 컬렉션이 `Object`를 저장했다.

```java
List list = new ArrayList();
list.add("hello");
list.add(123);                       // 아무 타입이나 들어감

String s = (String) list.get(1);     // 컴파일은 되지만 런타임에 ClassCastException
```

```java
List<String> list = new ArrayList<>();
list.add("hello");
list.add(123);                       // 컴파일 에러
String s = list.get(0);              // 형변환 불필요
```

- **컴파일 타임 타입 체크**: 잘못된 타입을 런타임이 아닌 컴파일 시점에 잡는다.
- **형변환 제거**: 꺼낼 때 캐스팅이 필요 없다.
- **코드 재사용**: 타입만 다른 같은 로직을 하나의 클래스/메서드로 작성한다.

## 제네릭 클래스와 메서드

```java
// 제네릭 클래스
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

Box<Integer> box = new Box<>();   // 다이아몬드 연산자 <> (Java 7+)

// 제네릭 메서드: 반환 타입 앞에 타입 파라미터 선언
public static <T> List<T> listOf(T a, T b) {
    return List.of(a, b);
}
```

관례적인 타입 파라미터 이름: `T`(Type), `E`(Element), `K`/`V`(Key/Value), `N`(Number), `R`(Result)

**타입 제한 (Bounded Type)**

```java
// Number의 하위 타입만 허용 → Number의 메서드(doubleValue 등)를 쓸 수 있음
public static <T extends Number> double sum(List<T> list) {
    return list.stream().mapToDouble(Number::doubleValue).sum();
}

// 여러 조건: 클래스가 먼저, 인터페이스는 &로 연결
public static <T extends Number & Comparable<T>> T max(List<T> list) { ... }
```

## 와일드카드와 PECS

제네릭은 **불공변(invariant)** 이다. `Integer`가 `Number`의 하위 타입이어도 `List<Integer>`는 `List<Number>`의 하위 타입이 아니다.

```java
List<Integer> ints = new ArrayList<>();
List<Number> nums = ints;   // 컴파일 에러
```

만약 허용된다면 `nums.add(3.14)`로 `List<Integer>`에 `Double`이 들어가 버리기 때문이다. (배열은 공변이라 `Object[] arr = new Integer[1]; arr[0] = "a";`가 컴파일되고 런타임에 `ArrayStoreException`이 난다.)

와일드카드(`?`)로 유연성을 준다.

| 표현 | 의미 | 읽기 | 쓰기 |
| --- | --- | --- | --- |
| `List<?>` | 아무 타입 | `Object`로 읽기 | `null`만 가능 |
| `List<? extends Number>` | Number 또는 그 하위 타입 | `Number`로 읽기 가능 | 불가 |
| `List<? super Integer>` | Integer 또는 그 상위 타입 | `Object`로만 읽기 | `Integer` 쓰기 가능 |

**PECS (Producer-Extends, Consumer-Super)**

- 값을 **꺼내서 쓰는(생산자)** 파라미터는 `extends`
- 값을 **집어넣는(소비자)** 파라미터는 `super`

```java
// Collections.copy의 시그니처
public static <T> void copy(List<? super T> dest, List<? extends T> src)
//                          소비자(넣기) → super    생산자(꺼내기) → extends
```

## 타입 소거 (Type Erasure)

제네릭 타입 정보는 **컴파일 시점에만 존재하고 바이트코드에서는 지워진다.** 제네릭이 없던 이전 버전 코드와의 하위 호환성을 위해서다.

```java
// 컴파일 전
public class Box<T> { private T value; public T get() { return value; } }
Box<String> box = new Box<>();
String s = box.get();

// 컴파일 후 (개념적으로)
public class Box { private Object value; public Object get() { return value; } }
Box box = new Box();
String s = (String) box.get();   // 컴파일러가 캐스팅을 삽입
```

- 제한 없는 `T` → `Object`로, `T extends Number` → `Number`로 치환된다.
- 필요한 곳에 컴파일러가 형변환을 넣어주고, 상속 시 다형성을 유지하기 위해 **브리지 메서드**를 만들기도 한다.
- 런타임에는 `List<String>`과 `List<Integer>`가 둘 다 그냥 `List`다.

## 제네릭의 제약

타입 소거 때문에 생기는 제약들이다.

```java
public class Box<T> {
    T t = new T();                       // X: 런타임에 T가 무슨 타입인지 모름
    T[] arr = new T[10];                 // X: 제네릭 배열 생성 불가
    static T staticField;                // X: static은 인스턴스마다 다른 T를 가질 수 없음
}

if (list instanceof List<String>) { }    // X: 런타임에 타입 인자 정보 없음
List<int> list;                          // X: 기본 타입 불가 → List<Integer> 사용 (박싱 비용)
```

- 런타임에 타입 정보가 필요하면 `Class<T>`를 파라미터로 넘긴다. (`objectMapper.readValue(json, User.class)`)
- `List<User>`처럼 중첩 제네릭 타입이 필요하면 Jackson의 `TypeReference`, Spring의 `ParameterizedTypeReference`처럼 **익명 하위 클래스의 상위 타입 정보**(이건 소거되지 않고 클래스 메타데이터에 남는다)를 이용하는 슈퍼 타입 토큰 기법을 쓴다.

```java
List<User> users = objectMapper.readValue(json, new TypeReference<List<User>>() {});
```
