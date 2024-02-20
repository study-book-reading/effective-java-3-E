## 아이템14. Comparable을 구현할지 고려하라

```java
//Comparable 인터페이스
public interface Comparable<T> {  //제네릭 사용

  public int compareTo(T o);    
}
```

Object의 equals와 다른 점 : 순서 비교 가능, 제네릭 적용.

equals와 같은데 굳이 Comparable을 구현한다는 것은 구현하는 해당 클래스의 인스턴스들에 자연적인 순서(natural order)가 있음을 의미.  

따라서 Comparable을 구현한 객체들의 배열은 정렬이 가능하다.
```java
Arrays.sort(a);
```

```java
public class WordList {
  public static void main(String[] args) {
    Set<String> s = new TreeSet<>();

    Collections.addAll(s, args);

    System.out.println(s);
  }
}
```

String클래스가 Comparable을 구현했기 때문에 명령줄 인수들을 (중복은 제거하고) 알파벳 순으로 출력한다.  

자바 플랫폼 라이브러리의 모든 값 클래스와 열거타입(아이템 34)이 Comparable을 구현했다.  
알파벳, 숫자, 연대 같이 순서가 명확한 값 클래스를 작성한다면 반드시 Comparable인터페이스를 구현하자.  


:star:**compareTo메서드의 일반 규약**  

**compareTo메서드 동작 방식**  
이 객체와 주어진 객체의 순서를 비교한다.  
이 객체가 주어진 객체보다 작으면 음의 정수를, 같으면 0을, 크면 양의 정수를 반환한다.  
이 객체와 비교할 수 없는 타입의 객체가 주어지면 ClassCastException을 던진다.  
다음 설명에서 sgn(표현식) 표기는 수학에서 말하는 부호 함수(signum function)를 뜻하며,  
표현식의 값이 음수, 0, 양수일 때, -1, 0, 1을 반환하도록 정의함.  

- Comparable을 구현한 클래스는 모든 x, y에 대해 sgn(x.compareTo(y)) == -sgn(y.compareTo(x))여야 한다(대칭성)  
(따라서 x.compareTo(y)는 y.compareTo(x)가 예외를 던질 때에 한 해 예외를 던져야 한다.)
- x.compareTo(y) > 0 && y.compareTo(z) > 0 이면, x.compareTo(z) > 0 이어야 한다.(추이성)  
- Comparable을 구현한 클래스는 모든 z에 대해 x.compareTo(y) == 0 이면, sgn(x.compareTo(z)) == sgn(y.compareTo(z)) 다.
- 필수는 아니지만 꼭 지키는 게 좋은 사항. (x.compareTo(y) == 0) == (x.equals(y)) 여야 한다. Comparable을 구현하고 이 권고를 지키지 않는 모든 클래스는 해당 사실을 명시헤야 한다. "주의: 이 클래스의 순서는 equals 메서드와 일관되지 않다." 

equals 메서드와 달리, compareTo메서드는 타입이 다르면 ClassCastException을 던지면 된다.  
비교를 사용하는 클래스의 예로 정렬된 컬렉션인 TreeSet과 TreeMap, 정렬 알고리즘을 활용하는 유틸리티 클래스인 Collections와 Arrays가 있다.  

equals와 동일하게 반사성, 대칭성, 추이성을 만족해야 하는 규약을 따르기 때문에, 주의사항도 똑같다.  
기존 클래스를 확장한 구체 클래스에서 새로운 값 컴포넌트를 추가했다면 compareTo 규약을 지킬 수 없다.  

우회법도 똑같이 확장 대신 독립된 클래스로 만들고 원래 클래스를 가리키는 인스턴스 필드를 둔 다.  
내부 인스턴스를 반환하는 '뷰' 메서드를 제공한다.  

**마지막 규약을 지켜야 하는 이유**  
compareTo로 줄지은 순서와 equals의 결과가 일관되게 된다.  
compareTo의 순서와 equals의 결과가 일관되지 않아도 동작은 한다.  
단, 이 클래스의 객체를 정렬된 컬렉션에 넣으면 해당 컬렉션이 구현한 인터페이스(Collection, Set 혹은 Map) 에 정의된 동작과 엇박자를 낼 것이다.  
정렬된 컬렉션들은 동치성 비교시 compareTo를 사용하기 때문이다.  


```java
public final Class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
  public int compareTo(CaseInsensitiveString cis) {
    return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
  }
}
```

Comparable<CaseInsensitiveString>을 구현했다.  
해당 클래스의 참조는 같은 클래스만 비교할 수 있다는 뜻으로, 일반적으로 따르는 패턴  

compareTo의 정수 기본 타입 필드 비교시 자바7에 추가된 박싱된 기본 타입 클래스들의 정적 메서드인 compare메서드를 사용하면 된다.  

비교시 핵심필드 먼저 비교하여 결과를 바로 반환하자.  
```java
public int compareTo(PhoneNumber pn) {
  int result = Short.compare(areaCode, pn.areaCode);
  if (result == 0) {
    result = Short.compare(prefix, pn.prefix);
    if (result == 0)
      result = Short.compare(lineNum, pn.lineNum);
  } 

  return result;
}
```

**비교자 생성**

자바 8에서 Comparator인터페이스가 메서드 연쇄 방식으로 비교자를 생성할 수 있다.  
이렇게 생성된 비교자들이 Comparable 인터페이스가 원하는 compareTo 메서드 구현에 활용될 수 있다.  
간결하지만 약간의 성능 저하가 있을 수 있다.  

```java
private static final Comparator<PhoneNumber> COMPARATOR = 
                                                comparingInt((PhoneNumber pn) -> pn.areaCode)
                                                  .thenComparingInt(pn -> pn.prefix)
                                                  .thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
  return COMPARATOR.compare(this, pn);
}
```

comparingInt에서 람다식에 입력 인수의 타입이 (PhoneNumber pn)로 명시한 이유는 자바의 타입 추론 능력이 이 상황에서 타입을 알아낼 만큼 강력하지 않기 때문이다.  
다음 thenComparingInt에선 타입 추론 능력이 충분하기 때문에 pn으로만 명시했다.  

short처럼 작은 경우 int용 버전을 사용하고, float은 double용을 사용하면 된다.  

```java
static Comparator<Object> hashCodeOrder = new Comparator<>() {
  public int compare(Object o1, Object o2) {
    return Integer.compare(o1.hashCode(), o2.hashCode());
  }
}
```

```java
static Comparator<Object> hashCodeOrder = Comparator.comparingInt(o -> o.hashCode());
```

---
### 결론
순서를 고려하는 값 클래스 작성시 Comparable 인터페이스를 구현하자.  
compareTo 메서드에서 필드 값 비교시 compare 메서드나 Comparator 인터페이스가 제공하는 비교자 생성 메서드를 사용하자. 

