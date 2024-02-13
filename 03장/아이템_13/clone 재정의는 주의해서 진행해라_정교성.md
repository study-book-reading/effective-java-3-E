## 아이템13. clone 재정의는 주의해서 진행해라

```java
//Object의 clone메서드
@IntrinsicCandidate
protected native Object clone() throws CloneNotSupportedException; 
```

```java
//Cleanable 인터페이스
public interface Cloneable {  //선언된 메서드가 없다.
}
```

Cloneable은 복제해도 되는 클래스임을 명시하는 용도의 믹스인 인터페이스(아이템 20)지만, 아쉽게도 의도한 목적을 제대로 이루지 못했다.  
자바의 다른 대표적인 믹스인 인터페이스로 Comparable이 있는데, compareTo 메서드를 통해 비교한다.  
사용하고자 하는 클래스에 Comparable 인터페이스를 구현하여 해당 메서드를 사용할 수 있다.  

>**믹스인 인터페이스란?** 
>대상 타입의 주 기능에 선택적 기능을 '혼합'한다고 해서 믹스인이라고 부름.

Comparable과 달리 Cloneable인터페이스는 비어있고 Clone메서드가 Object에 protected로 선언됐다.  
그래서 Cloneable을 구현하는 것만으론 외부 객체에서 clone메서드를 호출할 수 없다.  
리플렉션(아이템 65)을 사용하면 가능하지만, 100% 성공하는 것도 아니다.  
해당 객체가 접근이 허용된 clone 메서드를 제공한다는 보장이 없기 때문이다.

```java
//리플렉션을 활용하여 접근

// clone() 메서드를 호출할 객체 생성
MyClass obj = new MyClass();

// MyClass 클래스의 clone() 메서드 가져오기
Method cloneMethod = Object.class.getDeclaredMethod("clone");

// 메서드 접근성 변경 (protected -> public)
cloneMethod.setAccessible(true);

// clone() 메서드 호출
MyClass clonedObj = (MyClass) cloneMethod.invoke(obj);
```


---
### clone메서드 사용법

- 사용할 클래스에 Cloneable 구현
- clone 재정의, super.clone메서드 호출 후 public접근 제한자로 설정, 반환 타입은 클래스 자신으로 지정.
- super.clone을 호출한 후 필요한 필드를 전부 적절히 수정
  필드들에 '깊은 구조'가 있다면 깊은 복사를 통해 복제한 객체를 바라보도록 수정해야 함.
  만약 기본 타입 필드와 불변 객체만 있다면 생략가능 하지만, 일련번호나 고유ID의 경우 적절히 수정 필요.  

:star:**clone메서드의 일반 규약**
- 이 객체의 복사본을 생성해서 반환함.
- x.clone() != x 는 참이다. (반드시 만족해야 하는 것은 아니다)
- x.clone().getClass() == x.getClass() 는 참이다. (반드시 만족해야 하는 것은 아니다)
- x.clone().equals(x) 는 일반적으로 참이지만, 필수는 아니다.
- 관례상, 이 메서드가 반환하는 객체는 super.clone을 호출해 얻어야 한다.  
  이 클래스와 (Object를 제외한) 모든 상위 클래스가 이 관례를 따른다면 다음은 참이다.  
  x.clone().getClass() == x.getClass()

강제성이 없다는 점만 빼면 생성자 연쇄(constructor chanining)와 살짝 비슷한 매커니즘이다.  
즉, clone메서드가 super.clone이 아닌 생성자를 호출해 얻은 인스턴스를 반환해도 문제가 없다.  
하지만 하위 클래스에서 super.clone을 호출한다면 잘못된 클래스의 객체가 만들어져서 문제가 된다.  
(상위 클래스에서 new를 통해 객체를 반환하면, 하위 클래스에서도 상위클래스 객체를 반환함. super.clone을 통해 호출하면 하위클래스의 객체가 반환됨.)  

>**생성자 연쇄란?**
>this나 super를 사용하여 생성자에서 다른 생성자를 호출하는 기술

```java
@Override
public PhoneNumber clone() {
  try {
    return (PhoneNumber) super.clone();
  } catch (CloneNotSuppertedException e) {
    throw new AssertionError();
  }
}
```

:star:**가변 객체를 참조하는 클래스의 clone 사용**

