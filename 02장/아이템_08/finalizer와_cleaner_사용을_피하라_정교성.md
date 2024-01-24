## 아이템8 finalizer와 cleaner 사용을 피하라

자바의 객체 소멸자를 어떻게 사용해야 하는가?

자바의 객체 소멸자
1. finalizer(자바9 이후 사용자제 처리됨)
2. cleaner

---
### 1. finalizer란?

finalizer는 자바에서 객체 소멸 전에 특정 작업을 수행할 수 있게 하는 메커니즘을 말한다. finalize메서드를 통해 해당 메커니즘이 수행된다. finalize메서드란 Object클래스의 메서드로서 어떤 객체가 파괴되기 전 해당 객체의 활동을 클린업(해당 객체가 사용한 모든 자원 해제) 해주는 메서드이다.  

#### (1) finalize메서드를 언제 사용되나?

- 가비지 컬렉션이 JVM에 의해 처리될 때, finalize 메서드를 통해 객체가 파괴되기 전에 자원을 해제한다.  
- finalize메서드는 GC에 의해 한 번만 호출됨. finalize메서드에서 예외가 발생하거나, 개체가 finalize에서 자체적으로 부활하는 경우, 가비지 컬렉터는 finalize메서드를 다시 호출하지 않음.
- 자원 해제를 온전히 finalizer에 의존하면 안됨. finalize메서드가 호출되는지 안되는지를 보장할 수 없음.
- 자바에선 자원해제에 두 가지 다른 방법을 제공함. close메서드나 destroy메서드가 있고, 이들은 자동으로 수행하지 않기 때문에 직접 명시해줘야 함.
- 

#### (2) 사용예시

```java
import java.lang.*;

public class Demo {

  protected void finalize() throws Throwable {
    try {
      System.out.println("Inside finalize method of Demo Class.");
    } 
    catch (Throwable e) {
      throw e;
    } 
    finally {
      System.out.println("Calling finalize method of the Object class");

      // Object클래스의 finalize()를 호출
      super.finalize();
    }
  }

  public static void main(String[] args) throws Throwable {
    demo d = new demo();

    d.finalize();
  }
}
```

자바는 두 가지 객체 소멸자를 제공한다. 그중 finalizer는 예측할 수 없고, 상황에 따라 위험할 수 있어 일반적으로 불필요하다.  
오동작, 낮은 성능, 이식성 문제의 원인이 되기도 함.  

#### (3) Finalizers의 단점

- finalized 메서드의 사용은 대개 낮은 성능을 보여준다.
- 가비지 컬렉터를 즉시 호출하지 않고, 단지 가비지 컬렉션 과정을 수행하도록 JVM에 힌트를 주는 수준이다.
- JVM은 이미 가비지 컬렉션 자동 수행기능을 갖고 있고, 언제 GC를 호출해야하는지 더 잘 안다.
- finalize메서드의 실행여부를 보증할 수 없다.
- finalize메서드는 JVM에 의해 객체당 한 번만 호출된다. 심지어 해당 객체가 복원되면 finalize는 호출되지 않는다.
- 시스템이 열 수 있는 파일 개수에 한계가 있는데 finalizer 실행을 게을리 해서 파일을 계속 열 경우 새로운 파일을 열지 못함.
- 파일을 사용하고 닫지 못했기 때문에 외부에서 해당 파일을 건드리지 못할 경우가 생김.(교착상태)
- finalizer 스레드는 다른 애플리케이션 스레드보다 우선순위가 낮아서 실행될 기회가 밀려서 OutOfMemoryError를 발생시키는 경우도 있음.

##### (3)-1. 왜 성능상 문제가 생기는가?

GC의 특별대우

finalize() 메서드를 오버라이드 하면 해당 객체는 가비지 컬렉터가 특별하게 처리함. 가비지 수집 즉시 회수되지 않고 종료화 대상으로 먼저 등록됨. finalize()를 오버라이드 하지 않는 객체보다 수명이 한 사이클 김.

1. finalize()를 오버라이드 한 객체는 큐로 이동
2. 별도의 finalize 스레드가 해당 큐를 비우면서 각 객체마다 정의된 finalize()를 호출
3. finalize()가 종료되면 해당 객체는 다음 GC사이클에 진짜 수집될 준비를 마침

가비지 컬렉터가 finalize메서드를 오버라이드한 객체와 아닌 객체를 구분할 수 있는 이유는 오버라이드한 객체의 생성자 바디가 성공적으로 반환되는 시점에 해당 객체를 종료화 가능한 객체 목록에 등록하는 식으로 JVM에 구현되어 있기 때문임.

이러한 객체의 수명이 연장되는 문제 때문에 리소스가 낭비되는 시간이 더 길어지면서 성능에 문제가 될 수 있음.

#### (4) Finalizer의 대안 2가지

자바에서 객체에 접근할 수 없을 때 자원을 해제하는 다른 효율적이고 유연한 방법 2가지

1. Cleaner 클래스

자바9에서 finalizer가 사용자제 처리되고, cleaner라는 새로운 클래스가 추가됐다. 어떤 객체가 가비지 컬렉션 대상이 되면, Cleaner클래스의 객체가 해당 클래스의 자원을 해제한다. 

2. PhantomReference

