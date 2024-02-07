# equals는 일반 규약을 지켜 재정의하라

---

# 왠만하면 재정의 하지말자

equals 메서드는 재정의하기 쉬워 보이지만 곳곳에 함정이 도사리고 있다. 잘 사용하지 않으면 예상과 다른 결과를 초래한다.

문제를 회피하는 가장 쉬운 길은 아예 재정의하지 않는 것이다. 그냥 두면 그 클래스의 인스턴스는 오직 자기 자신과만 같게 된다. 그러니 아래에 열거한 상황 중 하나에 해당되지 않는다면 재정의하지 않는 것이 최선이다.

- **각 인스턴스가 본질적으로 고유하다.**
    - Thread가 좋은 예로, Object의 equals 메서드는 이러한 클래스에 딱맞게 구현되었다.
- **인스턴스의 논리적 동치성을 검사할 일이 없다.**
- **상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는다.**
    - Set 구현체는 AbstractSet이 구현한 equals를 상속받아 쓰고, List 구현체들은 AbsractList로부터 상속받아 그대로 쓴다.
- **클래스가 private 이거나 package-private 이고 equals 메서드를 호출할 일이 없다.**

equals가 실수라도 호출되는 것을 막고 싶다면 다음처럼 구현하자

```java
@Override
public boolean equals(Object obj){
	throw new AssertionError();
}
```

---

# 언제 재정의 해야할까

**객체 식별성(물리적으로 같은 주소를 가리키는지)가 아니라 논리적 동치성을 확인해야 하는데, 상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의되지 않았을 때다.**

주로 값 클래스(eg. Integer, String)들이 여기 해당한다.  값 클래스라 해도, 값이 같은 인스턴스가 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제 클래스(아이템 1)이라면 equals를 재정의하지 않아도 된다. Enum도 여기에 해당한다.

---

# equals 재정의 일반 규약

- **반사성:** null이 아닌 모든 참조 값 x에 대해, x.equals(x)는 true
- **대칭성:** null이 아닌 모든 참조 값 x,y에 대해, x.equals(y)가 true 이면 y.equals(x)도 true다
- **추이성:** null이 아닌 모든 참조 값 x,y,z에 대해, x.equals(y)는 true 이고 y.equals(z)도 true면, x.equals(z)도 true다.
- **일관성:** null이 아닌 모든 참조 값 x,y에 대해, x.equals(y)를 반복해서 호출하면 항상 true를 반환하거나 항상 false를 반환한다.
- **null-아님:** null이 아닌 모든 참조 값 x에 대해, x.equals(null)는 false다.

### 반사성

```java
public class CaseInsensitiveString {
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }

    @Override
    public boolean equals(Object obj) {
        if (obj instanceof CaseInsensitiveString) {
            return s.equalsIgnoreCase(((CaseInsensitiveString) obj).s);
        }

        if (obj instanceof String) {
            return s.equalsIgnoreCase((String) obj);
        }

        return false;
    }

    public static void main(String[] args) {
        CaseInsensitiveString cis = new CaseInsensitiveString("Hello");
        String s = "hello";

        System.out.println(cis.equals(s)); // true
        System.out.println(s.equals(cis)); // false

				List<CaseInsensitiveString> cises = new ArrayList<>();
        cises.add(cis);
        System.out.println(cises.contains(s)); // false
    }
}
```

**eqauls 규약을 어기면 그 객체를 사용하는 다른 객체들이 어떻게 반응할 지 알 수 없다.**

CaseInsensitiveString.equals()를 String과 연동하는 대신, 다음처럼 간단한 모습으로 바꾸자.

```java
@Override
    public boolean equals(Object obj) {
        return obj instanceof CaseInsensitiveString
                && ((CaseInsensitiveString) obj).s.equalsIgnoreCase(s);
    }
```

### 추이성

```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Point)) {
            return false;
        }

        Point p = (Point) obj;

        return p.x == x && p.y == y;
    }
}
```

이 클래스를 확장해서 점에 색상을 더한다.

```java
public class ColorPoint extends Point {
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof ColorPoint)) {
            return false;
        }

        return super.equals(obj) && ((ColorPoint) obj).color == color;
    }
		
		// 대칭성 위배
		public static void main(String[] args) {
        Point p = new Point(1, 2);
        ColorPoint cp = new ColorPoint(1, 2, Color.RED);

        System.out.println(p.equals(cp)); // true
        System.out.println(cp.equals(p)); // false
    }
}

enum Color {
    RED, ORANGE, YELLOW, GREEN, BLUE, INDIGO, VIOLET
}
```

아래와 같이 일반 점이면 색상을 무시하여 대칭성을 지킬 수 있다. 하지만 추이성이 깨진다.

