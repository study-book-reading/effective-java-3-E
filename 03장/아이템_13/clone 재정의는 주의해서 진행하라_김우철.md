## 아이템 13. clone 재정의는 주의해서 진행하라

### Cloneable 인터페이스
- Cloneable은 복제해도 되는 클래스임을 명시하는 용도의 믹스인 인터페이스이지만 의도한 목적을 제대로 이루지는 못했다.
- 빈 껍데기의 마커 인터페이스
```java
public interface Cloneable {
}
```

#### clone 메서드의 문제점
- clone 메서드가 선언된 곳이 Cloneable 인터페이스가 아닌 Object이다.
  - 일반적인 인터페이스이 용도에 맞지 않다.
  - 상위 클래스인 Object에 정의된 protected 메서드의 동작 방식을 결정하기 때문이다.
- clone 메서드의 접근 제한자가 protected 이다.
  - public 이여야 외부에서 사용할 수 있다.
  - 그래서 Cloneable을 구현하는 것만으로는 외부 객체에서 clone 메서드를 호출할 수 없다.
    - Override 한다음 접근 제어자를 public으로 변경해 줘야 한다.
- Object 클래스의 clone 메서드
```java
@IntrinsicCandidate
protected native Object clone() throws CloneNotSupportedException;
```

- 여러 문제점에도 불구하고 Cloneable 방식은 널리 쓰이고 있어서 잘 알아둬야 한다
  - clone 메서드를 잘 동작하게끔 구현하는 방법
  - 언제 구현해야 하는지
  - 가능한 다른 선택지

### Cloneable 인터페이스 용도
- Object의 protected 메서드인 clone의 동작 방식을 결정 한다.
- Cloneable을 구현한 클래스의 인스턴스에서 clone을 호출하면 그 객체의 필드들을 하나하나 복사한 객체를 반환한다.
- 그렇지 않은 클래스의 인스턴스에서 호출하면 CloneNotSupportedException을 던진다. 

### clone 메서드의 허술한 일반 규약
- ‘복사’의 정확한 뜻은 그 객체를 구현한 클래스에 따라 다를 수 있다.
  - 일반적인 의도
    1. 첫번째
      - 참
    ```
    x.clone() != x
    ```
    - clone은 원본과는 다른 인스턴스 여야 한다.

    2. 두번째
    - 참
    ```java
    x.clone().getClass() == x.getClass()
    ```
    - clone 결과물의 인스턴스 타입은 원본의 클래스 인스턴스 타입과 같다.

    3. 세번째
    ```java
    // 동치성
    x.clone().equals(x)
    ```
    - 같을 수도 있고 다를 수도 있음(애매모호함)
      - 달라야 하는 경우
        - 고유값 역할을 하는 필드가 있는 경우 clone 메서드 내에서 복사 후 고유 값을 수정해줘야 함 

### clone 사용시 주의사항
- clone으로 만들어지는 인스턴스는 생성자를 사용하지 않는다.
- clone() 메서드가 super.clone()이 아닌, 생성자를 호출해 얻은 인스턴스를 반환해도 컴파일러는 동일한 인스턴스로 본다.
- 하지만, 이 클래스의 하위 클래스에서 super.clone을 호출한다면 잘못된 클래스의 객체가 만들어져 하위 클래스의 clone이 깨지게 된다. 
  - clone 메서드에서 super.clone()이 아닌 new 키워드를 사용해서 호출하면 안된다.
```java
public class Item implements Cloneable {

    private String name;

    /**
     * 이렇게 구현하면 하위 클래스의 clone()이 깨질 수 있다. p78
     * @return
     */
    @Override
    public Item clone() {
        Item item = new Item();
        item.name = this.name;
        return item;
    }
}
```
- 상위 클래스에서 clone을 new로 구현

```java
public class SubItem extends Item implements Cloneable {

  private String name;

  @Override
  public SubItem clone() {
    return (SubItem) super.clone();
  }

  public static void main(String[] args) {
    SubItem item = new SubItem();
    SubItem clone = item.clone();

    System.out.println(clone != item);
    System.out.println(clone.getClass() == item.getClass());
    System.out.println(clone.equals(item));
  }
}
```
- 서브 클래스는 clone을 일반적인 규약에 맞춰 구현
- `SubItem clone = item.clone();` clone 호출시 에러 발생
  - super.clone()은 Item을 리턴
  - `(SubItem) Item;`와 같은 코드가 되기 때문에 캐스팅 익셉션 발생
    - 상위 타입은 구체적인 타입으로 변환 불가능
