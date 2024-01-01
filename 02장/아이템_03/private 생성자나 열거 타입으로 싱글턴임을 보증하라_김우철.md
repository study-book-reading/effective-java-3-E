## 아이템 03 private 생성자나 열거 타입으로 싱글턴임을 보증하라

### 개요

싱글톤 패턴이란 오직 한 개의 클래스 인스턴스만을 갖도록 보장한다.


### 싱글턴을 만드는 방식

#### 1. public static final 필드 방식의 싱글턴

```java
public class UniqueSuperCar {
    public static final UniqueSuperCar UNIQUE_SUPER_CAR = new UniqueSuperCar();
    
    private UniqueSuperCar() { }
    
    public void run() {
        System.out.println("run~~~");
    }
}
```

```java
public class App {
    public static void main(String[] args) {
        UniqueSuperCar uniqueSuperCar = UniqueSuperCar.UNIQUE_SUPER_CAR;
        uniqueSuperCar.run();
    }
}

// 출력 결과
// run~~~
```
private 생성자는 public static final 필드인 UniqueSuperCar.UNIQUE_SUPER_CAR 를 초기화할 때 단 한번만 호출된다  
static 키워드가 메소드 영역에 올라갈 때 UniqueSuperCar 객체가 생성된다

장점
- 간결하다
- 싱글톤임을 API에 명확하게 드러낼 수 있다

단점
1. 인터페이스를 구현하지 않은 싱글톤을 사용하는 클라이언트 코드는 테스트하기가 어려워진다. (싱글톤의 공통 단점)
  - 테스트 할 클라이언트 메서드가 해당 싱글톤을 사용하는 중이라면, 테스트를 할 때마다 해당 싱글톤을 같이 사용하게 된다. 
    **테스트의 주 목적이 클라이언트 메서드 안에서 싱글톤 외의 코드들을 테스트하는 것일 때, 싱글톤 객체가 굉장히 무겁거나 오래 걸린다면, 
    클라이언트 메서드는 테스트 할 때마다 쓸데없이 그 영향을 받게 될 것이다.**
  - 대책은 싱글톤 클래스가 인터페이스를 구현하는 것으로 구조를 바꾸고, 테스트 할 때만 대역을 해줄 가짜 객체를 만들어서 테스트 하는 것이다.

2. 리플렉션 API를 사용해서 private 생성자를 얼마든지 이용할 수 있다. (1,2번 방법 공통 단점)
  - 생성자가 2번 이상 호출되는 경우엔 예외를 던지는 식으로 방어할 수 있다. 
    하지만 이 대안을 사용하게 된다면, public static final 필드를 사용하는 싱글톤의 장점인 ‘간결함’ 은 사라질 수 있다.

3. 역직렬화 할 때 새로운 인스턴스가 생길 수 있다. (1,2번 방법 공통 단점)
  - 역직렬화 과정에서 생성자가 private임에도 불구하고 새로 객체를 생성한다
  - 해결책은 역직렬화를 할 때 사용되는 readResolve() 메서드를 싱글톤 인스턴스를 반환하게끔 구현해서 싱글톤 클래스에 제공해주면 된다. 

위 단점의 해결책

1. 인터페이스를 만든다.
  - 테스트하기 용이하도록 하기 위함.
2. 생성자를 private으로 설정 후 생성자에 유효성 검증 코드를 넣는다.
  - 리플렉션 방지하기 위함.
3. readResolve() 메서드를 싱글턴 객체를 반환하게끔 구현하는 것이다.
  - 역직렬화로 인한 새로운 객체 생성 방지하기 위함.

단, 위 1,2,3번을 전부 조치한다고 가정하면, ‘간결함’ 이라는 장점은 무조건 사라지게 될 것이다.

#### 2. private 생성자와 정적 팩토리 메서드 이용

```java
public class UniqueSuperCar {
    private static final UniqueSuperCar UNIQUE_SUPER_CAR = new UniqueSuperCar();
    
    private UniqueSuperCar() {
    }
    
    // 정적 팩토리 메서드
    public static UniqueSuperCar getInstance() {
        return UNIQUE_SUPER_CAR;
    }
}
```

```java
public class App {
    public static void main(String[] args) {
        UniqueSuperCar uniqueSuperCar1 = UniqueSuperCar.getInstance();
        uniqueSuperCar1.run();
    
        UniqueSuperCar uniqueSuperCar2 = UniqueSuperCar.getInstance();
        uniqueSuperCar2.run();
    
        System.out.println("같은 객체인가?? : " + (uniqueSuperCar1 == uniqueSuperCar2));
    }
}

// 출력 결과
// 싱글톤 : run~~~
// 싱글톤 : run~~~
// 같은 객체인가?? : true
```

1번 방법과 비교한 추가 장점

