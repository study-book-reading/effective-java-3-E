## 아이템 08 finalizer와 cleaner 사용을 피하라

### 개요

자원 반납을 목적으로 finalizer와 cleaner를 사용하지 말자

finalizer

- 예측할 수 없고, 상황에 따라 위험할 수 있어 보통 불필요하다.

cleaner

- finalizer보다는 덜 위험하지만, 여전히 예측할 수 없고 느리다. 이 역시 보통은 불필요하다.

GC의 대상은 되지만 바로 수거해 가지 않거나, 수거를 하지 못할 수도 있다.

### 단점

#### 즉시 수행된다는 보장이 없다.

- finalizer와 cleaner는 즉시 수행된다는 보장이 없다.
- 원하는 시점에 실행하게 하는 작업은 절대 할 수 없다.
    - 파일 닫기 시스템이 동시에 열 수 있는 파일 개수에는 한계가 있는데, finalizer와 cleaner가 언제 실행될지 몰라 파일을 계속 열어 둔다면 새로운 파일을 열지 못하게 되어 프로그램이 실패할 수
      있다.
- 낮은 우선 순위의 **finalizer 스레드**가 **다른 애플리케이션 스레드보다 우선순위가 낮을 시 문제**가 발생할 수 있다.
    - 객체 수천 개가 finalizer 대기열에서 회수되기만을 기다리다 프로그램이 죽어버릴 수 있다.
    - cleaner는 자신을 수행할 스레드를 제어할 수 있다는 면에서 조금 낫지만, 여전히 백그라운드에서 **가비지 컬렉터에게 제어되기** 때문에 **즉각 수행되리라는 보장은 없다.**

#### 얼마나 빠르게 실행될지 알 수 없다.

- 얼마나 신속히 수행될지는 가비지 컬렉터에게 기도해야 한다.
- 전적으로 가비지 컬렉터 알고리즘에 달려있으며 가바지 컬렉터마다 다르다.
- 현재 프로그래머가 테스트한 JVM에선 완벽하게 동작하여도 고객의 시스템에선 재앙을 일으킬 수 있다.

#### 우선순위가 낮다.

- 불행히도 Finalizer 쓰레드는 우선순위가 낮아서 실행될 기회를 얻지 못 할 수도 있다. (언제 실행될지 모른다.)
- Finalizer안에 작업이 있고 그 작업을 쓰레드가 처리하지 못 하고 대기하다가, 해당 인스턴스가 GC되지 않고 쌓이다가 결국 OutOfMemoryError를 발생할 수 있다.

#### 아예 실행되지 않을 수도 있다.

- 수행 여부조차 보장하지 않기 때문에 상태를 영구적으로 수정하는 작업에는 절대 finalizer와 cleaner에 의존해서는 안된다.
- 데이터베이스 같은 공유 자원의 영구 락 해제를 finalizer와 cleaner에게 맡겨 놓으면 분산 시스템 전체가 서서히 멈출 것이다.

```java
@Override
protected void finalize()throws Throwable{
        // 락걸린 대상을 이 객체가 소멸될 때 같이 락을 해제하면 되겠다! -> X
        }
```

#### 현혹되지 말자.

- System.gc 나 System.runFinalization 메서드에 현혹되지 말자.
- finalizer와 cleaner가 실행될 가능성을 높여줄 순 있으나 보장하지 않는다.
- 이를 보장해주겠다는 메서드가 2개 등장했지만 심각한 결함때문에 몇년간 deprecated 상태다
    - java11에선 삭제되었다.

