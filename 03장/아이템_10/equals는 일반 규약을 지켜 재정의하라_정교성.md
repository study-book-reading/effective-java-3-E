## 아이템 10. equals는 일반 규약을 지켜 재정의하라

equals 메서드는 그냥 두면 그 클래스의 인스턴스는 오직 자기 자신과만 같게 된다.

```java
//Object의 equals
public boolean equals(Object obj) {
    return (this == obj);
}
```

:star:**equals메서드를 재정의하지 않는 상황**
- 각 인스턴스가 본질적으로 고유하다.
- 인스턴스의 '논리적 동치성(logical equality)'을 검사할 일이 없다.
- 상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는다.
- 클래스가 private하거나 package-private이고 equals 메서드를 호출할 일이 없다.

:star:**equals메서드를 재정의하는 상황**
- 객체 식별성(두 객체가 물리적으로 같은가)이 아니라 논리적 동치성을 확인해야 하는데, 상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의되지 않았을 때. 주로 값 클래스가 해당됨.

> **값 클래스란?**  
> Integer나 String 처럼 값을 표현하는 클래스

값 클래스를 재정의할 경우, 값이 같은지 비교가 가능하며, Map의 키와 Set의 원소로 사용할 수 있게 됨.(equals, hashCode 재정의시)

값 클래스여도 같은 값이 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제 클래스라면 equals를 재정의 하지 않아도 됨.

equals 메서드를 재정의할 때는 반드시 일반 규약을 따라야 함.  

:star:**Object명세에 적힌 규약내용**
- 반사성 : null이 아닌 모든 참조 값 x에 대해 x.equals(x) = true
- 대칭성 : null이 아닌 모든 참조 값 x, y에 대해 x.equals(y)가 true면, y.equals(x)도 true다.
- 추이성 : null이 아닌 모든 참조 값 x, y, z에 대해 x.equals(y)가 true고, y.equals(z)가 true면, x.euqals(z)도 true다.
- 일관성 : null이 아닌 모든 참조 값 x, y에 대해 x.equals(y)를 반복해서 호출하면 항상 true를 반환하거나 항상 false를 반환한다.
- null-아님 : null이 아닌 모든 참조 값 x에 대해 x.equals(null)은 false다.

\*컬렉션 클래스들을 포함해 수 많은 클래스는 전달받은 객체가 equals 규약을 지킨다고 가정하고 동작하기 때문에 이 규약을 어기면 프로그램이 이상하게 동작하는 문제가 생긴다.

> **동치관계란?**  
> 어떤 이항관계가 반사성, 대칭성, 추이성을 만족하는 관계.
> 착각하면 안되는 것은 이항관계가 equals처럼 같다는 것의 관계일 수도 있고 어떤 다른 특징일 수도 있음.

equals 메서드가 쓸모 있으려면 모든 원소가 같은 동치류에 속한 어떤 원소와도 서로 교환할 수 있어야 한다.  

---
#### 반사성  

```java
//Integer의 equals
public boolean equals(Object obj) {
    if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
    }
    return false;
}

Integer x = 1;
System.out.println(x.equals(x));    //true 반환 = 반사성 만족, 만약 false가 반환되면 반사성 위배
```

---
#### 대칭성

```java
public final class CaseInsensitiveString {
  private final String s;

  public CaseInsensitiveString(String s) {
    this.s = Objects.requireNonNull(s);
  }

  //대칭성 위배
  @Override
  public boolean equals(Object o) {
    if (o instanceof CaseInsensitiveString)  
      return s.equalsIgnoreCase(             
        ((CaseInsensitiveString) o).s);
    if (o instanceof String)                 
      return s.equalsIgnoreCase((String) o); 
    return false;                            
  }
}
```

CaseInsensitiveString은 String과도 비교를 시도한다.

```java
CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String s = "polish";

cis.equals(s)  // true
s.equals(cis)  // false    //String의 equals는 CaseInsensitiveString타입에 대하여 false로 반환.
```

```java
List<CaseInsensitiveString> list = new ArrayList<>();
list.add(cis);

list.contains(s)  // false 반환, 하지만 구현하기 나름이라 OpenJDK버전이 바뀌거나 할 경우 다른 결과가 나올 수 있다.(케바케)
```

위의 케바케 경우 때문에 equals 규약을 어기면 그 객체를 사용하는 다른 객체들이 어떻게 반응할지 예측이 불가함.

따라서 CaseInsensitiveString의 equals를 String과도 연동하면 안됨.

```java
@Override
public boolean equals(Object o) {
  return o instanceof CaseInsensitiveString &&
    ((CaseInsensitive) o).s.equalsIgnoreCase(s);    //String과 연동하지 않고 오로지 CaseInsensitiveString끼리만 비교
}
```

