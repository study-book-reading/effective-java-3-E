# #14 Comparable을 구현할지 고려하라

---

# Comparable 특징

```java
public interface Comparable<T> {
    public int compareTo(T o);
}
```

- 단순 동치성 비교에 더해 순서까지 비교할 수 있다.
- 제네릭하다.
- `Comparable`을 구현했다는 것은 그 클래스의 인스턴스들에는 자연적인 순서가 있음을 의미한다.
- 대부분의 자바 플랫폼 라이브러리의 모든 값 클래스와 열거 타입은 `Comparable` 을 구현함
- 알바펫, 숫자, 연대 같이 순서가 명확한 값 클래스를 작성하다면 `Comparable` 를 구현하자

---

# compareTo 메서드 일반 규약

객체와 주어진 객체의 순서를 비교한다. 이 객체가 주어진 객체보다 작으면 음의 정수, 같으면 0, 크면 양의 정수를 반환한다.

이 객체와 비교할 수 없는 타입의 객체가 주어지면 ClassCastException을 던진다.

- `Comparable`을 구현한 클래스는 모든 x,y에 대해 `sgn(x.compareTo(y))` == `-sgn(y.compareTo(x))`여야 한다
    - `x.compareTo(y)` 가 예외를 던지면 `y.compareTo(x)`도 예외를 던져야 한다
- `Comparable`을 구현한 클래스는 추이성을 보장해야 한다.
    - `x.compareTo(y) > 0` && `y.compareTo(z) > 0`이면 `x.compareTo(z) > 0` 이다.
- `Comparable` 을 구현한 클래스는 모든 z에 대해 `x.compareTo(y) == 0` 이면 `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`이다.
- (`x.compareTo(y) == 0 ) == (x.equals(y))` 여야 한다.
    - **이 권고는 필수는 아니지만 지키는 것이 좋다.**
    - 이 권고를 지키지 않는 모든 클래스는 그 사실을 명시해야한다.
    - **이 클래스의 순서는 equals 메서드와 일관되지 않다.**

`Comparable` 규약을 지키지 못하면 비교를 활용하는 클래스와 어울리지 못하다. 비교를 활용하는 클래스의 예로는 정렬된 컬렉션 TreeSet, TreeMap, Collections, Arrays가 있다.

기존 클래스를 확장한 구체 클래스에서 새로운 값을 추가했다면 `compareTo` 규약을 지킬 방법이 없다.

`Comparable`을 구현한 클래스를 확장해 값을 추가하고 싶다면 확장하는 대신 독립된 클래스를 만들고, 이 클래스에 원래 클래스의 인스턴스를 가리키는 필드를 두자. 그런 다음 내부 인스턴스를 반환하는 뷰 메서드를 제공하면 된다.

`Comparable` 을 구현하지 않은 필드나 표준이 아닌 순서로 비교해야 한다면 비교자 `Comparator`를 대신 사용한다.  비교자는 직접 만들거나 자바가 제공하는 것 중에 골라 쓰면 된다.

```java
public class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {

    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = s;
    }

    @Override
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
}
```

책의 2판에서는 `compareTo` 메서드에서 정수 기본 타입 필드를 비교할 때는 관계 연산자인 >,< 를 사용하고, 실수 기본 타입 필드를 비교할 떄는 정적 메서드 `Double.compare`, `Float.compare` 를 사용하라고 권했지만

자바 7부터는 박싱된 기본 타입 클래스들에 새로 추가된 정적 메서드인 `compare`를 사용하면 된다.

클래스에 핵심 필드가 여러 개라면 어느 것을 먼저 비교하느냐가 중요하다. 가장 핵심적인 필드부터 비교해나가자.

```java
public class PhoneNumber implements Comparable<PhoneNumber> {

    private int areaCode;
    private int prefix;
    private int lineNum;

    public PhoneNumber(
            int areaCode,
            int prefix,
            int lineNum
    ) {
        this.areaCode = areaCode;
        this.prefix = prefix;
        this.lineNum = lineNum;
    }

    @Override
    public int compareTo(PhoneNumber pn) {
        int result = Integer.compare(areaCode, pn.areaCode);
        if (result == 0) {
            result = Integer.compare(prefix, pn.prefix);
            if (result == 0) {
                result = Integer.compare(lineNum, pn.lineNum);
            }
        }

        return result;
    }
}
```

---

# Comparator

자바 8에서는 Comparator 인터페이스가 일련의 비교자 생성 메서드와 팀을 꾸려 메서드 연쇄 방식으로 비교자를 생성할 수 있게 되었다.

이 방식은 간결하지만, 약간의 성능 저하가 있다.

```java
public class PhoneNumber implements Comparable<PhoneNumber> {

    private int areaCode;
    private int prefix;
    private int lineNum;
    private static final Comparator<PhoneNumber> COMPARATOR = 
            Comparator.comparingInt((PhoneNumber pn) -> pn.areaCode)
                    .thenComparingInt(pn -> pn.prefix)
                    .thenComparingInt(pn -> pn.lineNum);

    public PhoneNumber(
            int areaCode,
            int prefix,
            int lineNum
    ) {
        this.areaCode = areaCode;
        this.prefix = prefix;
        this.lineNum = lineNum;
    }

    @Override
    public int compareTo(PhoneNumber pn) {
        return COMPARATOR.compare(this, pn);
    }
}
```

---

# 정리

- 순서를 고려해야 하는 값 클래스를 작성한다면 꼭 Comparable 인터페이스를 구현하여, 비교 기능과 컬렉션과 어우러지도록 해야 한다.
- 박싱된 기본 타입 클래스가 제공하는 정적 compate 메서드나 Comparator 인터페이스가 제공하는 비교자 생성 메서드를 사용하자.
    - `compateTo` 메서드에서 필드의 값을 비교할 때 <,> 연산자는 쓰지 말아야 한다