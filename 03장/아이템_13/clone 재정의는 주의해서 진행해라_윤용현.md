# clone 재정의는 주의해서 진행해라

---

# Cloneable 인터페이스

- Cloneable 는 복제해도 되는 클래스임을 명시하는 용도의 인터페이스이다.
- 하지만, 의도한 목적을 제대로 이루지 못했다
    - clone 메서드가 선언된 곳은 Cloneable이 아닌 Object이고, 그마저도 protected 이다
- **Cloneable 인터페이스는 Object의 protected 메서드인 clone의 동작 방식을 결정한다.**
    - Cloneable을 구현한 클래스의 인스턴스에서 clone을 호출하면 그 객체의 필드들을 하나하나 복사한 객체를 반환하며, 그렇지 않은 클래스의 인스턴스에서 호출하면 CloneNotSupportedException을 던진다.
    - 이는 인터페이스가 구체 클래스의 동작 방식을 변경한 것이니 따라하면 안된다.
- **실무에서 Cloneable을 구현한 클래스는 clone 메서드를 public으로 제공하며, 사용자는 당연히 복제가 제대로 이뤄지리라 기대한다.**
    - 이 기대를 만족시키기 위해 그 클래스와 모든 상위 클래스는 복잡하고, 강제할 수 없고, 허술하게 기술된 프로토콜을 지켜야한다.
    - 그 결과 깨지기 쉽고, 위험하고, 모순적인 메커니즘이 탄생한다. 생성자를 호출하지 않고도 객체를 생성할 수 있게 되는 것이다.

---

# clone 메서드 일반 규약

> 이 객체의 복사본을 생성해 반환한다. '복사'의 정확한 뜻은 그 객체를 구현한 클래스에 따 라 다를 수 있다. 일반적인 의도는 다음과 같다. 어떤 객체 x에 대해 다음 식은 참이다.
- `x. clone() != x`
  또한 다음 식도 참이다.
- `x.clone().getClass() == x.getClass()`
  하지만 이상의 요구를 반드시 만족해야 하는 것은 아니다.
  한편 다음 식도 일반적으로 참이지만, 역시 필수는 아니다.
- `x. clone().equals (x)`
  관례상, 이 메서드가 반환하는 객체는 Super.clone을 호출해 얻어야 한다. 이 클래스와 (Object를 제외한) 모든 상위 클래스가 이 관례를 따른다면 다음 식은 참이다.
- `x.clone().getClass() = x.getClass()`
  관례상, 반환된 객체와 원본 객체는 독립적이어야 한다. 이를 만족하려면 `super.clone`으로 얻은 객체의 필드 중 하나 이상을 반환 전에 수정해야 할 수도 있다.
>

강제성이 없다는 점만 빼면 생성자 연쇄와 살짝 비슷하다. 즉, 클론 메서드가 super.clone이 아닌 생성자를 호출해 얻은 인스턴스를 반환해도 컴파일러 에러가 발생하지 않는다

하지만 이 클래스의 하위 클래스에서 super.clone을 호출한다면 잘못된 클래스 객체가 만들어져 하위 클래스의 clone 메서드가 제대로 동작하지 않게 된다.

---

# Clone 구현 예시

제대로 동작하는 clone 메서드를 가진 상위 클래스를 상속해 Cloneable을 구현해보고 싶다고 해보자.

### 1. super.clone()

```java
@Override
    protected Object clone() throws CloneNotSupportedException {
        try{
            return (PhoneNumber) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(); // Can't happen
        }
    }
```

먼저 super.clone을 호출한다. 그렇게 얻은 객체는 원본의 완벽한 복제본일 것이다.

클래스에 정의된 모든 필드는 원본 필드와 똑같은 값을 갖는다. 모든 필드가 기본 타입이거나 불변 객체를 참조한다면 이 객체는 완벽한 상태라 더 손볼 것이 없다.

### 2. 재귀 호출

클래스가 가변 객체를 참조하는 순간 문제가 된다.

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

		public Object pop() {
	    if (size == 0) {
	        throw new EmptyStackException();
	    }
	    Object result = elements[--size];
	    elements[size] = null; // 다 쓴 참조 해제
	    return result;
		}

    private void ensureCapacity() {
        if (elements.length == size) {
            elements = Arrays.copyOf(elements, 2 * size + 1);
        }
    }
}
```

clone 메서드가 단순히 super.clone 결과를 그대로 반환한다면 반횐된 인스턴스의 size 필드는 올바른 값을 갖겠지만, elements 필드는 원본 Stack 인스턴스와 똑같은 배열을 참조하게 된다.

Stack 클래스의 하나뿐인 생성자를 호출한다면 이러한 상황은 일어나지 않는다. **clone 메서드는 사실상 생성자와 같은 효과를 낸다. 즉, clone은 원본 객체에 아무런 해를 끼치지 않는 동시에 복제된 객체의 불변식을 보장해야 한다.**

이를 위해 Stack의 clone 메서드는 스택 내부 정보를 복사해야 하는데, 가장 쉬운 방법은 elements 배열의 clone을 재귀적으로 호출해주는 것이다. `elements.clone()` 의 결과를 Object[] 형변환할 필요는 없다. **배열의 clone은 런타임과 컴파일 타임 타입 모두 원본 배열과 똑같은 배열을 반환한다.**

```java
@Override
    protected Object clone() throws CloneNotSupportedException {
        try {
            Stack result = (Stack) super.clone();
            result.elements = elements.clone(); 
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(); // Can't happen
        }
    }
