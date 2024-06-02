## 아이템 26. 로(raw) 타입은 사용하지 말라

### 제네릭이란
- 제네릭은 자바5부터 추가되었고, 개발자가 안전하게 코딩할 수 있게 도와준다.

### 제네릭 등장 이전

```java
// Generic 사용하기 전
List numbers = new ArrayList();
numbers.add(10);
numbers.add("whiteship");

for (Object number: numbers) {
    System.out.println((Integer)number); // 문자열 꺼낼 시 에러
}
```

- 로우 타입을 사용한 컬렉션에는 Object 타입을 받기 때문에 아무 타입을 넣을 수 있다.
- 문제는 컬렉션에서 형변환 하고 값을 꺼내서 사용할 때 의도치 않은 타입이 들어가 있는걸 꺼내려고 할때 에러가 발생한다.

### 제네릭 등장 이후

```java
List<Integer> nuberms = new ArrayList<>();
nuberms.add(10);
nuberms.add("whiteship"); // 컴파일 에러

for (Integer number: nuberms) {
    System.out.println(number);
}
```

- 제네릭을 사용하면 컴파일 시점에 잘못된 타입을 알 수 있다.
- 꺼낼때 형변환을 할 필요 없다.

### 제네릭 용어

제네릭 용어를 정리할 때 **무언가 정의하는 입장과 사용하는 입장**으로 나눠서 생각하면 조금 더 쉽게 이해할 수 있다.

#### 제네릭 타입(정의)

```java
public class Box<E> {
```
- `Box<E>`는 제네릭 타입
- Box 클래스가 E라는 매개변수를 사용할 수 있다
- 어떤 타입이 사용될지 모르지만 E라고 지칭

#### 타입 매개변수(정의)

```java
E
```

#### 실제 타입 매개변수(사용)

```java
Box<Integer> box = new Box<>();
```

- `<Integer>`는 실제 타입 매개변수라고 부른다.

#### 매개변수화 타입(사용)

```java
Box<Integer> box = new Box<>();
```

- `Box<Integer>`는 매개변수화 타입
- Integer로 매개변수화된 Box 타입

#### 한정적 타입 매개변수(정의)

```java
public class Box<E extends Number> { // 한정적 타입 매개변수
-------------------------------------
  Box<Integer> box = new Box<>(); // 사용 가능
  Box<String> box = new Box<>(); // 컴파일 에러
```
- 한정적인 타입만 사용하도록 제한

#### 비한정적 와일드카드 타입(사용)

```java
Box<?> box = new Box<>();
```

- ?는 비한정적 와일드 카드 타입
    - 비한정적은 super나 extends가 없는 것
    - 와일드 타입은 아무 타입을 의미
- 와일드 카드는 컬렉션에 값을 추가할때 쓰는 용도는 아님
- 와일드 카드를 쓰면 컬렉션에 아무것도 넣을 수 없음

```java
Box<?> box = new Box<>();
box.add(1); // 컴파일 에러
box.add("10"); // 컴파일 에러
```
- 와일드 카드 타입에 추가시 컴파일 에러 발생

```java
private static void printBox(Box<?> box) {
    System.out.println(box.get());
}
```

- `Box<?> box`는 아무런 타입의 박스가 와도 된다. 그래서 Box box와 같이 로 타입으로 해도 되지만 안전하지 않다.

```java
Box<Integer> box = new Box<>();
box.add(1);
printBox(box); // 컴파일 에러

private static void printBox(Box<String> box) {
    System.out.println(box.get());
}
```

- String된 박스만 받는 경우 Integer로된 박스는 받을 수 없다.

#### 한정적 와일드카드 타입(사용)

```java
Box<? extends Number> box = new Box<>();
```

#### Object

```java
Box<Integer> box = new Box<>();
box.add(1);
printBox(box); // 컴파일 에러

private static void printBox(Box<Object> box) {
    System.out.println(box.get());
}
```

- Integer는 Object의 하위 클래스지만 **Object 박스는 Integer 박스를 받을 수 없다.**
    - 이러한 특징으로 인해 제네릭은 불공변하다라고 한다.
    - 참고로 배열은 공변하기 때문에 `Object[] objects = new String[1];`이 가능하다.
      - 하지만 배열의 이러한 특징은 런타임에 에러가 발생할 수 있다.
