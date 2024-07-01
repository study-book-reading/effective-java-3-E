# 한정적 와일드카드를 사용해 API 유연성 높이기

- 제네릭 프로그래밍에서 한정적 와일드카드를 사용하면 API의 유연성과 재사용성을 높일 수 있다.
- 한정적 와일드카드는 메서드 매개변수에 적용되어 다양한 타입의 객체를 처리할 수 있게 하며, 타입 안전성을 유지하면서도 보다 유연한 코드를 작성할 수 있도록 도와준다.

### 한정적 와일드 카드의 종류

1. 무공변 와일드카드 (Unbounded Wildcards): `<?>`

- 어떤 타입이든 허용됩니다.
- 예: `List<?>`

2. 상한 한정 와일드카드 (Upper Bounded Wildcards): `<? extends T>`

- T와 T의 하위 타입을 허용합니다.
- 주로 생산자 역할을 하는 매개변수에 사용됩니다.
- 예: `List<? extends Number>`

3. 하한 한정 와일드카드 (Lower Bounded Wildcards): `<? super T>`

- T와 T의 상위 타입을 허용합니다.
- 주로 소비자 역할을 하는 매개변수에 사용됩니다.
- 예: `List<? super Integer>`

### 생산자-소비자 원칙(PECS)

- 생산자 (Producer) - extends: 데이터를 생성하는 역할을 하는 경우 상한 한정 와일드카드를 사용한다.
- 소비자 (Consumer) - super: 데이터를 소비하는 역할을 하는 경우 하한 한정 와일드카드를 사용한다.

```java
import java.util.Collection;
import java.util.List;

public class WildcardExample {

    // 상한 한정 와일드카드 사용 예시 - 생산자
    public static <T> void copy(List<? extends T> src, List<? super T> dest) {
        for (T item : src) {
            dest.add(item);
        }
    }

    public static void main(String[] args) {
        List<Integer> integers = List.of(1, 2, 3);
        List<Number> numbers = new ArrayList<>();
        copy(integers, numbers);
        System.out.println(numbers);
    }
}
```

- 이 예제에서는 copy 메서드가 두 리스트를 받아서 첫 번째 리스트의 모든 요소를 두 번째 리스트에 추가한다.
- 여기서 첫 번째 리스트는 상한 한정 와일드카드를 사용하여 T의 하위 타입을 허용하고, 두 번째 리스트는 하한 한정 와일드카드를 사용하여 T의 상위 타입을 허용한다.

### 유연한 API 설계

와일드카드를 사용하면 더 유연하고 재사용 가능한 API를 설계할 수 있다.
예를 들어, 컬렉션에서 최댓값을 찾는 메서드를 작성할 때 재귀적 타입 한정을 사용하여 다양한 타입의 컬렉션을 처리할 수 있다.

```java
import java.util.Collection;
import java.util.Collections;

public class RecursiveTypeBound {

    // 재귀적 타입 한정 사용 예시
    public static <T extends Comparable<? super T>> T max(Collection<? extends T> coll) {
        if (coll.isEmpty()) {
            throw new IllegalArgumentException("컬렉션이 비어 있습니다.");
        }
        return Collections.max(coll);
    }

    public static void main(String[] args) {
        List<String> strings = List.of("apple", "banana", "cherry");
        System.out.println(max(strings)); // cherry 출력
    }
}
```

- 이 예제에서 max 메서드는 T 타입이 Comparable 인터페이스를 구현하고, T의 상위 타입과 비교할 수 있는 타입만 허용하도록 제한한다.
- 이를 통해 다양한 타입의 컬렉션에서 최댓값을 안전하게 구할 수 있다.

# 결론

1. 한정적 와일드카드 사용의 중요성
    - 와일드카드를 사용하면 API의 유연성과 재사용성을 크게 향상시킬 수 있다.
    - 클라이언트 코드가 다양한 타입의 객체를 처리할 수 있도록 한다.

2. 생산자-소비자 원칙 (PECS)
    - 생산자 역할을 하는 매개변수에는 extends 와일드카드를 사용하고, 소비자 역할을 하는 매개변수에는 super 와일드카드를 사용하여 타입 안전성을 보장한다.

3. 유연한 API 설계
    - 와일드카드를 활용하여 더 유연하고 재사용 가능한 API를 설계함으로써 코드의 가독성과 유지보수성을 향상시킬 수 있다.