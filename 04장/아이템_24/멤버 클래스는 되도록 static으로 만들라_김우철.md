## 아이템 24. 멤버 클래스는 되도록 static으로 만들라

### 중첩 클래스란?
- 다른 클래스 안에 정의된 클래스
- 자신을 감싼 바깥 클래스에서만 사용되어야 한다.
  - 그 외 쓰임새가 생기면 톱 레벨 클래스로 생성해야 한다.

### 중첩 클래스의 종류
- 정적 멤버 클래스
- (비정적) 멤버 클래스
- 익명 클래스
- 지역 클래스

- 정적 멤버 클래스를 제외한 클래스들은 내부 클래스라고 한다.

#### 정적 멤버 클래스
- class 내부에 static으로 선언된 클래스
- 바깥 클래스의 private 멤버에도 접근 가능
- private로 선언 시 바깥 클래스에서만 접근 가능
```java
public class Animal {
    private String name = "cat";

    // 열거 타입도 암시적 static
    public enum Kinds {
        MAMMALS, BIRDS, FISH, REPTILES, INSECT
    }

    private static class PrivateSample {
        private int temp;

        public void method() {
            Animal outerClass = new Animal();
            System.out.println("private" + outerClass.name); // 바깥 클래스인 Animal의 private 멤버 접근
        }
    }

    public static class PublicSample {
        private int temp;

        public void method() {
            Animal outerClass = new Animal();
            System.out.println("public" + outerClass.name); // 바깥 클래스인 Animal의 private 멤버 접근
        }
    }
}
```
- 바깥 클래스가 표현하는 객체의 한 부분(구성요소)일 때 사용

#### 비정적 멤버 클래스
- static이 붙지 않은 멤버 클래스
- 비정적 멤버 클래스의 인스턴스 메서드에서 `클래스명.this`를 사용해 바깥 인스턴스의 메서드나 참조를 가져올 수 있다.
- 바깥 인스턴스 없이 생성이 불가능하다.
    - 비정적 멤버 클래스의 인스턴스는 바깥 클래스의 인스턴스와 참조관계로 되어있다.
- 개념상 중첩 클래스의 인스턴스가 바깥 인스턴스와 독립적으로 존재할 수 있다면 정적 멤버 클래스로 만들어야 한다.
```java
public class TestClass {
    private String name = "yeonlog";

    public class PublicSample {
        public void printName() {
            // 바깥 클래스의 private 멤버 가져오기
            System.out.println(name);
        }

        public void callTestClassMethod() {
            // 바깥 클래스의 메소드 호출하기
            TestClass.this.testMethod();
        }
    }

    public PublicSample createPublicSample() {
        return new PublicSample();
    }

    public void testMethod() {
        System.out.println("hello world");
    }
}

    TestClass test = new TestClass();
    PublicSample publicSample1 = test.createPublicSample(); // 바깥 클래스의 인스턴스 메서드에서 생성자 호출
    PublicSample publicSample2 = test.new PublicSample(); // 바깥 인스턴스 클래스.new 멤버클래스()
```
단점
- 바깥 인스턴스 - 멤버 클래스 관계를 위한 시간과 공간 소모
- GC가 바깥 클래스의 인스턴스 수거 불가 → 메모리 누수 발생
    - 내부 클래스가 독립적으로 사용된다면 정적 멤버 클래스를 써야 한다.
- 참조가 눈에 보이지 않아 개발이 어려움
- **멤버클래스에서 바깥 인스턴스에 접근할 일이 없다면 무조건 static을 붙여서 정적 멤버 클래스로 만들자.**

