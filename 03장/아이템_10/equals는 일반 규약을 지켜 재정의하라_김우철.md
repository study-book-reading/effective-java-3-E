## 아이템 10 equals는 일반 규약을 지켜 재정의하라

### 개요

- equals 메서드는 재정의하기 쉬워 보이지만 곳곳에 함정이 도사리고 있어서 자칫하면 끔찍한 결과를 초래한다
- 다음 열거한 상황 중 하나에 해당한다면 재정의 하지 않는 것이 최선이다.

1. 각 인스턴스가 본질적으로 고유하다.
    - 싱글턴 패턴, enum 등은 인스턴스가 단 하나만 존재하기 때문에 equals를 구현할 필요가 없다.
2. 인스턴스의 ‘논리적 동치성’을 검사할 일이 없다.
    - 내용이 같으면 같은것이라 판단하겠다.
    - 대표적으로 문자열이 있다. Hello랑 Hello는 같다.
3. 상위 클래스에서 재정의한 equals가 제대로 잘 정의되어 있다면 하위 클래스에서는 equals를 직접 구현할 필요가 없다.
    - ArrayList 등의 상위 클래스인 AbstractList 클래스에서 equals가 잘 구현되어 있고, 구현체 들은 모두 같은걸 사용한다.
4. 클래스가 private이거나 package-private이고 equals 메서드를 호출할 일이 없다.

### equals는 언제 재정의 해야 할까?

- 객체 식별성이 아니라 논리적 동치성을 확인해야 하는데, 상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의되지 않았을 때다.

### Object 명세에 적힌 equals 규약

- equals 메서드는 반드시 동치관계(equivalence relation)을 구현하며, 다음을 만족한다.

1. 반사성 (reflexivity) : null이 아닌 모든 참조 값 x에 대해 x.equals(x) 는 true이다.
2. 대칭성 (symmetry) : null이 아닌 모든 참조 값 x, y에 대해 x.equals(y)가 true이면 y.equals(x)도 true이다.
3. 추이성 (transitivity) : null이 아닌 모든 참조 값 x, y, z에 대해 x.equals(y)가 true이고 y.equals(z)도 true이면, x.equals(z)도 true이다.
4. 일관성 (consistency) : null이 아닌 모든 참조 값 x, y에 대해 x.equals(y)를 반복해서 호출하면 항상 true를 반환하거나 항상 false를 반환해야 한다.
5. null 아님 : null이 아닌 모든 참조 값 x에 대하여 x.equals(null)은 false이다.

### 반사성

```java
x.equlas(x)==true
```

- null이 아닌 모든 참조 값 x에 대해 x.equals(x) 는 true이다.
- 객체는 자기 자신과 같아야 한다
- 인스턴스가 들어있는 컬렉션에 contains 메서드를 호출해 true가 나오면 반사성을 만족한 것이다.

```java
@Override
public boolean equals(Object o){
        if(o==null){
        return false;
        }
        return this==o;
        }
```

```java
public static void main(String[]args){
        List<Member> members=new ArrayList<>();
        Member member=new Member("rutgo",29);
        members.add(member);

        System.out.println(members.contains(member)); // true
        }
```

### 대칭성

```java
x.equals(y)==y.equals(x)
```

- null이 아닌 모든 참조 값 x, y에 대해 x.equals(y)가 true이면 y.equals(x)도 true이다.
- 대칭성 위배 예시

```java
// 잘못된 코드 - 대칭성 위배
public final class CaseInsensitiveString {
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }

    // 잘못된 코드 - 대칭성 위배
    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
        if (o instanceof String) // 한 방향으로만 작동
            return s.equalsIgnoreCase((String) o);
        return false;
    }
	... // Remainder omitted
}
```

- CaseInsenstiveString을 String과 동급으로 쓰겠다 라는 의도인데 잘못된 생각이다
    - String은 CaseInsenstiveString을 알지 못한다.