![image](https://github.com/study-book-reading/effective-java-3-E/assets/52563841/dbb592e5-45ea-4bf0-9118-d7f97da0b845)

- finalizer 동작 중 발생한 예외는 무시되며, 처리할 작업이 남아있더라도 그 순간 종료된다.
- 잡지 못한 예외 때문에 해당 객체는 자칫 마루리가 덜 된 상태로 남을 수 있다.
- 보통의 경우 잡지 못한 예외가 스레드를 중단시키고 스택 추적 내역을 출력한다. 하지만 같은 일이 finalizer에서 발생한다면 경고조차 출력하지 않는다.
- 그나마 cleaner를 사용하는 라이브러리는 자신의 쓰레드를 통제하기 때문에 이러한 문제가 발생하지 않는다

#### 성능에도 문제가 있다.

- AutoCloseable 객체를 생성하고 try-with-resources로 자원을 닫아서 가비지컬렉터가 수거하기까지 12ns가 걸렸다면 
finalizer를 사용한 객체를 생성하고 파괴하니 550ns가 걸렸다. (50배)
- finalizer가 가비지 컬렉터의 효율을 떨어지게 한다. 하지만 잠시 후 알아볼 안전망 형태로만 사용하면 66ns가 걸린다. 안전망의 대가로 50배에서 5배로 성능차이를 낼 수 있다.

#### finalizer 공격에 노출될 수 있다.

- 생성이나 직렬화 과정에서 예외가 발생하면 생성되다 만 객체에서 악의적인 하위 클래스의 finalizer가 수행될 수 있게 된다.

```java
public class KakaoBank {

    private int money;

    public KakaoBank(final int money) {
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
```

```java
public class BankAttack extends KakaoBank {

    public BankAttack(final int money) {
        super(money);
    }

    @Override
    protected void finalize() throws Throwable {
        this.transfer(1000000000); // 악의적인 코드
    }
}
```

```java
public class Main {
    public static void main(final String[] args) throws InterruptedException {
        KakaoBank bank = null;
        try {
            bank = new BankAttack(500);
            bank.transfer(1000);
        } catch (Exception e) {
            System.out.println("예외 터짐");
        }
        System.gc(); // gc가 일어나면서 finalize가 실행이 되면서 transfer()가 실행됨
        sleep(3000);
    }
}
```

- 방어 방법
    - 부모 클래스(KakaoBank)에서 final 클래스로 만들거나
    - finalizer() 메소드를 오버라이딩 한 다음 final을 붙여서 하위 클래스에서 오버라이딩 할 수 없도록 막는다

### finalizer와 cleaner 보다 더 좋은 해결책

1. 반납할 자원이 있는 클래스는 AutoClosable을 구현하고 클라이언트에서 close()를 호출한다.

 ```java
public class Sample implements AutoCloseable {
    public class AutoClosableIsGood implements Closeable {

        private BufferedReader reader;

        public AutoClosableIsGood(String path) {
            try {
                this.reader = new BufferedReader(new FileReader(path));
            } catch (FileNotFoundException e) {
                throw new IllegalArgumentException(path);
            }
        }

        //자원 해제
        @Override
        public void close() {
            try {
                reader.close();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```

2. try-with-resources를 사용한다.

 ```java
 public class App {

    public static void main(String[] args) {
        try (AutoClosableIsGood good = new AutoClosableIsGood("")) {
            // TODO 자원 반납 처리가 됨.

        }
    }
}
 ```

### cleaner, finalizer는 언제 사용할까?

1. 안전망 역할로 자원을 반납하려 할 때
2. 네이티브 자원을 정리할 때

- **자원의 소유자가 close 메서드를 호출하지 않는 것에 대비**
    - cleaner, finalizer가 즉시 호출될것이란 보장은 없지만, 클라이언트가 하지 않은 자원 회수를 늦게라도 해주는 것이 아예 안하는 것보다 낫다.
    - 하지만 이런 안전망 역할로 finalizer를 작성할 때 그만한 값어치가 있는지 신중히 고려해야 한다.
    - 자바에서는 안전망 역할의 `finalizer`를 제공한다.
        - `FileInputStream`, `FileOutputStream`, `ThreadPoolExecutor`가 대표적이다.

2. **네이티브 자원 정리**
    - cleaner와 finalizer를 적절히 활용하는 두 번째 예는 네이티브 피어와 연결된 객체이다.
    - 네이티브 피어란 일반 자바 객체가 네이티브 메서드를 통해 기능을 위임한 네이티브 객체를 말한다.
    - 그 결과 자바 피어를 회수할 때 네이티브 객체까지 회수하지 못 한다.

- 성능 저하를 감당할 수 없거나 자원을 즉시 회수해야 한다면 close 메서드를 사용해야 한다.

### cleaner 사용하기

```java
// cleaner를 안전망으로 활용하는 AutoCloseable 클래스
public class Room implements AutoCloseable {
    // 객체에 접근할 수 없을 때 리소스를 정리하거나 메모리를 해제하는 데 사용되는 유틸리티 클래스
    private static final Cleaner cleaner = Cleaner.create();

    // Room을 참조하지 말것!!! 순환 참조
    private static class State implements Runnable {
        int numJunkPiles;

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // **colse가 호출되거나, GC가 Room을 수거해갈 때 run() 호출**
        @Override
        public void run() {
            System.out.println("Room Clean");
            numJunkPiles = 0;
        }
    }

    // 방의 상태. cleanable과 공유한다.
    private final State state;

    // cleanable 객체. 수거 대상이 되면 방을 청소한다.
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state); // Room 객체에 접근할 수 없게 되면 state의 run() 메서드 호출하여 정리를 수행하기 위한 등록
    }

    @Override
    public void close() {
        cleanable.clean(); // run 메소드 실행
    }
}
```

- State 인스턴스가 Room 인스턴스를 참조할 경우 순환참조가 발생하고 가비지 컬렉터가 Room을 회수해갈 기회가 오지 않는다.
- State가 static인 이유도 바깥 객체를 참조하지 않기 위해서이다.
  - 정적이 아닌 중첩 클래스는 자동으로 바깥 객체의 참조를 갖는다.
  - static을 붙이지 않으며 내부 중첩 참조가 일어나서 외부 클래스가 gc 대상이여도 리소스가 사라지지 않는다.
  - 람다 또한 마찬가지이다. 람다 안에서 바깥 클래스의 참조가 존재하면 외부 클래스가 gc 대상이여도 리소스가 사라지지 않는다.
- 위 코드는 안전망을 만들었을 뿐이다. 클라이언트가 try-with-resources 블록으로 감쌌다면 방 청소를 정상적으로 출력한다.

```java
public static void main(final String[] args){
    try(Room myRoom=new Room(8)){
        System.out.println("방 쓰레기 생성~~");
    }
}
```
- AutoClosable를 사용하여 Rome이 구현하고 있는 close 메소드를 통해서 자원을 정리 

하지만 아래와 같은 코드는 방 청소 완료라는 메시지를 항상 기대할 수는 없다.

```java
public static void main(final String[]args){
        new Room(8);
        System.out.println("방 쓰레기 생성~~");

        // 단, 가비지 컬렉러를 강제로 호출하는 이런 방식에 의존해서는 절대 안 된다!
        // System.gc();
}
```
- `System.gc();`를 호출하여 가비지 컬렉터를 강제로 실행시켜 자원을 회수할 수 있다.
  - gc를 수행할때 Cleaner의 Queue에 들어간 작업이 실행된다.
  - Cleaner은 try-with-resource를 사용하지 않았을 때 자원을 회수하려는 일종의 안전망 역할인 셈이다.

## 참고

- [인프런 이펙티브 자바1](https://www.inflearn.com/course/%EC%9D%B4%ED%8E%99%ED%8B%B0%EB%B8%8C-%EC%9E%90%EB%B0%94-1)
- [2022-effective-java](https://github.com/woowacourse-study/2022-effective-java)
- [Item 8. finalizer와 cleaner 사용을 피하라](https://righteous-galette-116.notion.site/Item-8-finalizer-cleaner-560de54aedf34859be2eef2ab520da8e)
