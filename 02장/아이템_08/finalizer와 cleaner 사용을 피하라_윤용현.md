# Finalizer와 cleaner 사용을 피하라

---

## 단점

### 실행 보장 X

**finalizer와 cleaner는 즉시 실행된다는 보장이 없다. 자바에서는 finalizer나 cleaner의 수행 시점뿐 아니라 수행 여부조차 보장하지 않는다.**

`System.gc`나 `System.runFinalization` 메서드를 호출해서 finalizer나 cleaner를 실행될 가능성을 높여줄 수는 있으나, 보장해주진 않는다.

- 파일 닫기와 같은 작업을 finalizer나 cleaner에 맡기면, 시스템이 finalizer나 cleaner를 실행을 게을리해서 파일을 계속 열어두게 되면 새로운 파일을 열지 못할 수도 있다. (시스템이 동시에 열 수 있는 파일 개수에 한계가 있기 때문이다)
- finalizer와 cleaner를 얼마나 신속히 수행할지는 전적으로 GC 알고리즘에 달려있다.
    - GC가 바쁘다면 finalizer나 cleaner를 실행하는데 시간을 들이지 않을 수도 있다.
- finalizer 스레드는 다른 어플리케이션 스레드보다 우선 순위가 낮아서 실행될 기회를 제대로 얻지 못할 수도 있다. cleaner는 자신을 수행할 스레드를 제어할 수 있다는 점에서 finalizer보다는 낫지만, 그래도 일반적인 스레드보다는 우선 순위가 낮다.
- finalizer 동작 중 발생한 예외는 무시되며, 처리할 작업이 남았더라도 그 순간 종료된다.
    - 잡지 못한 예외 때문에 해당 객체는 자칫 마무리가 덜 된 상태로 남을 수 있다.

<br>

### 성능 문제

- `finalizer`를 사용한 객체를 생성하고 파괴하는 것은 GC에 비해 상당히 비싸다.
- `AutoCloseable`을 구현하여 생성된 객체를 GC가 수거하는 것에 비해 50배 느리다.

<br>

### 보안 문제

- 생성자나 직렬화 과정에서 예외가 발생하면 `finalizer`가 수행되는데, 이 `finalizer`를 악의적으로 오버라이딩한 하위클래스의 `finalizer`가 수행될 수 있다.
    - 심지어 이 `finalizer`를 정적 필드에 할당하면 GC에 의해 수거되지도 않는다.
- **`final`이 아닌 클래스를 finalizer 공격으로부터 방어하려면, 아무 일도 하지 않는 finalize 메서드를 만들고 final로 선언해야 한다.**

```java
public class Bank {

    private int money;

    public Bank(final int money) {
        if (money < 1000) {
            throw new RuntimeException("1000원 이하로 생성이 불가능해요.");
        }
        this.money = money;
    }

    void transfer(final int money) {
        this.money -= money;
        System.out.println(MessageFormat.format("{0}원 입금 완료!!", money));
    }
}

public class BankAttack extends Bank {

    public BankAttack(final int money) {
        super(money);
    }

    @Override
    protected void finalize() throws Throwable {
        this.transfer(1000000000);
    }
}


public class BankTest {
    
    public static void main(final String[] args) throws InterruptedException {
        Bank bank = null;
        try {
            bank = new BankAttack(500);
            bank.transfer(1000);
        } catch (Exception e) {
            System.out.println("예외 터짐");
        }
        System.gc();
        sleep(3000);
    }
    
}
```

실행 결과
```
예외 터짐
1,000,000,000원 입금 완료!!
```

<br><br><br>

## 대안

- **파일이나 스레드 등 종료해야 할 자원을 담고 있는 객체의 클래스에서 `AutoCloseable`을 구현하고, 클라이언트에서 인스턴스를 다 쓰고 나면 `close` 메서드를 호출하면 된다.**
    - **제대로 종료되도록 하려면 `try-with-resources`를 사용하면 된다.**
- 각 인스턴스는 자신이 닫혔는지를 추적하는 것이 좋다
    - 즉, `close` 메서드에서 이 객체는 더 이상 유효하지 않음을 필드에 기록하고, 다른 메서드는 이 필드를 검사해서 객체가 닫힌 후에 호출되었다면 `IllegalStateException`을 던진다.