```java
    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Point)) {
            return false;
        }

        if (!(obj instanceof ColorPoint)) {
            return obj.equals(this);
        }

        return super.equals(obj) && ((ColorPoint) obj).color == color;
    }

    public static void main(String[] args) {
        Point p = new Point(1, 2);
        ColorPoint cp1 = new ColorPoint(1, 2, Color.RED);
        ColorPoint cp2 = new ColorPoint(1, 2, Color.BLUE);
        
        System.out.println(cp1.equals(p)); // true
        System.out.println(p.equals(cp2)); // true
        System.out.println(cp1.equals(cp2)); // false
    }
```

이 방식은 무한 재귀에 빠질 가능성도 있다. Point의 또 다른 하위 클래스로 SmellPoint를 만들고, equals 와 동일한 방식으로 구현하면 cp.equals(sm)을 호출하면 StackOverflowError 가 발생한다.

이러한 현상은 모든 객체 지향 언어의 동치 관계에서 나타나는 근본적인 문제다.

**구체 클래스를 확장해 새로운 값을 추가하면서 equals 규약을 만족 시킬 방법은 존재하지 않는다.**

> **리스코프 치환 원칙**
>

"프로그램의 속성(정확성, 수행하는 작업 등)이 하위 타입의 객체에 의해 변경되지 않도록, 하위 클래스의 인스턴스를 상위 클래스의 인스턴스로 바꿀 수 있어야 한다."

- 이 원칙에 따르면, 어떤 클래스의 어떤 인스턴스(객체)도 그 클래스의 부모(상위) 클래스의 인스턴스로 대체될 수 있어야 하며, 이로 인해 프로그램의 기능에 어떠한 장애도 발생해서는 안된다
- 즉, 하위 클래스는 상위 클래스의 동작을 확장할 수 있지만, 기본적인 동작을 변경해서는 안된다.
- 어떤 타입에 있어 중요한 속성이라면 그 하위 타입에서도 마찬가지로 중요하다.
- 따라서 그 타입의 모든 메서드가 하위 타입에서도 똑같이 잘 작동해야 한다.
- 예를 들어, 모든 "새"가 "날 수 있다"고 가정하는 프로그램에 "타조" 클래스가 "새"의 하위 클래스로 추가된다고 할 때, 타조는 실제로 날 수 없기 때문에 "새" 클래스의 기본 가정과 맞지 않다. 이 경우, 타조 클래스는 리스코프 치환 원칙을 위반하는 것이다.

### 상속 대신 합성을 사용해라

```java
public class ColorPoint {

		// 상속 대신 합성
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = color;
    }

    public Point asPoint() {
        return point;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof ColorPoint)) {
            return false;
        }
        ColorPoint cp = (ColorPoint) obj;
        return cp.point.equals(point) && cp.color.equals(color);
    }

		public static void main(String[] args) {
        Point p = new Point(1, 2);
        ColorPoint cp1 = new ColorPoint(1, 2, Color.RED);
        ColorPoint cp2 = new ColorPoint(1, 2, Color.BLUE);

        System.out.println(cp1.equals(p)); // false
        System.out.println(p.equals(cp2)); // false
        System.out.println(cp1.equals(cp2)); // false
    }
}

enum Color {
    RED, ORANGE, YELLOW, GREEN, BLUE, INDIGO, VIOLET
}
```

**Point를 상속하는 대신 Point를 필드로 두고, 뷰 메서드를 public으로 추가하여 우회할 수 있다.**

추상 클래스의 하위 클래스에서라면 equals 규약을 지키면서도 값을 추가할 수 있다.

```java
public abstract class AbstractPoint {
    private final int x;
    private final int y;

    public AbstractPoint(int x, int y) {
        this.x = x;
        this.y = y;
    }

    protected boolean canEqual(Object obj) {
        return obj instanceof AbstractPoint;
    }

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof AbstractPoint)) {
            return false;
        }
        AbstractPoint other = (AbstractPoint) obj;
        return other.canEqual(this) && this.x == other.x && this.y == other.y;
    }

    // 추상 메서드: 하위 클래스에서 구현
    protected abstract boolean isEquivalent(AbstractPoint other);
}
```