가변 객체를 참조하면서 여러 가지 문제들이 발생한다.  

```java
public class Stack {
  private Object[] elements;
  private int size = 0;
  private static final int DEFAULT_INITIAL_CAPACITY = 16;

  //...생략
```

위의 스택클래스를 예로 들어 clone메서드를 사용한다면, 기본 타입인 size필드는 문제 없이 복사되지만, 배열참조인 elements는 같은 참조 주소를 복사하기 때문에 수정시 불변식을 해친다.  

clone 메서드는 사실상 생성자와 같은 효과며, 배열참조인 elements의 배열 clone 메서드를 재귀적으로 호출해서 Stack 내부 정보를 복사한다.

- 필드에 배열이 있는 경우
  해당 배열에 clone을 재귀적으로 호출하여 복제(배열의 clone메서드는 권장됨)
```java
@Override
public Stack clone() {
  try {
    Stack result = (Stack) super.clone();
    result.elements = elements.clone();     //배열의 clone은 런타임 타입과 컴파일타임 타입 모두 원본 배열과 똑같은 배열을 반환함.
  } catch (CloneNotSupportedException e) {
    throw new AssertionError();
  }
}
```

```java
public class HashTable implements Cloneable {
  private Entry[] buckets = ...;

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
//생략
}
```

- 복잡한 가변 상태를 갖는 클래스의 잘못된 clone메서드(가변 상태 공유, 얕은 복사)

```java
@Override public HashTable clone() {
  try {
    HashTable result = (HashTable) super.clone();
    result.buckets = buckets.clone();
    return result;
  } catch (CloneNotSupportedException e) {
    throw new AssertionError();
  }
}
```

>**얕은 복사 vs 깊은 복사**  
>얕은 복사는 원본 객체의 모든 필드를 복사, 러나 복사된 객체의 필드들은 원본 객체의 필드들을 참조. 필드들이 동일한 객체를 가리킴  
>이것은 가변객체가 있을 시 문제가 될 수 있음. 왜냐하면 한 객체가 필드값 변경시 다른 객체도 값이 변경됨.  
>
>깊은 복사는 원본 객체의 모든 필드를 복사하고, 필요한 경우에는 필드 내에 있는 객체들까지 재귀적으로 복사.  
>이렇게 함으로써 복사된 객체와 원본 객체는 완전히 별개의 객체가 됨.  
>깊은 복사는 객체를 완전히 분리하여 독립적인 복사본을 생성하므로, 복사된 객체의 수정이 원본 객체에 영향을 미치지 않음.  
>깊은 복사는 복사하는 객체와 그 객체 내의 모든 객체를 재귀적으로 복사하기 때문에 복잡한 구조를 가진 객체의 복사에는 시간과 자원이 많이 소모될 수 있음.  
>
>- 깊은 복사(재귀복사) 대체 방법
>
>1. 직렬화와 역직렬화
>2. 복사 생성자나 복사 팩토리 메서드
>3. Apache Commons 라이브러리 등의 유틸리티 라이브러리 활용
>4. 클론 라이브러리 활용


복제본은 자신만의 버킷 배열을 갖지만, 이 배열은 원본과 같은 연결 리스트를 참조하여 예기치 않은 동작을 할 가능성이 있다.  

- 복잡한 가변 상태를 갖는 클래스의 clone메서드
```java
public class HashTable implements Cloneable {
  private Entry[] buckets = ...;

  private static class Entry {
    final Object key;
    Object value;
    Entry next;

    Entry(Object key, Object value, Entry next) {
      this.key = key;
      this.value = value;
      this.next = next;
    }

    Entry deepCopy() {
      return new Entry(key, value, next == null ? null : next.deepCopy());
    }
  }

  @Override
  public HashTable clone() {
    try {
      HashTable result = (HashTable) super.clone();
      result.buckets = new Entry[buckets.length];
      for (int i = 0; i < buckets.length; i++)
        if (buckets[i] != null)
          result.buckets[i] = buckets[i].deepCopy();
      return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
  }
  //...생략
}
```