---
#### 추이성

```java
public class Point {
  private final int x;
  private final int y;

  public Point(int x, int y) {
    this.x = x;
    this.y = y;
  }

  @Override
  public boolean equals(Object o) {
    if (!(o instanceof Point))
      return false;
    Point p = (Point)o;
    return p.x == x && p.y == y;
  }
}

public class ColorPoint extends Point {
  private final Color color;

  public ColorPoint(int x, int y, Color color) {
    super(x, y);
    this.color = color;
  }
  //equals를 구현하지 않는다면 Point의 equals가 상속되어 Point끼리 비교 실행
}
```

대칭성 위배
```java
@Override
public boolean equals(Object o) {
  if (!(o instanceof ColorPoint))
    return false;
  return super.equals(o) && ((ColorPoint) o).color == color;
}
```
이 메서드는 일반 Point를 ColorPoint에 비교한 결과와 그 둘을 바꿔 비교한 결과가 다를 수 있다.  

```java
Point p = new Point(1, 2);
ColorPoint cp = new ColorPoint(1, 2, Color.RED);

// 대칭성 위배
p.equals(cp) // true
cp.equals(p) // false  
```

추이성 위배
```java
@Override
public boolean equals(Object o) {
  if (!(o instanceof Point))
    return false;

  // o가 일반 Point면 색상을 무시하고 비교.
  if (!(o instanceof ColorPoint))
    return o.equals(this);

  // o가 ColorPoint면 색상까지 비교
  return super.equals(o) && ((ColorPoint) o).color == color;
}
```
```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);

p1.equals(p2); // true
p2.equals(p3); // true
p1.equals(p3); // false
```

위 방식은 무한 재귀에 빠져서 StackOverFlowError를 일으킬 수도 있음.  
만약 Point의 또 다른 하위타입을 위와 같이 구현할 경우 무한 재귀에 빠진다.  

**그렇다면 해법은?**  

이 현상은 모든 객체 지향 언어의 동치관계에서 나타나는 근본적인 문제다.  
객체 지향적 추상화의 이점을 포기하지 않는 한 **구체 클래스를 확장해 새로운 값을 추가하면서 equals 규약을 만족시킬 방법은 존재하지 않는다.**

equals안의 instanceof 검사를 getClass검사로 바꾸면 규약도 지키고 값도 추가하면서 구체 클래스를 상속할 수 있다는 뜻으로 들리지만 아니다.

```java
@Override
public boolean equals(Object o) {
  if (o == null || o.getClass() != getClass())
    return false;
  Point p = (Point) o;
  return p.x == x && p.y == y;
}
```

이번 equals는 같은 구현 클래스의 객체와 비교할 때만 true를 반환함.  
괜찮아 보이지만 실제 활용 불가함.  
Point의 하위 클래스는 정의상 여전히 Point이므로 어디서든 Point로써 활용될 수 있어야 함.  

```java
private static final Set<Point> unitCircle = Set.of(
                                                new Point( 1,  0),
                                                new Point( 0,  1),
                                                new Point(-1,  0),
                                                new Point( 0, -1));

public static boolean onUnitCircle(Point p) {   //주어진 점이 (반지름이 1인) 단위원 안에 있는지 확인하는 메서드
  return unitCircle.contains(p);
}
```

```java
public class CounterPoint extends Point {
  private static final AtomicInteger counter = new AtomicInteger();

  public CounterPoint(int x, int y) {
    super(x, y);
    counter.incrementAndGet();
  }
  public static int numberCreated()  { return counter.get(); }
}
```

리스코프 치환 원칙에 따르면, 어떤 타입에 있어 중요한 속성이라면 그 하위 타입에서도 마찬가지로 중요하다.  
따라서 그 타입의 모든 메서드가 하위 타입에서도 똑같이 잘 작동해야 한다.  
"Point의 하위 클래스는 정의상 여전히 Point이므로 어디서든 Point로써 활용될 수 있어야 한다"는 뜻이다.  

> **리스코프 치환 원칙이란?**
> 서브 타입은 언제든지 기반 타입으로 교체할 수 있어야 한다는 것을 뜻함.  
> 부모 클래스의 인스턴스 대신 자식 클래스의 인스턴스를 사용했을 때 코드가 원래 의도대로 작동해야 함.   
> 다형성을 지원하기 위한 원칙임.  

CounterPoint의 인스턴스를 onUnitCircle 메서드에 넘기면?  
Point 클래스의 equals를 getClass를 사용해 작성했다면, onUnitCircle은 false 반환.  
CounterPoint 인스턴스의 x, y값과 무관하게 false 반환.  

원인은 컬렉션 구현체에서 주어진 원소를 담고 있는지 확인하는 방법 때문.  