1. API(클라이언트 코드)를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있다.
  -  팩토리 메서드를 이용해서 객체를 반환받는 클라이언트의 코드는 변경하지 않고, 팩토리 메서드의 구현부만 변경해서 동작을 바꿀 수 있는 것을 의미

2. 정적 팩토리를 제네릭 싱글턴 팩터리로 만들 수 있다.
  - 클래스를 제네릭 클래스로, 그리고 정적 팩토리를 제네릭 정적 팩토리로 구현했을 때, 제네릭 정적 팩토리는 대입 받을 인스턴스 변수의 제네릭 타입이
    어떤지에 따라 알아서 맞춰서 반환해준다.

```java
// 제네릭 클래스
public class UniqueSuperCar<T> {
    // 제네릭 타입을 모든 타입으로 강제 캐스팅이 가능하게끔 Object로 설정한다.
    private static final UniqueSuperCar<Object> UNIQUE_SUPER_CAR = new UniqueSuperCar<>();
           
    private UniqueSuperCar() {
    }
           
    // 제네릭 메서드
    @SuppressWarnings("unchecked")
    public static <E> UniqueSuperCar<E> getInstance() {
        // 여기서 강제 캐스팅을 해줘서 반환하는 형식으로 구현한다.
        return (UniqueSuperCar<E>) UNIQUE_SUPER_CAR;
    }
           
    public void print(T t) {
        System.out.println(t);
    }
           
    public void run() {
        System.out.println("싱글톤 : run~~~");
    }
}
```

```java
public class App {
    public static void main(String[] args) {
        // 오른쪽의 <String> 생략 가능
        UniqueSuperCar<String> uniqueSuperCar1 = UniqueSuperCar.<String>getInstance();
        uniqueSuperCar1.run();
        uniqueSuperCar1.print("aa");
           
        // 인스턴스를 반환받는 변수의 제네릭 타입을 보고 컴파일러는 어떤 제네릭 타입인지 유추할 수 있다.
        // 그래서 위의 getInstance() 메서드 옆에 있는 제네릭 표기를 생략할 수 있다.
        // 밑의 getInstance() 왼쪽 옆엔 <Integer> 가 생략되어 있는 것이다.
        UniqueSuperCar<Integer> uniqueSuperCar2 = UniqueSuperCar.getInstance();
        uniqueSuperCar2.run();
        uniqueSuperCar2.print(10);
    }
}
       
// 출력 결과
// 싱글톤 : run~~~
// aa
// 싱글톤 : run~~~
// 10
```

3. 정적 팩토리의 메서드 참조를 Supplier 함수로 사용할 수 있다.
  - 인자가 없고 반환값만 존재하는 함수형 인터페이스를 통해 사용 가능하다.
  - Supplier 함수로 사용하고 싶을 땐 장점이 될 수 있다

```java
public class App {
    public static void main(String[] args) {
        // 정적 팩토리 메서드는 이런식으로 Supplier 함수에 메서드 참조로 넣어서 이용할 수도 있다.
        UniqueSuperCar<String> uniqueSuperCar1 = printAll(UniqueSuperCar::getInstance);
        uniqueSuperCar1.print("aa");
           
        UniqueSuperCar<Integer> uniqueSuperCar2 = printAll(UniqueSuperCar::getInstance);
        uniqueSuperCar2.print(10);
    }
           
    public static <T> UniqueSuperCar<T> printAll(Supplier<UniqueSuperCar<T>> supplier) {
        UniqueSuperCar<T> uniqueSuperCar = supplier.get();
        supplier.get().run();
        return uniqueSuperCar;
    }
}
       
// 싱글톤 : run~~~
// aa
// 싱글톤 : run~~~
// 10
```

하지만 이러한 장점들이 필요없는 상황이라면 오히려 final 필드를 이용한 1번째 방식이 더 좋다.

#### 3. 열거 타입(Enum)을 이용 (이펙티브 자바 권장 방법)

장점

1. 모든 싱글톤 형태 중 가장 간결한 형태다.

2. 리플렉션에 안전하다.
 - 지정해준 열거 상수들만 인스턴스로 사용할 수 있다는 enum 클래스의 특징으로 인해 리플렉션이 통하지 않는다.

3. 역직렬화에 안전하다.
  - 직렬화 후 역직렬화로 가져와도 새로운 객체가 아닌 직렬화 했던 객체가 그대로 다시 나오게 된다. 

4. 인터페이스 구현을 통해 원활한 테스트도 가능하다.
  - Enum 클래스는 상속은 받을 수 없지만, 인터페이스 구현은 가능하다.
  - Enum은 결국 열거된 객체들 하나하나가 해당 Enum 클래스를 상속받고 있는 형식이다. 

## 참고

- [2022-effective-java](https://github.com/woowacourse-study/2022-effective-java)
- [private 생성자나 열거 타입으로 싱글톤임을 보증하라](https://tjdtls690.github.io/studycontents/java/2023-02-03-singleton02/)
