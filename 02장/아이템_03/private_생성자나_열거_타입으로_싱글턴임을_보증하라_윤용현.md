# private 생성자나 열거 타입으로 싱글턴임을 보증하라


## 싱글톤

### 싱글톤이 안티패턴이라고 불리는 이유

- private 생성자를 갖고 있기 때문에 상속할 수 없다.
- 테스트하기 어려워질 수 있다. 타입을 인터페이스로 정의한 다음 그 인터페이스를 구현해서 만든 싱글톤이 아니라면 싱글톤 인스턴스를 목 객체로 대체할 수 없기 때문이다.
- 객체를 전역 상태를 만들어서 어디서나 참조할 수 있게 되므로 객체 지향적인 설계 원칙에 어긋난다.

<br><br><br>

### public static final 필드 방식으로 싱글톤 구현

```Java
public class Foo {
    public static final Foo INSTANCE = new Foo();

    private Foo() {
        // ...
    }
    
    public void doSomething() {
        // ...
    }
}
```

생성자를 private 으로 감춰두고 유일한 인스턴스에 접근할 수 있는 수단으로 public static 멤버를 마련하는 방법

private 생성자는 public static final 필드인 INSTANCE 를 초기화할 때 딱 한번만 호출된다.

**해당 클래스가 싱글턴임이 API 에 명백히 드러난다.**

<br><br><br>

### 정적 팩터리 메서드 방식으로 싱글톤 구현

```Java
public class Foo {
    private static final Foo INSTANCE = new Foo();

    private Foo() {
        // ...
    }

    public static Foo getInstance() {
        return INSTANCE;
    }
    
    public void doSomething() {
        // ...
    }
}
```

마음이 바뀌면 API를 바꾸지 않고도 싱글톤이 아니게 변경할 수 있다.

```Java
public class Foo {
    private static final Foo INSTANCE = new Foo();

    private Foo() {
        // ...
    }

    // 싱글톤 인스턴스 반환
    public static Foo getInstance() {
        return INSTANCE;
    }

    public void doSomething() {
        // ...
    }
}

// 나중에 변경된 버전
public class Foo {
    private Foo() {
        // ...
    }

    // 새로운 인스턴스를 생성하여 반환
    public static Foo getInstance() {
        return new Foo();
    }

    public void doSomething() {
        // ...
    }
}
```

정적 팩토리를 제네릭 싱글톤 팩토리로 만들 수 있다.

```Java
public class GenericSingletonFactory {
    private static final IdentityFunction<Object> INSTANCE = new IdentityFunction<>();

    private static class IdentityFunction<T> implements UnaryOperator<T> {
        @Override
        public T apply(T t) {
            return t;
        }
    }

    @SuppressWarnings("unchecked")
    public static <T> UnaryOperator<T> generate() {
        return (UnaryOperator<T>) INSTANCE;
    }
}
```

정적 팩토리의 메서드 참조를 공급자(supplier)로 사용할 수 있다.

```Java
public class Foo {
    private static final Foo INSTANCE = new Foo();

    private Foo() {
        // ...
    }

    public static Foo getInstance() {
        return INSTANCE;
    }

    public void doSomething() {
        // ...
    }
}

public class Main {
    public static void main(String[] args) {
        Supplier<Foo> fooSupplier = Foo::getInstance;
        Foo foo = fooSupplier.get();
        foo.doSomething();
    }
}
```

<br><br><br>

### 3. 열거 타입 방식의 싱글턴

```Java
public enum Foo {
    INSTANCE;
    
    public void doSomething() {
        // ...
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Foo.INSTANCE.doSomething();
    }
}
```

- INSTANCE 인스턴스가 하나만 존재함을 보장한다.
- private 생성자를 제공하지 않아도 되고, 리플렉션 공격에도 안전하다.
- Enum 싱글턴은 자동으로 직렬화를 지원하며, 역직렬화 과정에서도 동일한 인스턴스를 유지한다. 추가적인 직렬화 작업이 필요로 하지 않다.
- 단, 만들려는 싱글턴이 `Enum` 외의 클래스를 상속해야 한다면 이 방법은 사용할 수 없다.

>**대부분 상황에서는 원소가 하나뿐인 열거 타입이 싱글턴을 만드는 가장 좋은 방법이다.**

<br><br><br>

## 직렬화/역직렬화

### 직렬화

- 객체를 바이트 스트림으로 변환하는 것
- 객체를 파일로 저장하거나 통신 할 때에는, 참조형식의 데이터를 사용할 수 없다.
    - 직렬화 과정을 통해 참조 형식의 데이터를 싹 끌어모아서 값 형식 데이터로 모두 변환시킨다.
- 직렬화된 객체는 그 객체를 구성하는 데이터도 직렬화 가능한 데이터여야 한다.
    - 직렬화된 객체를 역직렬화 할 때, 직렬화된 객체를 구성하는 데이터도 역직렬화 되어야 하기 때문이다.

<br><br>

### 싱글톤 직렬화

`public` 싱글톤 방식이던 정적 메서드 방식이던 싱글톤 클래스를 직렬화하려면 단순히 `Serializable` 을 구현한다고 선언하는 것만으로는 부족하다.

싱글톤 패턴에서의 인스턴스는 `static` 필드로 관리하는데, `static` 필드는 직렬화 대상에서 제외된다.

문제는 역직렬화 과정에서 발생한다.

역직렬화 시, `static` 필드에 저장된 싱글톤 인스턴스는 영향을 받지 않는다. 대신, 역직렬화 과정은 항상 새로운 인스턴스를 만들어낸다.
만약 싱글톤 클래스에 비 `transient` 인스턴스 필드가 존재한다면, 역직렬화된 인스턴스는 원래의 싱글톤 인스턴스와는 다른 객체가 된다.

**이러한 문제를 해결하기 위해서는 모든 인스턴스 필드를 일시적(`transient`)이라고 선언하여 싱글톤 인스턴스 필드를 직렬화에서 제외하고, `readResolve` 메서드를 제공하여 역직렬화시 싱글톤 인스턴스의 일관성을 유지해야한다.**

**그렇지 않으면 직렬화된 인스턴스를 역직렬화할 때마다 새로운 인스턴스가 만들어진다.**


#### readResolve 메서드

```Java
public class Singleton implements Serializable {
    private static final long serialVersionUID = 1L;
    public static final Singleton INSTANCE = new Singleton();

    private Singleton() {
        // 싱글톤 생성자
    }

    private Object readResolve() {
        // 싱글톤 인스턴스를 반환하여 새로운 인스턴스 생성 방지
        return INSTANCE;
    }
}
```

역직렬화 과정이 완료된 후 호출되어, 싱글톤의 원래 인스턴스를 반환한다

이 메서드를 통해 역직렬화로 생성된 새로운 인스턴스를 무시하고, 기존의 싱글톤 인스턴스를 유지한다