- 서브 클래스에서 clone을 재정의 안하면
  - 역시나 캐스팅 익셉션이 발생
  - 재정의를 하지 않아도 **Object에 구현되어 있는 clone() 메서드가 사용되면서 super.clone()을 호출**
  - 그럼 또 다시 item에 있는 clone이 호출됨
  - 따라서 상위 클래스의 클론에서 new 키워드를 사용하면 안됨
  - 적절한 구현 방법
  ```java
  public Item clone(){
    Item result = null;
    try {
        result = (Item) super.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
  }
  ```
  - 컴파일 환경으로 봤을 때 Item 타입이지만 서브 클래스에서 호출하는 순간 서브 타입으로 나오게 된다.
  - 호출된 clone() 메서드가 속한 객체 타입의 인스턴스가 생성되고 반환된다.
  - super.clone()이 어디서 호출되느냐에 따라서 실제 나오는 인스턴스 타입이 달라짐
  - 그래서 절대로 클론을 구현할 때 생성자를 호출해서 만들면 안되고 항상 super.clone()으로 호출해야 한다.
- 하위 클래스에서 super.clone()을 호출한다면 상황에 따라 잘못된 클래스의 객체가 만들어질 수 있다.
  - 하위 클래스에 추가된 상태나 필드값이 있을 경우, clone() 메서드를 재정의 해야 한다. 
  - 상위 클래스의 clone() 값을 그대로 반환하면 안되고, 하위 클래스 타입으로 변환한 뒤 적절한 값을 세팅 후 반환 해야 한다.
- clone()을 재정의한 클래스가 final 클래스인 경우에는 걱정할 필요가 없다.
- 모든 필드가 기본 타입이거나 불변 객체를 참조하는 경우 이 객체는 완벽한 상태이므로 clone()을 제공하지 않는 것이 좋다.
  - 쓸데 없는 복사를 지양한다는 관점에서 불변 클래스는 굳이 clone 메서드를 제공하지 않는 것이 좋다.
  - 불변 객체여도 clone()을 제공한다면 Cloneable을 implements 받고 clone() 메서드 안에서 super.clone()을 사용하면 된다.
```java
public class Coffee implements Cloneable {
    @Override
    public synchronized Coffee clone() { // synchronized: 스레드 안정성 
        Coffee clone;
        try {
            clone = (Coffee) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
        return clone;
    }
}
```
- 재정의한 메서드의 반환 타입은 상위 클래스의 메서드가 반환하는 타입의 하위 타입일 수 있다.
  - 이를 공변 반환 타입이라고 한다.
- 클라이언트가 형변환 하지 않도록 Cloneable 인터페이스를 제공한다.
- CloneNotSupportedException 는 Checked Exception으로 되어 있어 try-catch 블록으로 묶었다.
  - 잘 설계되었다고 볼 수 없다.
  - 개발자가 할 수 있는게 없는데 체크드 익셉션으로 설계하여 의미 없는 코드만 늘어났다.
  - UnChecked Exception이 되었어야 한다.

### 가변 객체를 참조하는 경우 예시 1
```java
// 코드 13-2 가변 상태를 참조하는 클래스용 clone 메서드
    // TODO stack -> elementsS[0, 1]
    // TODO copy -> elementsC[0, 1]
    // TODO elementsS[0] == elementsC[0]
@Override public Stack clone() {
        try {
            Stack result = (Stack) super.clone();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
```
- clone 메서드가 단순히 super.clone()의 결과를 그대로 받아온다면 어떻게 될까?
  - 반환된 Stack 인스턴스의 size 필드는 올바른 값을 갖겠지만, elements 필드는 원본 Stack 인스턴스와 똑같은 배열을 참조할 것이다.
  - 원본이나 복제본 중 하나를 수정하면 다른 하나도 수정되어 불변식을 해친다.
  - 따라서, 프로그램이 이상하게 동작하거나 NullPointerException이 발생하게 된다.
- clone 메서드는 사실상 생성자와 같은 효과를 낸다
  - 즉, **clone은 원본 객체에 아무런 해를 끼치지 않는 동시에 복제된 객체의 불변식을 보장**해야 한다.

