## 아이템 14. Comparable을 구현할지 고민하라

### 개요

- Primitive 자료형은 쉽게비교 및 정렬이 가능하다

```java
        int a=1;
        int b=2;

        System.out.println(a>b); // 부등호 비교, false 반환

        int[]integers={1,3,7,4,5};
        Arrays.sort(integers);
        System.out.println(Arrays.toString(integers)); // 정렬
```

- 객체의 비교는 어떻게 비교하고 정렬할 수 있을까?
```java
public class Book {
  private final String title;
  private final int price;

  // 생성자

  // getter
}
```

- String은 Comparable을 구현했기 때문에 Collection.sort로 정렬이 가능하다
```java
String a = "book";
        String b = "apple";
        List<String> list = Arrays.asList(a, b);

        System.out.println(list);

        Collections.sort(list);
        System.out.println(list); // 정렬됨
```

- Comparable을 구현하지 않은 객체를 정렬 하려고 할 때 에러 발생
```java
List<Book> books = Arrays.asList(
  new Book(1, "book"),
  new Book(1, "book"),
);

Collections.sort(books); // error: no suitable method found for sort(List<Book>)
```

### Comparable 이란?
```java
package java.lang;
import java.util.*;

public interface Comparable<T> {

    public int compareTo(T o);
    }
```
- 객체의 정렬 기준을 정하는데 사용되는 인터페이스
- compareTo 특징
  - 단순 동치성 비교
  - 순서 비교
  - 제네릭

- Comparable을 구현한 객체들의 배열은 손쉽게 정렬할 수 있다.
  - 자바 플랫폼 라이브러리의 모든 값 클래스와 열거 타입이 Comparable을 구현했다.

### Comparable 규약
```java
public class Person implements Comparable<Person> {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public int compareTo(Person other) {
        if (this.age < other.age) {
            return -1; // 이 객체가 매개변수로 들어온 객체보다 작음
        } else if (this.age > other.age) {
            return 1; // 이 객체가 매개변수로 들어온 객체보다 큼
        }
        return 0; // 두 객체가 같음
    }

    // Getter와 Setter 생략
}
```
- 객체(this)가 매개변수로 들어온 객체보다 작으면 음의 정수(-1), 같으면(0), 크면 (+1)을 반환
- 객체와 매개변수로 들어온 객체의 타입이 다르다면 ClassCastException을 반환한다.

#### 반사성
```java
x.compareTo(x) == true
```

#### 대칭성
```java
x.compareTo(y) == -y.compareTo(x)
```
- 두 객체 참조의 순서를 바꿔 비교해도 예상한 결과가 나와야 한다는 의미다.
- 즉, x.compareTo(y)가 1이라면 -y.compareTo(x)는 -1이 나와야 된다는 규약이다.


#### 추이성
```java
(x.compareTo(y) > 0 && y.compareTo(z) > 0) == x.compareTo(z) > 0
```
- 첫 번째가 두 번째보다 크고 두 번째가 세 번째보다 크다면 첫번째는 세번째 보다 크다

#### 일관성
```java
x.compareTo(y) == 0 == (x.compareTo(z) == y.compareTo(z))
```
- 크기가 같은 객체들끼리는 어떤 객체와 비교하더라도 항상 같아야 한다.

- 위 규약들은 equals의 규약과 동일하다.
- 마찬가지로 ‘기존 클래스를 확장한 구체 클래스에서 새로운 값 컴포넌트를 추가 했다면 compareTo 규약을 지킬 방법이 없다.’
- 우회법도 같다. (컴포지션, view 메서드)

##### 일관성 예외케이스
```java
(x.compareTo(y) == 0) == x.equals(y)
```
- 일관성 중 예외 케이스가 있다
- 필수가 아니지만 꼭 지키는게 좋다.
- compareTo 메서드로 수행한 동치성 테스트의 결과가 equals와 같아야 한다.
- 위 사항을 지키지 않고 정렬을 보장하는 성격의 컬렉션(TreeSet 등)에 넣으면 해당 컬렉션이 구현한 인터페이스에 정의한 동작과 엇박자를 낼 것이다.
- 정렬된 컬렉션(TreeSet 등)은 equals가 아닌 compareTo를 사용해 동치성을 비교하기 때문이다.
```java
        BigDecimal number1 = new BigDecimal("1.0");
        BigDecimal number2 = new BigDecimal("1.00");

        Set<BigDecimal> set1 = new HashSet<>();
        set1.add(number1);
        set1.add(number2);

        Set<BigDecimal> set2 = new TreeSet<>();
        set2.add(number1);
        set2.add(number2);

        System.out.println(set1); // 2개
        System.out.println(set2); // 1개
```
- HashSet은 원소를 2개 갖게 된다.
  - equals로 비교하면 서로 다르기 때문이다.
  - 스케일 정보까지 정확하게 확인한다
- TreeSet은 원소를 하나만 갖게 된다
  - compareTo 메서드로 비교하면 두 BigDecimal 인스턴스가 같기 때문이다.
  - compareTo는 스케일 정보를 버리고 비교하는듯 하다.
- 만약에 equals 비교시 같지 않은 경우가 있으면 문서화 권장