```java
public class ConcretePoint extends AbstractPoint {
    private final Color color;

    public ConcretePoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }

    // 실제 동등성 비교를 수행
    @Override
    protected boolean isEquivalent(AbstractPoint other) {
        if (!(other instanceof ConcretePoint)) {
            return false;
        }
        ConcretePoint obj = (ConcretePoint) other;
        return this.color.equals(obj.color);
    }

    @Override
    protected boolean canEqual(Object obj) {
        return obj instanceof ColorPoint;
    }

    @Override
    public boolean equals(Object obj) {
        return super.equals(obj) && isEquivalent((AbstractPoint) obj);
    }

    public static void main(String[] args) {
        Point p = new Point(1, 2);
        ConcretePoint cp1 = new ConcretePoint(1, 2, Color.RED);
        ConcretePoint cp2 = new ConcretePoint(1, 2, Color.BLUE);

        System.out.println(cp1.equals(p)); // false
        System.out.println(p.equals(cp2)); // false
        System.out.println(cp1.equals(cp2)); // false
    }
}
```

구체 클래스에서는 isEquivalent 메서드를 오버라이드하여 색상을 비교한다.

이런 방식으로 추가적인 값을 포함시킬 수 있다.

### 일관성

**두 객체가 같다면(어노 하나 혹은 두 객체 모두가 수정되지 않는 한) 앞으로도 영원히 같아야함을 의미한다.**

**클래스가 불변이든 가변이든 equals 판단에 신뢰할 수 없는 자원이 끼어들게 해서는 안된다.**

ex) java.net.URL의 equals는 주어진 URL과 매핑된 호스트 IP 주소를 이용해 비교한다. 호스트 이름을 IP 주소를 바꾸려면 네트워크를 통해야 하는데, 그 결과가 항상 같다고 보장할 수 없다.

**이런 문제를 해결하기 위해서 equals는 항시 메모리에 존재하는 객체만을 사용한 결정적 계산만 수행해야 한다.**

---

# ⭐️ equals 메서드 구현 정리

### 구현 로직

1. == 연산자를 사용해 입력이 자기 자신의 참조인지 확인한다.
2. instanceof 연산자로 입력이 올바른 타입인지 확인한다.
3. 입력을 올바른 타입으로 형변환한다.
4. 입력 객체와 자기 자신의 대응되는 ‘핵심' 필드들이 모두 일치하는지 모두 확인한다.

### 타입별 로직

- float과 double을 제외한 기본 타입 필드는 == 연산 비교
- 참조 타입 필드는 각각의 equals 메서드로 비교
- float과 double 필드는 특수한 부동소수 값 등 때문에 각각 정적 메서드인 `Float.compare(float, float)`와 `Double.compare(double,double)`로 비교한다.
- 배열 필드는 원소 각각을 위의 지침대로 비교한다. 배열의 모든 원소가 핵심 필드라면 Arrays.equals() 메서드 들 중 하나를 사용한다.
- 때론 Null도 정상 값으로 취급하는 참조 타입 필드가 있는데, 이런 필드는 Objects.equals(Object, Object)로 비교해 NullPointerException 발생을 예방하자.

### 어떤 필드 먼저 비교해야 하나

- 다를 가능성이 더 크거나 비교하는 비용이 싼 필드 먼저 비교하자
- 동기화용 락 필드 같이 객체의 논리적 상태와 관련 없는 필드는 비교하면 안된다.

### 신경써야 할 것

- equals를 재정의할 땐 hashCode도 반드시 재정의하자.
- 너무 복잡하게 해결하려 들지 말자. 필드들의 동치성만 검사해도 규약을 어렵지 않게 지킬 수 있다.
- Object 외의 타입을 매개변수로 받지 말자.

**equals를 구현했다면 세가지만 자문해보자. 대칭적인가? 추이성이 있는가? 일관적인가?**

자문에서 끝내지 말고 단위 테스트도 작성해 돌려보자.

```java
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        rangeCheck(areaCode, 999, "area code");
        rangeCheck(prefix, 999, "prefix");
        rangeCheck(lineNum, 9999, "line num");
        this.areaCode = (short) areaCode;
        this.prefix = (short) prefix;
        this.lineNum = (short) lineNum;
    }

    private static void rangeCheck(int arg, int max, String name) {
        if (arg < 0 || arg > max) {
            throw new IllegalArgumentException(name + ": " + arg);
        }
    }

    @Override
    public boolean equals(Object obj) {
        if (obj == this) {
            return true;
        }
        if (!(obj instanceof PhoneNumber)) {
            return false;
        }
        PhoneNumber pn = (PhoneNumber) obj;
        return pn.lineNum == lineNum && pn.prefix == prefix && pn.areaCode == areaCode;
    }
}
```

---

# 요약

- 꼭 필요한 경우가 아니면 equals를 재정의하지 말자
- 많은 경우에 Objects의 equals가 원하는 비교를 정확히 수행해준다.
- 재정의를 해야한다면 핵심 필드를 빠짐없이, 다섯가지 규약을 지켜가며 비교한다.