```

elements 필드가 final 이었다면 이 방식은 작동하지 않는다. **가변 객체를 참조하는 필드는 final로 선언하라라는 일반 용법과 충돌한다.** 또한, clone을 재귀적으로 호출하는 것만으로 충분하지 않을 때도 있다.

### 3. 참조 필드 clone

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
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = buckets.clone();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

복제본은 자신만의 버킷 배열을 갖지만, 이 배열은 원본과 같은 연결리스트를 참조한다. **즉, 가변 상태를 공유한다.**

이를 위해서 각 버킷을 구성하는 연결 리스트를 복사해야 한다.

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

        // 재귀적으로 복사
        Entry deepCopy() {
            return new Entry(key, value, next == null ? null : next.deepCopy());
        }
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++) {
                if (buckets[i] != null) {
                    result.buckets[i] = buckets[i].deepCopy();
                }
            }
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }

}
```

간단하며 버킷이 길지 않다면 잘 작동하지만, 연결 리스트를 복제하는 방법으로는 좋지 않다. **재귀 호출 때문에 리스트의 원소 수만큼 스택 프레임을 소비하여, 리스트가 길면 스택 오버 플로를 일으킬 위험이 있다.**

이문제를 피하려면 deepCopy 재귀 호출 대신 반복자를 써서 순회해야 한다.

```java
  			Entry deepCopy() {
            Entry result = new Entry(key, value, next);
            for (Entry p = result; p.next != null; p = p.next) {
                p.next = new Entry(p.next.key, p.next.value, p.next.next);
            }
            return result;
        }
```

### 4. 고수준 메서드 호출

super.clone을 호출하여 얻은 객체의 모든 필드를 초기 상태로 설정한 다음, 원본 객체의 상태를 다시 생성하는 고수준 메서드를 호출하는 방식

고수준 API를 활용해 복제하면 간단하고 우아한 코드를 얻게되지만, 저수준에서 바로 처리할 때보다 느리기도 하고, 필드 단위 객체 복사를 우회하기 때문에 Cloneable 아키텍처와 어울리지 않다

---

# 상속에서의 clone

생성자에서는 재정의될 수 있는 메서드를 호출하지 않아야 하는데 clone 메서드도 마찬가지다.

만약 clone이 하위 클래스에서 재정의한 메서드를 호출하면, 하위 클래스는 복제 과정에서 자신의 상태를 교정할 기회를 잃게 되어 원본과 복제본의 상태가 달라질 가능성이 크다.

따라서 앞 문단에서 얘기한 put(key, value) 메서드는 final이거나 private이어야 한다(private이라

면 finall이 아닌 public 메서드가 사용하는 도우미 메서드일 것이다).

상속용 클래스는 Cloneable 클래스를 구현해서는 안된다.

Object의 방식을 모방할 수 있다. 제대로 작동하는 clone 메서드를 구현해 protected로 두고 CloneNotSupportedException도 던질수 있다고 선언하는 것이다. 이 방식은 Object를 바로 상속할 때 처럼 Cloneable 구현 여부를 하위 클래스에서 선택하다록 해준다.

다른 방법으로는, clone을 동작하지 않게 구현해놓고 하위 클래스에서 재정의하지 못하게 할 수 있다.

```java
@Override
protected final Object clone() throws CloneNotSupportedException {
	throw new CloneNotSupportedException();
}
```

Cloneable을 구현한 스레드 안전 클래스를 작성할 떄는 clone 메서드 역시 적절히 동기화해줘야 한다. 요약하면, Cloneable을 구현하는 모든 클래스는 clone을 재정의해야 한다. 이 때, 접근 제한자는 public으로, 반환 타입은 클래스 자신으로 변경한다.

---

# 복사 생성자와 복사 팩토리를 사용하자

Cloneable을 이미 구현한 클래스를 확장한다면 어쩔 수 없이 clone을 잘 작동하도록 구현해야 한다. 그렇지 않은 상황에서는 복사 생성자와 복사 팩토리를 사용하는 게 더 나을 수 있다.

복사 생성자란 단순히 자신과 같은 클래스의 인스턴스를 인수로 받는 생성자이다.

```java
public Yum(Yum yum){};

public static Yum newInstance(Yum yum){};
```

복사 생성자와 복사 팩토리는 Cloneable/clone 방식 보다 더 나은 면이 많다

- 생성자를 쓰지 않는 객체 생성 방식을 사용하지 않음
- 엉성하게 문서화된 규약에 기대지 않음
- 정상적인 final 필드 용법과 충돌하지 않음
- 불필요한 검사 예외를 던지지 않음
- 형변환이 필요없음
- 인터페이스 타입의 인스턴스를 인수로 받을 수 있음

---

# 정리

- 새로운 인터페이스를 만들 때 Cloneable 을 확장해서는 안되며, 새로운 클래스도 이를 구현해서는 안된다
- final 클래스라면 Cloneable 을 구현해도 위험이 크지 않지만, 성능 최적화 고한점ㅇ에서 검토한 후 드물게 허용해야 한다.
- 기본 원칙은 복제 기능은 생성자와 팩토리를 이용하는 게 최고
    - 단, 배열만은 clone 메서드 방식이 가장 깔끔하기는 하다.


---

# Q

- Cloneable을 구현한 스레드 안전 클래스를 작성할 때는 clonse 메서드 역시 적절히 동기화해줘야 한다