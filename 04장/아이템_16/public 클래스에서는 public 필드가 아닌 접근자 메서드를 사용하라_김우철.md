## 아이템 16. public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라

### 퇴보한 클래스
```java
class Point {
	public int x;
	public int y;

	public Point(int x, int y) {
		this.x = x;
		this.y = y;
	}
}
```
- 필드에 직접 접근할 수 있으니 캡슐화의 이점을 제공하지 못함
- API를 수정하지 않고는 내부 표현을 바꿀 수 없고 getter, setter를 이용하면 표현 변경 가능
- 불변식을 보장할 수 없으며 외부에서 필드에 접근할 때 부수 작업을 수행할 수 없다.
- 1차원적인 접근만 가능하고, 추가 로직을 삽입할 수 없다.

### default 및 private 중첩 클래스
- default 클래스 혹은 private 중첩 클래스라면 데이터 필드를 노출한다 해도 하등의 문제가 없다.
- default 클래스는 패키지 바깥에서는 접근 불가능하다.
  - 패키지 내부에서만 조작 가능
- Point클래스는 private이기 때문에 TopPoint안에서만 접근 가능하다.
```java
// private 중첩 클래스 예제
public class TopPoint {

    private static class Point {
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

//Collecitions
```

### 자바 플랫폼 라이브러리 Bad Case
- Point와 Dimension 클래스
  - public 클래스의 필드를 직접 노출 시킨 사례
  - 불변성을 보장할 수 없고 심각한 성능 문제
```java
public class Dimension extends Test {
    public int width;
    public int height;

		// Dimension 클래스의 필드는 가변으로 설계되어 getSize를 호출하는 모든 곳에서 
		// 방어적 복사를 위해 인스턴스를 새로 생성해야만 한다.
    // -> 필드값이 public이라 바뀌면 안되기 때문에 getSize() -> size()가 호출될 때 마다
    //    Dimension 객체를 새로 생성
		// JVM 힙 영역 메모리 계속 쌓이는 문제
		
		/**
     * getSize() 메서드를 호출 할 때마다
     * Dimension 인스턴스를 생성하고 있다.
     */
    public Dimension getSize() {
        return size();
    }


    // @Deprecated : 빌드할때 경고 메시지를 보여줌
    @Deprecated
    public Dimension size() { // 방어적 복사를 위해 인스턴스를 새로 생성
        return new Dimension(width, height);
    }
```

### 불변 필드를 노출한 public 클래스
- public 클래스의 필드가 불변이라도 좋지 않다.
- 불변식은 보장할 수 있지만.. 
  - API를 변경하지 않고는 표현 방식을 바꿀 수 없고
  - 필드를 읽을 때 부수 작업을 수행할 수 없다는 단점은 여전하다
```java
//각 인스턴스가 유효한 시간을 표현함을 보장 (final)
public final class Time {
    private static final int HOURS_PER_DAY = 24;
    private static final int MINUTES_PER_HOURS = 60;

    public final int hour;
    public final int minute;

    public Time(int hour, int minute) {
        if (hour < 0 || hour >= HOURS_PER_DAY)
            throw new IllegalArgumentException("시간: " + hour);
        if (minute < 0 || minute >= MINUTES_PER_HOURS)
            throw new IllegalArgumentException(": " + minute);
        this.hour = hour;
        this.minute = minute;
    }
    //..생략
}
```

```java
// 고정 되서 사용하는 변수 값은 static 으로 정적 메모리에 할당해두고 사용하는 것을 권장
public class messageConstants{
    public static final String MESSAGE_START = "시작";
    public static final String MESSAGE_END = "끝";
}

// 외부 클래스에서 messageConstants.MESSAGE_START 로 접근 가능.
```

### 요약
> public 클래스는 절대 가변 필드를 직접 노출해서는 안된다. 
하지만 package-private 클래스나 private 중첩 클래스에서는 종종 (불변이든 가변이든) 필드를 노출하는 편이 나을 때도 있다.