- 와일드 카드 타입을 사용해야 한다
  - Box<? extends Object> box
  - Box<?> box

```java
  private static void printBox(Box<? extends Object> box) {
        System.out.println(box.get());
    }
```
- `? extends Object`는 Object를 상속 받는 임의의 타입을 의미
  - `extends Object`는 생략해도 됨

```text
공변: A가 B의 하위 타입이면, T[A]는 T<B>의 하위 타입이다.
Super 클래스 - Sub 클래스 (상속 관계)
Super[] - Sub[] (상속 관계)

무공변/불공변성: A가 B의 하위 타입이어도 T<A>와 T<B>는 관계가 없다.
Super 클래스 - Sub 클래스 (상속 관계)
List<Super> List<Sub> (아무 관계 아님)
```
- `? extends Object` 예시를 보면 알겠지만 한정적 와일드 카드를 사용하여 제네릭도 공변성을 가질 수 있다. 

### 로타입을 강제하지 않는 이유
- 자바에서 로타입을 강제하지 않은 이유는 자바5 이전의 코드를 지원할 수 있도록 하기 위함이다.
- 자바는 출시된 지 10년 후 제네릭을 추가했으며, 이전 코드와의 호환성을 유지하기 위해 로타입을 지원하는 eraser 기법을 사용한다.
- eraser 기법은 컴파일시 제네릭이 다 지워지는 기법이다.

```text
자바 5이전: 로타입 → 컴파일 → 로타입
자바 5이후: 제네릭 → 컴파일(이레이저 기법) → 로타입
```

```java
public static void main(String[] args) {
    Box<Integer> box = new Box<>();
    box.add(10);
    System.out.println(box.get() * 100); // 주목

    printBox(box);
}

private static void printBox(Box<?> box) {
    System.out.println(box.get());
}
```

```java
INVOKEVIRTUAL me/whiteship/chapter05/item26/terms/**Box.get ()Ljava/lang/Object**;
**CHECKCAST java/lang/Integer**
INVOKEVIRTUAL java/lang/Integer.intValue ()I
BIPUSH 100
```

- 첫번째 줄: `box.get() * 100` 에서 box.get()시 Object로 꺼냄
- 두번째 줄: Object를 Integer로 캐스팅
- 컴파일시 소스 코드에 명시한 타입은 없어지지만 소스 코드에서 넣은 타입을 기반으로 캐스팅을 한다.


### 로타입 단점
- 로 타입과 Object를 선언한 타입의 차이는 무엇일까?
    - `List` vs `List<Object>`
- **안정성과 표현력**의 차이가 있다.

#### 예시1

```java
// 코드 26-4 런타임에 실패한다. - unsafeAdd 메서드가 로 타입(List)을 사용 (156-157쪽)
public class Raw {
    public static void main(String[] args) {
        List<String> strings = new ArrayList<>();
        unsafeAdd(strings, Integer.valueOf(42));
        String s = strings.get(0); // 컴파일 에러 발생
    }

    private static void unsafeAdd(List list, Object o) {
        list.add(o); // String 타입의 제네릭에 Integer 값을 넣음
    }
}

```

- get 으로 꺼낼때 ClassCastException 에러가 발생한다.
    - String을 기대했지만 Integer가 튀어나오기 때문이다.
- 리스트에 넣을때 잘못 들어가서 안정성이 깨졌다.
    - 타입 안전성을 잃음

- unsafeAdd의 매개변수는 `(List 로타입, Object)`로 이루어져 있음
    - 어떤 리스트 타입이 와도 Object이기 때문에 무조껀 넣을 수 있음
    - 위 예시 처럼 `List<String>`으로 와도 Integer 값을 넣을 수 있음
    - `List<String>, String o` 와 같이 작성했었어야 맞음