- 가변 객체를 갖는 클래스를 복제 하기 위해서는 내부 정보를 복사해야 한다.
  - 가장 쉬운 방법은 elements 배열의 clone을 재귀적으로 호출하는 것이다.
```java
public class Stack implements Cloneable {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 다 쓴 참조 해제
        return result;
    }

    private void ensureCapacity() {
        if (elements.length == size) {
            elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }

    @Override
    public Stack clone() {
        try {
            Stack clone = (Stack) super.clone();
            clone.elements = elements.clone(); // 얇은 복사 문제
            return clone;
        } catch(CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```
#### 문제점
- 하지만, 이 방법도 문제가 있다.
  - 깊은 복사가 아닌 얇은 복사가 되어 배열 객체 자체의 주소 값은 다르지만, 배열의 각 요소가 참조하는 값들은 동일하다.
  - 프리미티브 타입이면 문제가 없다.
- 위의 스택 예제에서 elements 필드가 final 이였다면 작동하지 않는다.
- Cloneable 아키텍처는 **‘가변 객체를 참조하는 필드는 final로 선언하라’** 는 일반 용법과 충돌한다.
- 복제할 수 있는 클래스를 만들기 위해 일부 필드에서 final 한정자를 제거해야 할 수도 있다는 말이다.
- 스택 깊은 복사
```java
@Override public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = new Object[elements.length];
        for (int i = 0; i < elements.length; i++) {
            // 여기서 elements[i].clone()은 각 요소의 깊은 복사를 가정합니다.
            // 이는 elements의 각 요소가 깊은 복사를 지원해야 한다는 가정하에 작성된 코드입니다.
            result.elements[i] = elements[i].clone(); // 이 부분은 실제 구현에 따라 달라질 수 있습니다.
        }
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

### 가변 객체를 참조하는 경우 예시 2
```java
public class HashTable implements Cloneable {

    private Entry[] buckets = new Entry[10];