### compareTo 작성요령
- 타입을 인수로 받는 제네릭 인터페이스이므로 컴파일 시 인수 타입은 정해진다.
- 동치인지를 비교하는게 아니라 순서를 비교한다
- Comparable을 구현하지 않은 필드나 표준이 아닌 순서로 비교해야 한다면 비교자(Comparator)를 대신 사용한다.
  - 직접 만들 수도 있고 자바가 제공하는 것중에 사용해도 된다.
```java
  public int compareTo(CaseInsensitiveString cis) {
          return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
      }
```
  - 대소문자를 구분하지 않는 방식으로 문자열 비교

- 프리미티브 타입을 비교할 때 `<` 나 `>` 대신 Wrapper 클래스의 compare를 사용하자
  - 인자 순서는 명확하게 지켜줘야 한다.
```java
public class Position implements Comparable<Position> {
  private final int position;
  
  @Override
  public int compareTo(Position target) {
      return Integer.compare(position, target.position);
  }
}
```
  - compareTo 메서드에서 관계 연산자 <, >를 사용하는 이전 방식은 거추장스럽게 오류를 유발하니, 추천하지 않는다.
    - 가독성이 떨어져 부등호를 잘못 해석하여 발생하는 오류

- 클래스에 핵심 필드가 여러 개라면 핵심적인 필드부터 비교해 나가자
```java
public class Student implements Comparable<Student> {
    private int grade;
    private String name;
    private int age;

    @Override
    public int compareTo(Student o) {
        int result = Integer.compare(grade, o.grade);

        if (result == 0) {
            result = CharSequence.compare(name, o.name);
            if (result == 0) {
                result = Integer.compare(age, o.age);
            }
        }

        return result;
    }
}
```

### Comparator
- Comparator 인터페이스가 일련의 비교자 생성 메서드와 팀을 꾸려 메서드 연쇄 방식으로 비교자를 생성할 수 있게 되었다.
- 간결함에 매혹되지만, 약간의 성능 저하가 뒤따르게 된다.
  - 이제는 자바21이 나왔는데 아직까지도 성능이 좋지 않을까는 확인이 필요함
- 수많은 보조 생성 메서드들로 중무장하고 있다.
  - `comparingInt`
  - `comparingLong`
  - `thenComparingDouble`
  - …
- 또한, 객체 참조용 비교자 생성 메서드도 준비하고 있다.
- 예제 코드
```java
import java.util.Comparator;

public class Student implements Comparable<Student> {
    private static final Comparator<Student> COMPARATOR =
            Comparator.comparingInt((Student student) -> student.grade)
                    .thenComparing((Student student) -> student.name)
                    .thenComparingInt((Student student) -> student.age);

    private int grade;
    private String name;
    private int age;

    @Override
    public int compareTo(Student o) {
        return COMPARATOR.compare(this, o);
    }
}
```
- Comparator.comparingInt는 static 메서드로 객체를 생성
- 그 후 default 메서드를 이용하여 메서드 체이닝

#### Comparable와 Comparator의 비교
- Comparable
  - 자기 자신과 매개변수 객체를 비교
  - 주로 정렬해야 하는 클래스를 구현체로 만들어서 사용
  - long 패키지 → import 필요 x
- Comparator
  - 두 매개변수 객체를 비교
  - 주로 익명 객체로 구현해서 사용
    - 익명객체: 이름이 없는 객체
  - util 패키지 → import 필요


### 주의사항
- 두 값의 차이를 가지고 비교를 하는 방법도 있다. 아래의 코드처럼 hashCode를 비교한다고 해보자
  - 예시는 Comparator 이지만 Comparable도 마찬가지
  ```java
  private static final Comparator<Student> HASHCODE_COMPARATOR = new Comparator<Student>() {
  
  @Override
  public int compare(Student o1, Student o2) {
    return o1.hashCode() - o2.hashCode();
  }
  }
  ```
- 가장 큰 담점은 정수 오버플로우나 부동소수점 계산 방식에 따른 오류를 발생 시킬 수 있다.
  - 월등히 빠르지도 않다.
  - 뺼셈 과정에서 자료형의 범위를 넘어버리는 경우가 있다.
  - 실수 뺼셈의 경우에는 부동소수점 계산에 따른 오류가 날 수 있다.
- 따라서 아래와 같은 방식으로 적용하자
```java
private static final Comparator<Student> HASHCODE_COMPARATOR = new Comparator<Student>() {
    @Override
    public int compare(Student o1, Student o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};

private static final Comparator<Student> HASHCODE_COMPARATOR = Comparator.comparingInt(
            Object::hashCode);
```

### Tip
- Comparator 자리에 Comparable의 compareTo를 넣을 수 있다.
```java
cars.stream()
  .map(Car::getPosition)
  .max(Position::compareTo) // max의 인자는 원래 Comparator가 들어 간다
  .orElse(Position.fromStartLine());
```

### 결론
- 알파벳, 숫자, 연대 같이 순서가 명확한 값 클래스를 작성한다면 반드시 Comparable 인터페이스를 구현하자.
- `compareTo` 메서드에서 필드의 값을 비교할 때 `<` 와 `>` 연산자는 쓰지 말아야 한다. 그 대신 박싱된 기본 타입 클래스가 제공하는 `정적 compare` 메서드나 `Comparator` 인터페이스가 제공하는 비교자 생성 메서드를 사용하자.
- 두 값의 차이로 비교값을 사용하지 말자.