private 클래스인 HashTable.Entry는 깊은 복사(deep copy)를 지원하도록 보강  
deepCopy메서드는 자신이 가리키는 연결리스트 전체를 복사하기 위해 자신을 재귀적으로 호출  
간단한 방법이지만, 버킷이 길다면 스택 프레임 소비로 인한 스택오버플로가 발생할 가능성이 있다.  

- 재귀가 아닌 반복자를 사용한 복사
```java
Entry deepCopy() {
  Entry result = new Entry(key, value, next);
  for (Entry p = result; p.next != null; p = p.next)
    p.next = new Entry(p.next.key, p.next.value, p.next.next);
  return result;
}
```

- 성능은 느리지만 간단한 고수준 API활용 복사 방법

super.clone을 호출하여 얻은 객체의 모든 필드를 초기 상태로 설정  
원본 객체의 상태를 다시 생성하는 고수준 메서드들을 호출  

```java
//객체를 초기 상태로 재설정하는 메서드
public void reset() {
    this.value = 0; // 기본값으로 초기화
    this.name = null; // 기본값으로 초기화
    this.data.clear(); // Hashtable 초기화
    // 다른 초기화 작업들 수행...
}

// 객체 복제 메서드
@Override
public MyClass clone() {
    try {
        MyClass cloned = (MyClass) super.clone();
        // 복제된 객체의 필드들을 초기 상태로 설정
        cloned.reset();
        // Hashtable 깊은 복사
        Enumeration<String> keys = this.data.keys();
        while (keys.hasMoreElements()) {
            String key = keys.nextElement();
            cloned.data.put(key, this.data.get(key));
        }
        return cloned;
    } catch (CloneNotSupportedException e) {
        // 예외 처리
        e.printStackTrace();
        return null;
    }
}
```

이 방법은 저수준 처리보다 느리고, Cloneable 아키텍처의 기초가 되는 필드 단위 객체 복사를 우회하기 때문에 전체 Cloneable 아키텍처와는 어울리지 않는 방식임

:star:**주의사항**  
상속용 클래스에는 Cloneable을 구현하면 안 된다.  
1. Object의 방식을 모방하여 제대로 작동하는 clone메서드를 구현 protected로 두고 CloneNotSupportedException을 던질 수 있다고 선언  
이 방식은 Object를 바로 상속할 때처럼 Cloneable 구현 여부를 하위 클래스에게 선택하도록 함  
2. clone을 동작하지 않게 구현해놓고 하위 클래스에서 재정의하지 못하게 함  

```java
@Override
protected final Object clone() throws CloneNotSupportedException {
  throw new CloneNotSupportedException();
}
```
---
### 동시성 고려하기
cloneable을 구현한 스레드 안전 클래스를 작성할 때는 clone 메서드 역시 적절히 동기화해줘야 함.  
Object의 clone 메서드는 동기화를 고려하지 않음.  
super.clone 호출 외에 다른 할 일이 없더라도 clone을 재정의하고 동기화해줘야 함.  

---
### Cloneable의 대안
- 복사 생성자와 복사 팩터리

>복사생성자란?
>자신과 같은 클래스의 인스턴스를 인수로 받는 생성

>복사팩터리란?
>복사 생성자를 모방한 정적 팩터리

```java
//복사 생성자
public Yum(Yum yum) { ... };
```

```java
//복사 팩터리
public static Yum newInstance(Yum yum) { ... };
```

- 복사 생성자/팩터리의 이점은?
1. 생성자 방식이 더 안전.
2. 엉성하게 문서화된 규약에 의존하지 않음
3. 정상적인 final 필드 용법과도 충돌하지 않음
4. 불필요한 검사 예외를 던지지 않음
5. 형변환 필요 없음
6. 해당 클래스가 구현한 '인터페이스'타입의 인스턴스를 인수로 받을 수 있음(타입 변환 가능)
   ex) HashSet의 객체 s를 TreeSet으로 변환시, new TreeSet<>(s)로 처리

---
### 결론
Cloneable을 사용하지 말고 더 안전하고 나은 생성자와 팩터리를 활용하자.  
배열은 clone 메서드 방식을 사용할 수 있다.


[참조]
- [@IntrinsicCandidate 관련](https://vixxcode.tistory.com/204)
- [이펙티브 자바, 쉽게 정리하기 - item 13. clone 재정의는 주의해서 진행하라](https://jake-seo-dev.tistory.com/31)