    private static class Entry {
        final Object key;
        Object value;
        Entry next;

        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
```

```java
    @Override
    public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = buckets.clone();
            return result;
        } catch(CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
```
- 해시테이블용 clone 메서드
  - Stack 처럼 단순히 버킷 배열의 clone을 재귀적으로 호출
- 복제본은 자신만의 버킷 배열을 갖지만, 이 배열은 **원본과 같은 연결 리스트를 참조**하여 원본과 복제본 모두 예기치 않게 동작할 가능성이 생긴다.
- 이를 해결하기 위해서는 각 버킷을 구성하는 연결 리스트를 복사해야 한다.

#### 문제점
```java
        public Entry deepCopy() {
            return new Entry(key, value, next == null ? null : next.deepCopy());
        }
```

```java
/**
     * TODO hasTable -> entryH[],
     * TODO copy -> entryC[]
     * TODO entryH[0] != entryC[0]
     *
     * @return
     */
    @Override
    public HashTable clone() {
        HashTable result = null;
        try {
            result = (HashTable)super.clone();
            result.buckets = new Entry[this.buckets.length];

            for (int i = 0 ; i < this.buckets.length; i++) {
                if (buckets[i] != null) {
                    result.buckets[i] = this.buckets[i].deepCopy(); // p83, deep copy
                }
            }
            return result;
        } catch (CloneNotSupportedException e) {
            throw  new AssertionError();
        }
    }
```
- clone 메서드
  - 적절한 크기의 새로운 버킷 배열 할당
  - 원래의 버킷 배열을 순회하며 비지 않은 각 버킷에 대해 깊은 복사를 수행
- Entry의 deepCopy()는 자신을 재귀적으로 호출
  - 좋지 않은 방법
  - 재귀 호출은 리스트의 원소 수 만큼 스택 프레임을 소비하기 때문에 스택 오버플로우를 일으킬 위험이 있다.
    - 노드가 많이 연결된 링크드 리스트라면 쌓이는 stack이 넘칠 수 있다.
  - deepCopy를 재귀 호출 대신에 반복문을 사용하여 순회하는 방향으로 수정해야 한다.
```java
public Entry deepCopy() {
            Entry result = new Entry(key, value, next);
            for (Entry p = result ; p.next != null ; p = p.next) {
                p.next = new Entry(p.next.key, p.next.value, p.next.next);
            }
            return result;
        }
```

### 가변 객체를 복제 하는 방법
- 초기 객체 생성시에만 super.clone()을 호출하고 고수준 API를 사용해서 객체를 복사하는 방법
  - 고수준 PAI란 hashMap의 get, put 같은 메서드들 
- 이 방법은 저수준에서 바로 처리할 때보다는 느리다

#### 상속용 클래스에 Cloneable 인터페이스 사용을 권장하지 않는다.
- 해당 클래스를 확장 하려는 프로그래머에게 많은 부담을 주기 때문이다.
- 상속을 의도한 클래스(추상 클래스)에 Cloneable 인터페이스를 사용하지 않는게 좋다
- 확장하려는 프로그래머에게 많은 짐을 떠안겨준다.
  - clone을 올바르게 구현하려면 어떻게 해야 하지?
- 상위 클래스에서 Cloneable을 사용해야 한다면 직접 구현해서 하위 클래스에서 구현 못하게끔 막는 것도 좋은 방법이다.

#### clone() 메서드 내에서 상태 값을 수정하는 코드 작성 주의사항
- 생성자에서는 재정의될 수 있는 메서드를 호출하지 않아야 하는 것처럼 clone() 메서드도 마찬가지이다.
  - 객체를 만드는 과정에서 재정의 가능한 메서드를 호출하는거 자체가 객체 생성이 깨질 수 있는 위험기 있다. 

#### clone() 메서드 재정의 시 주의사항
- clone() 메서드는 CloneNotSupportedException을 던진다고 선언되어 있지만 재정의한 메서드는 수정해야 한다.
- public clone 메서드에서는 throws 절을 없애야 한다.
- 검사 예외를 비검사 예외로 수정해야 그 메서드를 사용하기에 편리하다.

#### 동기화가 필요한 프로세스에서 Cloneable 구현

- Cloneable을 구현한 스레드 안전 클래스를 작성할 때는 clone 메서드 역시 적절히 동기화해줘야 한다.
  - synchronized 사용
- super.clone() 호출 외에 다른 할 일이 없더라도 clone을 재정의하고 동기화 해줘야 한다.

### 정리

- Cloneable 인터페이스를 구현하는 모든 클래스는 clone을 재정의 한다.
- 접근 제한자는 public 으로 수정한다.
- 반환 타입은 클래스 자신으로 수정 한다.
- clone() 메서드는 가장 먼저 super.clone()을 호출한 후 필요한 필드를 적절하게 수정한다.
  - 객체의 내부에 숨어 있는 모든 가변 객체를 복사하고, 복제본이 가진 객체 참조 모두가 복사된 객체를 가리키도록 한다.
  - 내부 복사는 주로 clone을 재귀적으로 호출해 구현하지만, 이 방식이 항상 최선은 아니다.
- 기본 타입 필드와 불변 객체 참조만 갖는 클래스인 경우 어떤 필드도 수정할 필요가 없다.
  - 단 일련번호나 고유 ID는 비록 기본 타입이나 불변일지라도 수정해줘야 한다.

### Clone 대안

- Cloneable을 이미 구현한 클래스를 확장한다면 어쩔 수 없이 cloen이 잘 작동하도록 구현해야 한다.
- 그렇지 않은 상황에서는 복사 생성자와 복사 팩터리라는 더 나은 객체 복사 방식을 제공할 수 있다.
  - 복사 생성자란 단순히 자신과 같은 클래스의 인스턴스를 인수로 받는 생성자를 말한다
    ```java
    // 복사 생성자
    public PhoneNumber(PhoneNumber phoneNumber) {
        this(phoneNumber.areaCode, phoneNumber.prefix, phoneNumber.lineNum);
    }
    ```
    
    ```java
    public static Yum newInstance(Yum yum) { ... }; // 복사 팩터리
    ```

- 해당 클래스가 구현한 ‘인터페이스’ 타입의 인스턴스를 인수로 받을 수 있다.
  - HashSet 객체 s를 TreeSet 타입으로 복제할 수 있다.
  - clone 으로는 불가능한 걸 변환 생성자로는 간단히 new TreeSet<>(s)할 수 있다.
  ```java
  public TreeSet(Collection<? extends E> c) {
    //..
  }
  ```

### Clone 단점 정리

- 생성자를 쓰지 않는 방식인 위험한 객체 생성 메커니즘
- 엉성하게 문서화된 규약
- 정상적인 final 필드 용법과도 충돌
- 불필요한 검사 예외를 던짐
- 형변환도 필요함

### 조쉬아 블로그 clone 인터뷰 내용
- https://www.artima.com/articles/josh-bloch-on-design