컬렉션 구현체(HashMap, HashSet 등)에선 equals메서드를 이용.  
CounterPoint의 인스턴스는 어떤 Point와도 같을 수 없기 때문.  

```java
//HashMap의 Key비교로직(HashSet은 HashMap를 통해 구현했기 때문에 동일)

public boolean containsKey(Object key) {
    return getNode(key) != null;
}

final Node<K,V> getNode(Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n, hash; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & (hash = hash(key))]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))  //equals를 확인(getClass를 통해 비교시 Point와 CounterPoint는 false반환)
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))  //equals를 확인
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
}
```

구체 클래스의 하위 클래스에서 값을 추가할 방법은 없지만 괜찮은 우회 방법이 있다.  
"상속 대신 컴포지션을 사용하라"(아이템18)를 따르면 된다.  
Point를 상속하는 대신 Point를 ColorPoint의 private 필드로 두고,  
ColorPoint와 같은 위치의 일반 Point를 반환하는 뷰(view) 메서드(아이템6)를 public으로 추가한다.  

```java
public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }
    //Point 뷰 반환
    public Point asPoint(){
        return point;
    }

    @Override
    public boolean equals(Object o){
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
}
```

- ColorPoint vs ColorPoint : Point와 Color값까지 비교  
- ColorPoint vs Point : ColorPoint의 asPoint메서드로 Point값끼리만 비교

```java
ColorPoint cp1 = new ColorPoint(1,0, ColorPoint.Color.RED);

Point p1 = new Point(1,0);

System.out.println(cp1.asPoint().equals(p1));  //true, Point끼리 비교
```

추상클래스의 하위클래스라면 equals규약을 지키면서도 값을 추가할 수 있다.  
상위 클래스를 직접 인스턴스로 만들지 않기 때문에 위의 내용을 고민할 필요가 없다.  

---
#### 일관성

일관성이란? 두 객체가 같다면(어느 하나 혹은 두 객체 모두가 수정되지 않는 한) 앞으로도 같아야 한다는 뜻.

- 가변 객체는 비교 시점에 따라 서로 다르거나 같을 수도 있다.
- 불변 객체는 한 번 다르면 끝까지 달라야 한다.  
- 클래스가 불편이든 가변이든 equals의 판단에 신뢰할 수 없는 자원이 끼어들게 해서는 안된다.  

ex) java.net.URL의 equals는 주어진 URL과 매핑된 호스트의 IP주소를 이용해 비교한다.  
호스트 이름을 IP 주소로 바꾸려면 네크워크를 통해야하므로, 그 결과가 항상 같다고 보장할 수 없다.  
따라서 이런 문제를 피하려면 equals는 항시 메모리에 존재하는 객체만을 사용한 결정적 계산만 수행해야 한다.  

---
#### null-아님

모든 객체가 null과 같지 않아야 한다는 뜻.

```java
@Override
public boolean equals(Object o) {
  if (o == null)    // null을 확인하는 명시적 null 검사는 필요하지 않다.
    return false;
  //...
}

@Override
public boolean equals(Object o) {
  if (!(o instanceof MyType)) // 타입 검사를 통해 묵시적 null 검사를 한다.
    return false;
  MyType mt = (MyType) o;
  //...
}
```

:star:**양질의 equals 메서드 구현 방법 단계**

1. **== 연산자를 사용해 입력이 자기 자신의 참조인지 확인한다.**  
   자기 자신이면 true를 반환한다. 이는 단순한 성능 최적화용으로, 비교 작업이 복잡한 상황일 때 값어치를 할 것이다.
2. **instanceof 연산자로 입력이 올바른 타입인지 확인한다.**  
   그렇지 않다면 false를 반환한다.  
   이때의 올바른 타입은 equals가 정의된 클래스인 것이 보통이지만, 가끔은 그 클래스가 구현한 특정 인터페이스가 될 수도 있다.  
   어떤 인터페이스는 자신을 구현한 (서로 다른) 클래스끼리도 비교할 수 있도록 equals 규약을 수정하기도 한다.  
   이런 인터페이스를 구현한 클래스라면 equals에서 (클래스가 아닌) 해당 인터페이스를 사용해야 한다.  
   ex) Set, List, Map, Map.Entry 등의 컬렉션 인터페이스들
3. **입력을 올바른 타입으로 형변환한다.**  
   앞서 2번에서 instanceof 검사를 했기 때문에 이 단계는 100% 성공한다.
4. **입력 객체와 자기 자신의 대응되는 '핵심'필드들이 모두 일치하는지 하나씩 검사한다.**
   모든 필드가 일치하면 true를, 하나라도 다르면 false를 반환한다.
   2단계에서 인터페이스를 사용했다면 입력의 필드 값을 가져올 때도 그 인터페이스의 메서드를 사용해야 한다.
   타입이 클래스라면 (접근 권한에 따라) 해당 필드에 직접 접근할 수도 있다.

