## 아이템26. 로 타입은 사용하지 말라 

### 로타입이란? 로타입 vs 제네릭

- 제네릭 클래스/인터페이스: 클래스/인터페이스 선언에 타입 매개변수를 사용한 클래스. 통틀어 제네릭 타입이라고 부름.
ex) List<E>, List인터페이스는 타입 매개변수 E를 받는다.  

- 제네릭 타입은 매개변수화 타입을 정의하여 사용한다. 클래스/인터페이스 이름이 나오고 꺽쇠 안에 실제 타입 매개변수들을 나열한다.
ex) List<String>, 원소의 타입이 String인 리스트를 뜻함.

- 제네릭 타입을 하나 정의하면, 그에 딸린 로 타입(raw type)도 함께 정의됨. 로 타입이란 제네릭 타입에서 타입 매개변수를 전혀 사용하지 않을 때를 말함.
ex) List<E>의 로 타입은 List

로타입이 존재하는 이유는 제네릭이 생기기 전 코드와 제네릭 사용 코드 간의 **호환성** 떄문임.

제네릭 지원 전 컬렉션 선언 예시)
```java
private final Collection stamps = ... ;  //Stamp 인스턴스만 취급함   <- 주석으로 표시(강제성이 없음)
```
위 코드는 현재도 동작하긴하지만 좋지 않은 예이다.  

그 이유는 강제성이 없기 때문에 실수로 다른 타입을 사용할 경우에도 경고 메시지만 표시할 뿐 컴파일되고 실행된다.  

실수로 다른 타입을 사용했을 경우의 예시)
```java
//Stamp가 아닌 Coin을 넣음
stamps.add(new Coin(...));   //unchecked call 경고를 뱉음

//반복자의 로 타입
for (Iterator i = stamp.iterator(); i.hasNext(); ) {
  Stamp stamp = (Stamp) i.next(); //ClassCastException 발생
  stamp.cancel();
}
```

오류 발생이 런타임 시점으로 늦춰지기 때문에 추후 문제가 발생했을 때 문제 해결에 어려움이 발생할 수 있다.

제네릭 사용시의 타입 안전성)
```java
private final Collection<Stamp> stamps = ...;
```

위 처럼 선언하면 stamps에는 Stamp 인스턴스만 넣어야 함(컴파일러가 보장).  
컴파일러 경고(아이템 27)를 숨기지 않았다면 컴파일 됐을 시 의도대로 동작함을 보장함.

Test.java:9: error: incompatible types: Coin cannot be converted to Stamp  
stamps.add(new Coin());  


컴파일러는 컬렉션에서 원소를 꺼내는 모든 곳에 보이지 않는 형변환을 추가하여 절대 실패하지 않음을 보장함.  

자바에선 호환성을 위해 로타입을 남겨뒀고, 제네릭 구현에는 소거(erasure; 아이템 28) 방식을 사용함.  

---

### 로타입과 유사한 Object는 사용해도 되는가? 그 차이는?  

List같은 로타입은 사용하면 안된다고 했는데, List<Object>같은 임의 객체를 허용하는 매개변수화 타입은 괜찮은가?  
결론적으로 List는 제네릭타입과 전혀 관련이 없고, List<Object>는 모든 타입을 허용한다는 의사를 컴파일러에 전달한 것이다.  

List는 타입안전성이 없고, List<Object>는 타입안전성이 있다.  

```java
public static void main(String[] args) {
  List<String> strings = new ArrayList<>();
  unsafeAdd(strings, Integer.valueOf(42));
  String s = strings.get(0); // 컴파일러가 자동으로 형변환 코드를 넣어줌.
}

private static void unsafeAdd(List list, Object o) {
  list.add(o);
}
```
위 코드는 컴파일 되지만 로타입인 List를 사용하여 다음과 같은 경고가 발생함.  
Test.java:10: warning: [unchecked] unchecked call to add(E) as a member of the raw type List  
list.add(o);  

이 프로그램을 이대로 실행하면 strings.get(0)의 결과를 형변환하려 할 때 ClassCastException을 던진다. Integer를 String으로 변환하려 시도한 것이다.  
로타입을 사용하였고, 형변환에 대한 check를 하지 않았으니 uncheck경고대로 형변환시 실패로 인한 문제가 발생함.  

만약 List를 매개변수화 타입인 List<Object>로 바꾼다면, 오류 메시지가 출력되면서 컴파일조차 되지 않는다.  

이렇게 되면 타입이 유연하지 않기 때문에 사용하기 귀찮다고 느낄 수 있다.  

---

### 와일드카드 타입을 사용하여, 타입 안전하며 유연하게 사용할 수 있다.

```java
//제네릭 사용이 익숙하지 않은 경우, 타입 확인이 안되는 로타입(Set)을 사용했다.
static int numElementsInCommon(Set s1, Set s2) {
  int result = 0;
  for (Object o1 : s1) {
    if (s2.contains(o1))
      result++;
  return result;
}
```

위 코드는 동작은 하지만 타입이 안정적이지 않다. 

따라서 비한정적 와일드카드타입을 대신 사용하는 게 좋다.  
- 비한정적 와일드카드타입: 상한, 하한이 없는 와일드카드 꺽쇠에 ? 로 표시
- 한정적 와일드카드타입: 상한, 하한이 있는 와일드카드 꺽쇠에 ? extends T, ? super T 로 표시

로타입엔 아무거나 넣을 수 있지만, 와일드카드타입(?)에는 null외에는 어떠한 것도 넣을 수 없다.  


---

### 로타입의 예외

1. class 리터럴에는 로타입을 써야한다.  
자바 명세는 class 리터럴에 매개변수화 타입을 사용하지 못하게 했다. (배열과 기본 타입은 허용)
List.class, String[].class, int.class는 허용.
List<String>.class와 List<?>.class는 허용안함.
컴파일 시에는 꺽쇠가 생략되기 때문에 해당 class 경로 파일을 찾을 수 없음.

2. instanceof 연산자를 사용하는 케이스
런타임에는 제네릭 타입 정보가 지워지므로 instanceof 연산자는 비한정적 와일드카드 타입 이외의 매개변수화 타입에는 적용할 수 없음.
로타입이든 비한정적 와일드카드 타입이든 instanceof 연산자는 완전히 똑같이 동작함.

```java
if (o instanceof Set) {   //로타입
  Set<?> s = (Set<?>) o;  //와일드카 타입
}
```