```java
        String polish="polish";
        List<CaseInsensitiveString> list=new ArrayList<>();
        list.add(cis);
        System.out.println(list.contains(polish)); // false
```

- 위와 같은 방식도 false가 나온다

```java
public boolean contains(Object o){
        Iterator<E> it=iterator();
        if(o==null){
        while(it.hasNext())
        if(it.next()==null)
        return true;
        }else{
        while(it.hasNext())
        if(o.equals(it.next())) // 해당 부분
        return true;
        }
        return false;
        }
```

- contains의 구현 방식을 살펴보면 o.equals()를 사용한다. 즉, String.equals()를 사용하기 때문에 false로 나온다.
- 문제를 해결 하려면 간단하다. 내가 만든 타입끼리만 비교하면 된다.

```java
@Override public boolean equals(Object o){
        return o instanceof CaseInsensitiveString&&
        ((CaseInsensitiveString)o).s.equalsIgnoreCase(s);
        }
```

### 추이성

```java
x.equals(y)==true
        y.equals(z)==true
        x.equals(z)==true
```

- 첫 번째 객체와 두 번째 객체가 같고, 두 번째 객체와 세 번째 객체가 같다면, 첫 번째 객체와 세 번째 객체도 같아야 한다

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

        Point p = (Point) o;
        return p.x == x && p.y == y;
    }
	... // Remainder omitted
}
```

- 상위 클래스

```java
public class ColorPoint extends Point {
    private final Color color; // 추가

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }
	... // Remainder omitted
}
```

- 하위 클래스에 Color 필드 추가

```java
// 대칭성 위배
@Override public boolean equals(Object o){
        if(!(o instanceof ColorPoint))
        return false;
        return super.equals(o)&&((ColorPoint)o).color==color;
        }
```

- ColorPoint의 equals 메서드
- 하위 클래스에서 Color 컬러가 추가
    - 부모 클래스에 있는 equals를사용해서 x, y 좌표값과 추가된 컬러값을 비교하고 있다.
  ```java
    Point p = new Point(1,2);
    ColorPoint cp = new ColorPoint(1,2,RED);
    p.equals(cp) // true
    cp.equals(p) // false
  ```
    - p.equals(cp): point의 equals는 색상을 무시하므로 true
    - cp.equals(p): ColorPoint의 equals는 같은 ColorPoint가 아니라서 매번 false

```java
// 추이성 위반
@Override public boolean equals(Object o){
        if(!(o instanceof Point))
        return false;

        // o가 일반 Point면 색상을 무시하고 비교한다.
        if(!(o instanceof ColorPoint))
        return o.equals(this);

        // o가 ColorPoint면 색상까지 비교한다.
        return super.equals(o)&&((ColorPoint)o).color==color;
        }
```

- Point <-> ColorPoint간의 대칭성을 만족한 코드
- 그러나 추이성을 위반한다.

```java
    ColorPoint p1=new ColorPoint(1,2,RED);
        Point p2=new Point(1,2);
        ColorPoint p3=new ColorPoint(1,2,BLUE);

        p1.equals(p2) // true
        p2.equals(p3) // true
        p1.equals(p3) // false
```

- p1-p2, p2-p3 비교에서는 색상을 무시 했지만, p1-p3에서는 색상까지 고려 했기 때문이다.

- 구체 클래스를 확장해 새로운 값을 추가 하면서 equals 규약을 만족시키는 방법은 없다.
- 그렇다면 서브 클래스는 서브클래스 끼리만 비교하도록 해야 하는 걸까?

```java
public class Point {