:star:**equals 메서드 구현시 주의사항**
1. 기본 타입 필드는 == 연산자로 비교
2. 참조 타입 필드는 각각의 equals 메서드로 비교
3. float과 double 필드는 각각 정적 메서드인 Float.compare(float, float)과 Double.compare(double, double)로 비교  
(Float.NaN, -0.0f, 특수한 부동소수 값 등을 다뤄야 하기 때문에 == 연산자로 비교하기엔 부족)
(Float.equals와 Double.equals 메서드들은 오토박싱을 수반할 수 있으니 성능상 좋지 않음)
4. 배열 필드는 원소 각각을 앞서의 지침대로 비교, 모든 원소가 핵심 필드라면 Arrays.equals 사용  
5. null도 정상 값으로 취급하는 참조 타입 필드는 정적 메서드인 Objects.equals(Object, Object)로 비교해 NullPointException 발생을 예방
6. 비교하기가 아주 복잡한 필드를 가진 클래스의 경우, 그 필드의 표준형을 저장해둔 후 표준형끼리 비교하면 훨씬 경제적이고 불변클래스에 알맞음.
7. 어떤 필드를 먼저 비교하느냐가 equals의 성능을 좌우하기도 함. 다를 가능성이 큰 필드들 끼리 혹은 비교하는 비용이 싼 필드를 먼저 비교
8. equals를 재정의할 땐 hashCode도 반드시 재정의하자(아이템 11).
9. 너무 복잡하게 해결하려 들지 말자. 필드들의 동치성만 검사해도 equals규약을 어렵지 않게 지킬 수 있다.
10. Object 외의 타입을 매개변수로 받는 equals 메서드는 선언하지 말자. ex) public boolean equals(MyClass o) { ... }, @Override를 명시함으로 실수 예방가능


equals를 구현 후 다음 세 가지를 자문해보고, 단위 테스트를 돌려봐라.  
1. 대칭적인가?
2. 추이성이 있는가?
3. 일관적인가?

위의 규약대로 구현한 예시
```java
public final class PhoneNumber {
  private final short areaCode, prefix, lineNum;

  public PhoneNumber(int areaCode, int prefix, int lineNum) {
    this.areaCode = rangeCheck(areaCode, 999, "지역코드");
    this.prefix = rangeCheck(prefix, 999, "프리픽스");
    this.lineNum = rangeCheck(lineNum, 9999, "가입자 번호");
  }

  private static short rangeCheck(int val, int max, String arg) {
    if (val < 0 || val > max)
      throw new IllegalArgumentException(arg + ": " + val);
    return (short) val;
  }

  @Override
  public boolean equals(Object o) {
    if (o == this)
      return true;
    if (!(o instanceof PhoneNumber))
      return false;
    PhoneNumber pn = (PhoneNumber)o;
    return pn.lineNum  == lineNum  &&
           pn.prefix   == prefix   &&
           pn.areaCode == areaCode;
  }
  //...생략
}
```
---
#### 결론

꼭 필요한 경우에만 equals를 재정의하자.  
IDE의 기능을 활용하거나, AutoValue, lombok, Immutables같은 도구를 사용하자.  

---
[참조]
- [이펙티브-자바-아이템-10.-equals는-일반-규약을-지켜-재정의-하라](https://inpa.tistory.com/entry/OOP-%F0%9F%92%A0-%EC%95%84%EC%A3%BC-%EC%89%BD%EA%B2%8C-%EC%9D%B4%ED%95%B4%ED%95%98%EB%8A%94-LSP-%EB%A6%AC%EC%8A%A4%EC%BD%94%ED%94%84-%EC%B9%98%ED%99%98-%EC%9B%90%EC%B9%99)
- [아주-쉽게-이해하는-LSP-리스코프-치환-원칙](https://velog.io/@lychee/%EC%9D%B4%ED%8E%99%ED%8B%B0%EB%B8%8C-%EC%9E%90%EB%B0%94-%EC%95%84%EC%9D%B4%ED%85%9C-10.-equals%EB%8A%94-%EC%9D%BC%EB%B0%98-%EA%B7%9C%EC%95%BD%EC%9D%84-%EC%A7%80%EC%BC%9C-%EC%9E%AC%EC%A0%95%EC%9D%98-%ED%95%98%EB%9D%BC)
- [Lombok, AutoValue and Immutables, or How to write less and better code returns](https://codeburst.io/lombok-autovalue-and-immutables-or-how-to-write-less-and-better-code-returns-2b2e9273f877)