- 그렇다면 `List<Object>`면 넣을 수 있을까?
```java
public static void main(String[] args) {
    List<String> strings = new ArrayList<>();
    unsafeAdd(strings, Integer.valueOf(42)); // 컴파일 에러 발생
    String s = strings.get(0); 
}

private static void unsafeAdd(List<Object> list, Object o) {
        list.add(o); 
}
```
- `List<String>`과 `List<Object>`는 제네릭의 불공변성 특징으로 인해 List<Object>에 List<String> 넣을 수 없다.

```java
public static void main(String[] args) {
  List<String> strings = new ArrayList<>();
  unsafeAdd(Collections.singletonList(strings), "test";
  String s = strings.get(0); 
}
 
private static void unsafeAdd(List<Object> list, Object o) {
  list.add(o);
}
```
- `Collections.singletonList`를 사용하면 ``List<Object>`에 넣을 수는 있다.
  - `Collections.singletonList(strings)`는 List<List<String>>를 반환, List는 Object의 하위 타입이기 때문에 넣을 수 있다.
- 하지만, List<List<String>>에 Object를 넣을 수 없기 때문에 add시 `UnsupportedOperationException`이 발생한다.

#### 예시2

```java
static int numElementsInCommon(Set<?> s1, Set<?> s2) {
    int result = 0;
    for (Object o1 : s1) {
        if (s2.contains(o1)) {
            result++;
        }
    }

    return result;
}

public static void main(String[] args) {
    System.out.println(Numbers.numElementsInCommon(Set.of(1, 2, 3), Set.of(1, 2)));
}
```

- `static int numElementsInCommon(Set s1, Set s2) {` 로 쓰면 안정성이 깨진다
    - Set 컬렉션에 모든 타입을 넣을 수 있어서 그렇다.
- `Set<?> s1, Set<?> s2` 는 각각 한 종류만 다루는 모든 타입을 받겠다
    - 아래 케이스 모두 가능

```java
Numbers.numElementsInCommon(Set.of(1, 2, 3), Set.of(1, 2))
Numbers.numElementsInCommon(Set.of("1", "2", "3"), Set.of(1, 2))
Numbers.numElementsInCommon(Set.of(1, 2, 3), Set.of(1, 2))
```

```java
Set<Integer> set1 = new HashSet();
Set<String> set2 = new HashSet();

Set<?> kwc = set1; // 이것도 되고
kwc = set2; // 이것도 된다.
```

- **<?>는 어떤 타입이라도 담을 수 있는 가장 범용적인 매개변수 변수와 셋 타입이다.**
- 로 타입도 아무거나 받을 수 있으니 똑같은거 아닌가요?
    - 로타입은 아무 값이나 추가할 수 있어서 안전하지 않음
    - 그러나 와일드 카드 타입은 null을 제외하면 아무것도 추가할 수 없어서 안전하다
    - 즉, 안정성의 차이
- 대부분 모든 경우에 타입을 사용해라.
- 로 타입을 써야하는 예외적인 경우도 있다.

### 로 타입을 사용해야 하는 경우

#### .class를 쓰는 경우
```java
public class UseRawType<E> {

    private E e;

    public static void main(String[] args) {
        System.out.println(UseRawType<Integer>.class); // 컴파일 에러
        System.out.println(UseRawType.class);
    }
}
```
- `UseRawType<Integer>.class` 컴파일 에러
    - `Cannot access class object of parameterized type`
- **자바에선 .java가 아닌 컴파일된 .class 파일을 참고 한다.**
- 하지만 컴파일 타임에서 제네릭은 제거되므로 UseRawType.class만 남는다.
- 따라서 컴파일 타임에 `UseRawType<String>.class`가 없기 때문에 소스 코드 상으로도 사용할 수 없다.

#### instanceof를 쓰는 경우

```java
public class UseRawType<E> {
    private E e;
    public static void main(String[] args) {
        UseRawType<String> stringType = new UseRawType<>();
        System.out.println(stringType instanceof UseRawType);
        System.out.println(stringType instanceof UseRawType<String>); // 가능
    }
}
```

- instanceof에는 제네릭을 UseRawType<String>으로 쓸수는 있지만 제네릭은 어차피 소거됨
- 코드를 장황하게 만드는 일
- .class와 instanceof를 쓸때는 로타입을 사용하여 나머지 경우는 모두 매개변수화 타입을 사용하는 것을 권장