    @Override
    public boolean equals(Object o) {
        // getClass
        if (o == null || o.getClass() != this.getClass()) {
            return false;
        }

        Point p = (Point) o;
        return this.x == p.x && this.y = p.y;
    }

}
```

- 이렇게 하면 같은 객체일때만 비교할 수 있다. Point와 ColorPoint는 무조껀 다르다고 할 것이다.
- 하지만, 리스코프 치환 원칙(LSP)을 위반한다.
    - 리스코프 치환 원칙은 상위 클래스 타입으로 동작하는 코드를 하위 클래스 타입의 인스턴스를 주더라도 동일하게 동작 해야하는 원칙

```java
   private Set<Point> unitCircle=Set.of(
        new Point(1,0),new Point(0,1),
        new Point(-1,0),new Point(0,-1));

        Point p1=new Point(1,0);
        Point p2=new CounterPoint(1,0);

        unitCircle.contains(p1); // true
        unitCircle.contains(p2); // false
```

- unitCircle.contains(p2);도 false가 나오는데 이도 리스코프 치환 원칙을 위반한 것이다.

```java
        // getClass
        if(o==null||o.getClass()!=this.getClass()){
                return false;
                }
```

- 이유는 getClass를 사용해서 Point와 ColorPoint를 구분하기 때문이다.
- 핵심은 ColorPoint도 결국 Point이므로 Point로 활용될 수 있어야 한다는 점이다.

- 가장 좋은 방법은 아이템 18의 조언처럼 상속 대신 컴포지션을 사용하라 과 뷰(view) 메서드를 Public으로 추가하는 것이다.

```java
// 코드 10-5 equals 규약을 지키면서 값 추가하기 (60쪽)
public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    /**
     * 이 ColorPoint의 Point 뷰를 반환한다.
     */
    public Point asPoint() {
        return point;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }

    @Override
    public int hashCode() {
        return 31 * point.hashCode() + color.hashCode();
    }
}
```

```java
    Point p =new ColorPoint().asPoint()
```

- 포인트 뷰를 이용하면 ColorPoint도 Point로 취급할 수 있다.

- 자바 라이브러리에서도 잘못된 상속의 equals() 구현한 경우가 있는데 바로 java.sql.Timestamp와 java.util.Date가 된다.

```java
public class Date
        implements java.io.Serializable, Cloneable, Comparable<Date> {
    public boolean equals(Object obj) {
        return obj instanceof Date && getTime() == ((Date) obj).getTime();
    }
}
```

```java
public class Timestamp extends java.util.Date {
    public boolean equals(Timestamp ts) {
        if (super.equals(ts)) {
            if (nanos == ts.nanos) {
                return true;
            } else {
                return false;
            }
        } else {
            return false;
        }
    }
```

```java
        long time=System.currentTimeMillis();
        Timestamp timestamp=new Timestamp(time);
        Date date=new Date(time);