객체가 마무리 되는 동안 객체를 복원할 수 없음을 보장함.

  - **자바의 참조 4가지**
    1. 강력한 참조(Strong)
    2. 소프트 참조(Soft)
    3. 약한 참조(Weak)
    4. 팬텀 참조(Phantom)

강력한 참조를 제외한 나머지 세 참조클래스는 get()메서드를 통해 참조된 객체를 반환하고, clear()메서드를 통해 개체에 대한 참조를 제거함.

- 팬텀참조 특징
1. 가장 약한 참조
2. 개체에 대한 다른 참조가 남아있지 않은 경우에만 작동

#### (4)-1. 사용예시

- cleaner 예시
```java
public class Room implements AutoCloseable {
  private static final Cleaner cleaner = Cleaner.create();

  //청소가 필요한 자원. 절대 Room을 참조해선 안된다.(순환 참조 발생하여 가비지 컬렉터가 회수하지 않음)
  private static class State implements Runnable {
    int numJunkPiles = numJunkPiles;

    State(int numJunkPiles) {
      this.numJunkPiles = numJunkPiles;
    }

    //close메서드나 cleaner가 호출한다.
    @Override
    public void run() {
      System.out.println("방 청소");
      numJunkPiles = 0;
    }
  }

  //방의 상태. cleanable과 공유한다.
  private final State state;

  //cleanable 객체. 수거 대상이 되면 방을 청소한다.
  private final Cleaner.Cleanable cleanable;

  public Room(int numJunkPiles) {
    state = new State(numJunkPiles);
    cleanable = cleaner.register(this, state);  //register(Object, Runnable) 
  }

  @Override
  public void close() {
    cleanable.clean();
  }
}
```

- PhantomReference 예시
```java
public class Main{
  public static void main(String[] args) {
      System.out.println("start time = " + LocalDateTime.now());

      LargeObject largeObject = new LargeObject();

      //ReferenceQueue 생성
      ReferenceQueue<LargeObject> referenceQueue = new ReferenceQueue<>();

      //PhantomReference생성
      PhantomReference<LargeObject> phantomReference = new PhantomReference<>(largeObject,referenceQueue);

      //largeObject 참조 해제
      largeObject = null;

      System.gc();

      Reference<? extends LargeObject> referenceFromQueue;
      while ((referenceFromQueue = referenceQueue.poll()) != null){
          if(referenceFromQueue == phantomReference) {
              //여기에서 해당 객체의 리소스를 해제하거나, 반납하는 작업을 수행
              System.out.println("LargeObject 객체가 가비지 컬렉션 되었습니다.");
              System.out.println("collected time = " + LocalDateTime.now());
          }
      }
  }
  static class LargeObject {
      private final byte[] data = new byte[1024* 1024 * 100];
  }
}

//출력결과
//start time 만 출력되는 경우가 있음
//Thread.sleep()을 적당히 추가해주면, collected time이 출력됨.
```

자바9에서는 finalizer를 사용자제(deprecated) API로 지정하고 cleaner를 그 대안으로 소개했다.(하지만 자바 라이브러리에서도 finalizer를 여전히 사용한다)  
cleaner는 finalizer보다는 덜 위험하지만, 여전히 예측할 수 없고, 느리고, 일반적으로 불필요하다.  

- C++ 비 메모리 자원 회수 : 파괴자(destructor)
- Java 비 메모리 자원 회수 : 가비지 컬렉터, try-with-resource, try-finally

자바 언어 명세는 어떤 스레드가 finalizer를 수행해야할지 명시하지 않으니 문제를 예방하는 방법은 사용하지 않는 방법 뿐이다.  
cleaner는 자신을 수행할 스레드를 제어할 수 있다는 면에서 조금 낫다.  
하지만 여전히 백그라운드에서 수행되며 가비지 컬렉터의 통제하에 있으니 즉각 수행되리라는 보장은 없다.  

System.gc나 System.runFinalization 메서드에 현혹되지 말라. finalizer와 cleaner가 실행될 가능성을 높여줄 수는 있으나, 보장해주진 않는다.  

사실 이를 보장해주겠다는 메서드가 2개 있었다. 바로 System.runFinalizerOnExit와 그 쌍둥이인 Runtime.runFinalizersOnExit이다.  
하지만 이 두 메서드는 **심각한 결함**있다.

>#### runFinalizersOnExit의 심각한 결함이란?
>runFinalizersOnExit메서드를 호출하면 다른 스레드가 해당 개체를 동시에 조작하는 동안 활성 개체에 대해 종료자가 호출되어 비정상적인 동작이나 교착상태가 발생할 수 있음.


---
#### 결론

finalizer는 이제 deprecated 됐으니 사용하지 말라.  
cleaner는 안전장치용으로 사용할 필요가 있을 때만 사용하자.


[참조]
- https://madplay.github.io/post/java-finalize
- https://jaeyeong951.medium.com/finalize-%EC%9D%80%ED%87%B4%EC%8B%9D-4a52fb855910
- https://www.itworld.co.kr/news/224419
- https://adnoctum.tistory.com/567
- https://www.scaler.com/topics/finalize-method-in-java/
- https://bugs.openjdk.org/browse/JDK-8198250
- https://jaime-note.tistory.com/462
- https://codegym.cc/ko/groups/posts/ko.213.javaui-paenteom-chamjo