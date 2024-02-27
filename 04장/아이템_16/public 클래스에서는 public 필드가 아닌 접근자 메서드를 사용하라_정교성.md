## 아이템16. public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라

인스턴스 필드들을 모아놓는 퇴보한 클래스

```java
class Point {
  public double x;
  public double y;
}
```

**이러한 클래스(필드 노출)의 단점**
- API수정없이 내부 표현 바꿀 수 없음
- 불변식 보장할 수 없음
  - 클라이언트가 직접 필드 수정가능
- 외부에서 필드에 접근시 부수 작업을 수행할 수 없음
  - 필드에 바로 접근이 가능하여 필드 사용에 부가 로직이 필요한 경우 처리 불가 

**대안으로 접근자, 변경자 메서드를 사용하여 캡슐화**

필드를 모두 private으로 바꾸고, public접근자(getter)를 추가

```java
class Point {
  private double x;
  private double y;

  public Point(double x, double y) {
    this.x = x;
    this.y = y;
  }

  public double getX() { return this.x; }
  public double getY() { return this.y; }

  public void setX(double x) { this.x = x; }
  public void setY(double y) { this.y = y; }
}
```

- 내부 표현 변경 가능
  - getter/setter 등 메서드 사용하여 내부 표현을 변경할 수 있음


public 클래스가 아닌 package-private 클래스 혹은 private 중첩 클래스라면 데이터 필드를 노출해도 문제 없음  

- package-private : 같은 패키지에서 접근 가능
- private 중첩 클래스 : 해당 클래스를 감싸고 있는 외부 클래스만 접근 가능

클라이언트 코드가 이 클래스의 내부 표현에 묶이긴 하지만, 클라이언트도 해당 패키지 안에서만 동작하기 때문에 API 변경에 대한 부담이 없음.

```java
public class TopPoint {

    private static class Point {   //private 중첩 클래스
        public double x;
        public double y;
    }

    public Point getPoint() {
        Point point = new Point();  // TopPoint 외부에선
        point.x = 3.5;              // Point 클래스 내부 조작이
        point.y = 4.5;              // 불가능 하다.
        return point;
    }
}
```

**public 클래스의 필드를 직접 노출한 예시**

- java.awt.package의 Point와 Dimension클래스

```java
public class Point extends Point2D implements java.io.Serializable {
    public int x;    //public으로 필드 외부에 노출
    public int y;

    public double getX() {
        return x;
    }
    public double getY() {
        return y;
    }

    public void move(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

**그렇다면 public 클래스의 필드가 불변이라면?**

해결되지 못한 문제
- API 변경하지 않고 표현 방식 바꿀 수 없음
- 필드 접근 시 부수작업 수행 불가

public 클래스의 불변 필드 노출 
```java
public final class Time {
  private static final int HOURS_PER_DAY = 24;
  private static final int MINUTES_PER_HOUR = 60;

  public final int hour;
  public final int minute;

  public Time(int hour, int minute) {
    if (hour < 0 || hour >= HOURS_PER_DAY)
      throw new IllegalArgumentException("시간: " + hour);
    if (minute < 0 || minute >= MINUTE_PER_HOUR)
      throw new IllegalArgumentException("분: " + minute);
    this.hour = hour;
    this.minute = minute;
  }
}
```

---
## 결론

public 클래스는 가변 필드를 노출하면 안됨.  
불변 필드라면 노출이 가능하지만, 몇몇 이점을 놓칠 수 있음.  
예외로 package-private 클래스나 private 중첩 클래스는 종종(불변이든 가변이든) 필드 노출이 편할 때도 있음.  

---
[참고] 
- https://it-mesung.tistory.com/191
- https://hyeon9mak.github.io/Effective-Java-item16/