사용처
- 비정적 멤버 클래스는 어댑터를 정의할 때 자주 쓰인다
  - 어떤 클래스의 인스턴스를 감싸 마치 다른 클래스의 인스턴스처럼 보이게 하는 뷰로 사용
  - HashMap은 keySet과 같이 자신의 컬렉션 뷰를 구현할 때 비정적 멤버 클래스를 사용한다.
  - Map의 key에 해당하는 값들을 Set으로 반환 해 주는 데 어댑터를 이용해서 **Map을 Set으로 제공**한다.
  - Set과 List같은 다른 컬렉션 인터페이스 구현체들도 자신의 반복자를 구현할 때 비 정적 멤버 클래스를 주로 사용한다.
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    ...
    public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new KeySet();
            keySet = ks;
        }
        return ks;
    }
    final class KeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }
        public final Iterator<K> iterator()     { return new KeyIterator(); }
        public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator() {
            return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (Node<K,V> e : tab) {
                    for (; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
}
```

LinkedList의 Iterator 비정적 멤버 클래스 구현
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    ...
    public ListIterator<E> listIterator(int index) {
        checkPositionIndex(index);
        return new ListItr(index);
    }
    private class ListItr implements ListIterator<E> {
        private Node<E> lastReturned;
        private Node<E> next;
        private int nextIndex;
        private int expectedModCount = modCount;
        ...
    }
}
```

#### 익명 클래스
- 이름이 없는 클래스
- 바깥 클래스의 멤버가 아님
- 쓰이는 시점에 선언 및 인스턴스 생성
- 자바8 이후엔 람다 사용

```java
List<Integer> list = Arrays.asList(10, 5, 6, 7, 1, 3, 4);
Collections.sort(list, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return Integer.compare(o1, o2);
    }
});
System.out.println(list);


        Collections.sort(list, (o1, o2) -> Integer.compare(o1, o2));
        Collections.sort(list, Comparator.comparingInt(o -> o));
```

제약사항
- 비정적인 문맥에서 사용될 때만 바깥 클래스의 인스턴스 참조 가능
    - static으로 선언된 메서드에서는 static만 접근 가능하다.
- ~~정적 문맥에서 static final 상수 외의 정적 멤버 갖기 불가능하다.~~
- 선언 지점에서만 인스턴스 생성 가능하다
- instanceof 검사 및 클래스 이름이 필요한 작업 수행 불가
    - 선언과 동시에 인스턴스 생성하고 더이상 사용하지 않는다
- 여러 인터페이스를 구현할 수 없고, 인터페이스를 구현하는 동시에 다른 클래스를 상속할 수도 없다.
- 익명 클래스를 사용하는 클라이언트는 그 익명 클래스가 상위 타입에서 상속한 멤버 외에는 호출할 수 없다

```java
abstract class Shape {
    abstract void draw();
}

public class Main {
    public static void main(String[] args) {
        Shape circle = new Shape() {
            void draw() {
                System.out.println("원을 그립니다.");
            }

            void customMethod() {
                System.out.println("익명 클래스의 특별한 메서드");
            }
        };

        circle.draw();
        // circle.customMethod(); // 컴파일 에러: customMethod()는 호출 불가
    }
}
```
- 인터페이스 구현 및 다른 클래스의 상속 불가능하다
- 표현식이 중간에 등장하므로 짧지 않ㅇ 가독성이 떨어진다.

```java
public class Calculator {
    private int x;
    private int y;

    public Calculator(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int plus() {
        Operator operator = new Operator() {
            private static final String COMMENT = "더하기"; // 상수
            // private static int num = 10; // 상수 외의 정적 멤버는 불가능
            // ★ 이거 되는데 왜 되는지 의문 ★
          
            @Override
            public int plus() {
                // Calculator.plus()가 static이면 x, y 참조 불가
                return x + y;
            }
        };
        return operator.plus();
    }
}

interface Operator {
    int plus();
}
```

사용처
- 즉석에서 작은 함수 객체나 처리 객체를 만드는데 주로 사용
- 자바8이후 람다로 사용

```java
List<Integer> list = Arrays.asList(10, 5, 6, 7, 1, 3, 4);

// 익명 클래스 사용
Collections.sort(list, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return Integer.compare(o1, o2);
    }
});

// 람다 도입 후
Collections.sort(list, Comparator.comparingInt(o -> o));
```
- 정적 팩터리 메서드 구현 시 사용
```java
static List<Integer> intArrayAsList(int[] a) {
    Objects.requiredNonNull(a);
    
    return new AbstracktList<>() {
        @Override public Integer get(int i) {
            return a[i];
        }
    }
}
```

#### 지역 클래스
- 지역변수를 선언할 수 있는 곳이면 실질적으로 어디서든 선언 가능
- 가독성을 위해 짧게 작성해야 한다.
- 유효범위는 지역 변수와 동일

```java
public class TestClass {
    private int number = 10;

    public TestClass() {
    }

    public void foo() {
        // 지역변수처럼 선언
        class LocalClass {
            private String name;
            // private static int staticNumber; // 정적 멤버 가질 수 없음

            public LocalClass(String name) {
                this.name = name;
            }

            public void print() {
                // 비정적 문맥에선 바깥 인스턴스를 참조 가능
                // foo()가 static이면 number에서 컴파일 에러
                System.out.println(number + name);
            }
        }
        LocalClass localClass1 = new LocalClass("local1"); // 이름이 있고
        LocalClass localClass2 = new LocalClass("local2"); // 반복해서 사용 가능
    }
}
```
