## 아이템 03 private 생성자나 열거 타입으로 싱글턴임을 보증하라

싱글턴이란?  
인스턴스를 오직 하나만 생성할 수 있는 클래스  
예시) 함수와 같은 무상태 객체, 설계상 유일해야 하는 시스템 컴포넌트

싱글턴 사용시 주의사항
인터페이스로 정의한 다음 그 인터페이스를 구현해서 만든 싱글턴이 아닌 클래스를 싱글턴으로 만들면 이를 사용하는 클라이언트를 테스트하기가 어려워질 수 있음  
왜? 싱글턴 인스턴스를 가짜(mock)를 사용하여 구현으로 대체할 수 없기 때문

싱글턴을 만드는 방식
1. public static final 필드 방식

```java
public class Singleton {
    public static final Singleton INSTANCE = new Singleton();
    private Singleton() {   //private 생성자는 INSTANCE필드를 초기화 할 때 한 번만 호출됨
        ...
    }   
    //public이나 protected생성자가 없으므로 인스턴스가 전체 시스템에서 하나뿐임을 보장
}
```

(예외) 권한이 있는 클라이언트는 리플렉션 API(아이템65)인 AccesibleObject.setAccessible을 사용해 private 생성자를 호출할 수 있음.
이러한 공격을 방어하려면 생성자를 수정하여 두 번째 객체 생성시 예외 발생.

---

2. 정적 팩터리 메서드를 public static 멤버로 제공

```java
public class Singleton {
    public static final Singleton INSTANCE = new Singleton();
    private Singleton() { 
        ...
    }   
    public static Singleton getInstance() { return INSTANCE; }
}
```

Singleton.getInstance는 항상 같은 객체의 참조를 반환하므로 제2의 인스턴스 생성 불가
(위의 예외 동일 발생)

장점
1. API를 바꾸지 않고도 싱글턴에서 싱글턴이 아니도록 변경 가능
2. 정적 팩터리를 제네릭 싱글턴 팩터리로 만들 수 있음(아이템30)
3. 정적 팩터리의 메서드 참조를 공급자로 사용할 수 있음(아이템 43, 44)

위 장점들이 굳이 필요치 않을 경우 public 필드 방식 사용이 좋음

싱글턴 클래스를 직렬화(12장 참조)할 경우  
Serializable 구현 선언 외에 모든 인스턴스 필드를 일시적(transient)으로 선언, readResolve 메서드를 제공(아이템 89)  
이렇게 하지 않으면 직렬화된 인스턴스를 역직렬화할 때마다 새로운 인스턴스가 만들어짐.

```java
//싱글턴 보장 readResolve 메서드
private Object readResolve() {
    //싱글턴 필드를 반환하고, 새로운 인스턴스는 가비지 컬렉터가 회수
    return INSTANCE;
}
```

3. 원소가 하나인 열거 타입을 선언 - 가장 바람직한 방법

```java
public enum Singleton {
    INSTANCE;
}
```

장점
1. 간결함
2. 직렬화에 추가적인 작업 필요 없음
3. 복잡한 직렬화 상황이나 리플렉션 공격에도 제2의 인스턴스 생성 방지

주의, 만들려는 싱글턴이 Enum 외의 클래스를 상속해야 한다면 사용 불가(열거 타입이 다른 인터페이스를 구현하도록 선언할 수는 있음)