<br><br><br>

## finalizer와 cleaner 언제 사용해야 하나?

- 자원 소유자가 close 메서드를 호출하지 않는 것에 대비한 안전망 역할
    - cleaner나 finalizer가 즉시 호출되리라는 보장은 없지만, 클라이언트가 하지 않은 자원 회수를 늦게라도 해주는 것이 더 낫다
- 네티이브 피어(일반 자바 객체가 네이티브 메서드를 통해 기능을 위임한 네이티브 객체) 연결 객체 해제
    - GC는 네이티브 피어가 자바 객체가 아니니 그 존재를 알지 못한다.
    - cleaner나 finalizer가 나서서 네이티브 피어를 해제해주어야 한다.
    - 단, 성능 저하를 감당할 수 있고 네이티브 피어가 심각한 자원을 가지고 있지 않을 때만 해당한다.

<br><br><br>

## Cleaner 사용 예제

cleaner는 finalizer와 달리 클래스의 public api에 나타나지 않고 순전히 내부 구현에 달려있다.

```java
import java.lang.ref.Cleaner;

public class Room implements AutoCloseable {

    private static final Cleaner cleaner = Cleaner.create();

    private static class State implements Runnable {
        int numJunkPiles; // 방 안의 쓰레기 수

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // close 메서드나 cleaner가 호출한다.
        @Override
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // 방의 상태. cleanable과 공유한다.
    private final State state;

    // cleanable 객체. 수거 대상이 되면 방을 청소한다.
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override
    public void close() {
        cleanable.clean();
    }
}
```

`State` 의 `run` 메서드 호출되는 상황은 둘 중 하나다.

- Room close 메서드를 호출할 때
    - close 메서드에서는 cleanable의 clean 메서드를 호출하고, clean 메서드는 State의 run 메서드를 호출한다.
- GC가 Room을 회수할 때까지 클라이언트가 close를 호출하지 않는다면, cleaner가 Room을 회수하기 전에 State의 run 메서드를 호출한다.

**State 인스턴스는 절대로 Room 인스턴스를 참조해서는 안 된다. 그렇게 하면 순환참조가 생겨 GC가 Room 인스턴스가 회수되지 않을 것이다. State 인스턴스는 오직 cleaner가 호출하는 run 메서드만 사용한다.**
**State가 정적 중첩 클래스인 이유가 여기에 있다. 정적이 아닌 중첩 클래스는 자동으로 바깥 객체의 참조를 가지게 되기 때문이다.** 이와 비슷하게 람다 역시 바깥 객체의 참조를 찾기 쉬우니 사용하지 않는 것이 좋다.

위 코드는 cleaner를 사용하여 안정망으로만 사용한 것이다. 클라이언트가 Room 생성을 `try-with-resources`로 감싸도록 하면 자동 청소는 전혀 필요하지 않게 된다.

```java
public class Main {
    public static void main(String[] args) {
        try (Room room = new Room(7)) {
            System.out.println("안녕");
        }
    }
}
```

<br>

하지만 아래와 같은 코드는 방 청소 완료라는 메시지를 항상 기대할 수는 없다.

```java
public static void main(final String[] args) {
  new Room(8);
  System.out.println("방 쓰레기 생성~~");
}
```

<br><br><br>

## 정리

clenear(자바 8까지는 finalizer)는 안전망 역할이나 중요하지 않는 네이티브 자원 회수용으로만 사용하자.
물론 이런 경우라도 불확실성과 성능 저하에 주의해야 한다.

**그냥 finalizer나 cleaner를 사용하지 않는 것이 낫다.**

<br>

## Ref

- https://www.youtube.com/watch?v=6kNzL1bl1kI
- https://github.com/woowacourse-study/2022-effective-java/blob/main/02%EC%9E%A5/%EC%95%84%EC%9D%B4%ED%85%9C_08/finalizer%EC%99%80_cleaner_%EC%82%AC%EC%9A%A9%EC%9D%84%20%ED%94%BC%ED%95%98%EB%9D%BC.md
- https://javabom.tistory.com/74
- https://github.com/Java-Bom/ReadingRecord/issues/7