        // 대칭성 위배! P60
        System.out.println(date.equals(timestamp)); // true
        System.out.println(timestamp.equals(date)); // false
```
- Timestamp는 Date를 확장 한 후 nanoseconds 필드를 추가했다.
- 그 결과로 Timestamp의 equals는 대칭성을 위배하며, Date 객체와 한 컬렉션에 넣거나 서로 섞어 사용하면 엉뚱하게 동작할 수 있다.

### 일관성
```java
x.equals(y) == true;
x.equals(y) == true;
x.equals(y) == true;
```
- 두 객체가 같다면 앞으로도 영원히 같아야 한다는 뜻이다.
- 일관성은 객체안에 들어있는 값이 바뀌는 경우에는 깨질 수가 있다
- 그러나 불변객체면 일관성이 항상 보장된다.

```java
 // 일관성 위배 가능성 있음. P61
        URL google1 = new URL("https", "about.google", "/products/");
        URL google2 = new URL("https", "about.google", "/products/");
        System.out.println(google1.equals(google2));
```
- virtual host인 경우는 실제 IP 값이 도메인이 가리킨 IP 값이 바뀔 수도 있다
- 일반적인 도메인이라도 IP 가 바뀔 수가 있다
- 심볼링 링크도 마찬가지다.
- 책에서는 IP까지는 복잡하게 체크하지말고 도메인 네임이 같으면 같다고 판단하는걸 권장한다.

### null 아님
```java
x.equals(null) == false;
```
- null이 아닌 모든 참조 값 x에 대해, x.equals(null)은 false이다.
```java
@Override
public boolean equals(Object o) {
    if (!(o instanceof Member)) {
        return false;
    }
    Member member = (Member)o;

    return email.equals(member.email) && name.equals(member.name) && age.equals(member.age);
}
```
- equals()를 재정의 시 타입 검사를 하게 된다면 null 검사를 따로 할 필요 없이 지켜지게 된다. 이걸 묵시적 null 검사라고 한다.

### equals 메서드 구현 방법 단계별 정리
1. == 연산자를 사용해 입력이 자기 자신의 참조인지 확인한다
    1. 단순한 성능 최적화용으로 비교 작업이 복잡한 상황일 때 값어치를 한다.
2. instanceof 연산자로 입력이 올바른 타입인지 확인한다.
    1. **null 아님**에서 언급했듯이 묵시적 null 체크를 할 수 있다.
    2. 또한 Collection interface처럼 특정 인터페이스의 타입 체크를 할 수도 있다.
3. 입력을 올바른 타입으로 형변환 한다
    1. instanceof 검사를 했기 때문에 100% 성공한다
4. 입력 객체와 자기 자신의 대응되는 핵심 필드들이 모두 일치하는지 하나씩 검사한다

```java
@Override
public boolean equals(final Object o) {
    // 1. == 연산자를 사용해 입력이 자기 자신의 참조인지 확인한다.
    if (this == o) {
        return true;
    }

    // 2. instanceof 연산자로 입력이 올바른 타입인지 확인한다.
    if (!(o instanceof Point)) {
        return false;
    }

    // 3. 입력을 올바른 타입으로 형변환 한다.
    final Piece piece = (Piece) o;

    // 4. 입력 개체와 자기 자신의 대응되는 핵심 필드들이 모두 일치하는지 하나씩 검사한다.
    
    // float와 double을 제외한 기본 타입 필드는 ==를 사용한다.
    return this.x == p.x && this.y == p.y;
    
    // 필드가 참조 차입이라면 equals를 사용한다.
    return str1.equals(str2);
    
    // null 값을 정상 값으로 취급한다면 Objects.equals로 NullPointException을 예방하자.
    return Objects.equals(Object, Object);
}
```

### 주의사항
- 기본 타입은 ==으로 비교하고 그 중 double, float는 Double.compare(), Float.compare()을 이용해 검사한다. 
  - 이유는 부동소수점을 다뤄야 하기 때문이다.
- 배열의 모든 원소가 핵심 필드이면 Arrays.equals 메서드(오버로딩)들 중 하나를 사용하자.
- null이 정상 값으로 취급할 때는 Objects.equals()를 이용해 NPE를 방지하자
- 최상의 성능을 바란다면 다를 가능성이 더 크거나 비교하는 비용이 싼 필드를 먼저 비교하자.
- 동기화용 락 필드 같이 객체의 논리적 상태와 관련 없는 필드를 비교하면 안된다.
  - 락은 어떤 객체가 하나가 가지고 있는 특수한 필드인게 아니라서 그렇다.
- equals를 재정의할 땐 hashCode도 반드시 재정의 하자
- Object **외의** 타입을 매개변수로 받는 equals 메서드는 선언하지 말자

### equals를 대신 구현 해주는 방법
- equals를 직접 구현하면 실수할 여지가 많다.
- 그래서 툴을 사용하면 좋다
  - google 라이브러리의 AutoValue
  - lombok의 @EqualsAndHashCode
  - record
  - IDE의 자동 생성 기능

### 결론
- equals는 필요할 때만 재정의 하자(아이템 11)
- equals를 다 구현했다면 대칭적인가? 추이성인가? 일관적인가?를 자문해보자
