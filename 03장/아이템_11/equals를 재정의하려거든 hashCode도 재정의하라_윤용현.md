# equals를 재정의하려거든 hashCode도 재정의하라

---

# 규약

- equals 비교에 사용되는 정보가 변경되지 않았다면, 애플리케이션이 실행되는 동안 그 객체의 hashCode 메서드는 몇 번을 호출해도 일관되게 항상 같은 값을 반환해야 한다. 단, 애플리케이션을 다시 실행한다면 이 값이 달라져도 상관없다.
- equals(object)가 두 객체를 같다고 판단했다면, 두 객체의 hashCode는 똑같은 값을 반환해야 한다.
- equals(object)가 두 객체를 다르다고 판단했더라도, 두 객체의 hashCode가 서로 다른 값을 반환할 필요는 없다.  단, 다른 객체에 대해서는 다른 값을 반환해야 해시테이블 의 성능이 좋아진다.
    - HashMap은 해시 코드가 다른 엔트리끼리는 동치성 비교를 시도조차 하지 않도록 최적화되어 있다.

hashCode 재정의를 잘못했을 때 크게 문제가 되는 조항은 두번째다. 논리적으로 같은 객체는 같은 해시코드를 반환해야 한다.  아이템 10의 PhoneNumber 예시를 들어보자.

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(111,222,4444),"하이");

m.get(new PhoneNumber(111,222,4444)); // 를 실행하면 null이 반환된다
```

이 문제는 PhoneNumber에 적절한 hashCode 메서드만 작성해주면 해결된다. 어떻게 해시 코드를 반환해야할까?

---

# 좋은 해시 함수를 작성하는 요령

1. int 변수 result를 선언한 후 값 c로 초기화한다. 이 때 c는 해당 객체의 첫번째 핵심 필드를 단계 2.a 방식으로 계산한 해시코드이다. ( 여기서 핵심 필드는 equals 비교에 사용되는 필드를 말한다)
2. 해당 객체의 나머지 핵심 필드 f 각각에 대해 다음 작업을 수행한다.
    1. 해당 필드의 해시 코드 c를 계산한다.
        - 기본 타입 필드라면, Type.hashCode(f) (Type은 해당 기본 타입의 박싱 클래스)
        - 참조 타입 필드면서 이 클래스의 equals 메서드가 이 필드의 equals 를 재귀적으로 호출해 비교한다면, 이 필드의 hashCode를 재귀적으로 호출한다. 계산이 더 복잡해질 것 같으면, 이 필드의 표준형(ca-nonical representation)을 만들어 그 표준형의 hashCode를 호출한 다. 필드의 값이 nuLL이면 0을 사용한다(다른 상수도 괜찮지만 전통 적으로 0을 사용한다).
        - 필드가 배열이라면, 핵심 원소 각각을 별도 필드처럼 다룬다. 이상의 규칙을 재귀적으로 적용해 각 핵심 원소의 해시코드를 계산한 다음, 단계 2.b 방식으로 갱신한다. 배열에 핵심 원소가 하나도 없다면단순히 상수(0을 추천한다)를 사용한다. 모든 원소가 핵심 원소라면 `Arrays.hashCode` 를 사용한다.
    2. 단계 2.a에서 계산한 해시코드 c로 result를 갱신한다. 코드로는 다음과 같다. `result = 31 * result + c);`
3. result를 반환한다.

해시코드를 다 구현했다면 이 메서드가 동치인 인스턴스에 대해 똑같은 해시코드를 반환할 지 자문해보고, 단위 테스트를 작성하자.

파생 필드는 해시코드 계산에서 제외해도 된다. 즉, 다른 필드로부터 계산해 낼 수 있는 필드는 모두 무시해도 된다.

또한, equals 비교에 사용되지 않은 필드는 반드시 제외한다.

```java
// phoneNumber hashCode 예시
@Override
public int hashCode(){
	int result = Short. hashCode(areaCode);
	result = 31 * result + Short.hashCode(prefix);
	result = 31 * result + Short.hashCode(lineNum);
	return result;
}
```

---

# Objects.hash()

Objects 클래스가 제공하는 정적 메서드 hash는 앞선 요령대로 구현한 코드와 비슷한 수준의 hashCode 함수를 단 한줄로 작성할 수 있다. 하지만 속도는 더 느리다.  입력 인수를 담기 위한 배열이 만들어지고, 입력 중 기본 타입이 있다면 박싱과 언박싱도 거쳐야 하기 때문이다. 그러니 hash 메서드는 성능에 민감하지 않은 상황에서만 사용하자.

```java
// phoneNumber hashCode 예시
@Override
public int hashCode(){
		return Objects.hash(lineNum,prefix,areaCode);
}
```

클래스가 불변이고 해시코드를 계산하는 비용이 크다면, 매번 새로 계산하기 보다는 캐싱하는 방식을 고려해보자.

이 타입의 객체가 주로 해시 키로 사용될 것 같다면 인스턴스가 만들어질 때 해시코드를 계산해둬야 한다. 또는 hashCode가 처음 불릴 때 계산하는 지연 초기화 전략도 사용할 수 있다. 이 방법은 스레드 세이프를 고려해야 한다.

```java
// 해시 코드 지연 초기화 예시

private int hashCode; // 자동으로 0으로 초기화

@Override
public int hashCode(){
	int result = hashCode;
	if(result == 0) {
		int result = Short. hashCode(areaCode);
		result = 31 * result + Short.hashCode(prefix);
		result = 31 * result + Short.hashCode(lineNum);
		hashCode = result;
	}
		return result;
}
```

성능을 높인답시고 해시코드를 계산할 때 핵심 필드를 생략해서는 안된다.

속도는 빨라지겠지만 해시 품질이 나빠져 해시테이블 성능이 떨어질 수 있다.

hashCode가 반환하는 값의 생성 규칙을 API 사용자에게 알리지 말자. 그래야 클라이언트가 이 값에 의지하지 않게 되고, 추후에 계산 방식을 바꿀 수 있다.

---

# 요약

- equals를 재정의할 때는 hashCode도 반드시 재정의한다.