## 아이템 11. equals를 재정의하려거든 hashCode도 재정의하라

앞선 아이템 10에서 Point클래스의 equals를 재정의하여 onUnitCircle메서드를 실행하면 분명 같은 x, y값을 가진 Point인스턴스들인데 true가 아닌 false를 반환함.

```java
//HashMap, HashSet 같은 컬렉션의 원소로 사용시 문제 발생 예시
Point p1 = new Point( 1, 0);
Point p2 = new Point( 0, 1);
Point p3 = new Point(-1, 0);
Point p4 = new Point( 0,-1);

System.out.println(onUnitCircle(p1));  //false, 1, 0값이 있음에도 false 반환
System.out.println(onUnitCircle(p2));  //true
System.out.println(onUnitCircle(p3));  //true
System.out.println(onUnitCircle(p4));  //false, 0, -1값이 있음에도 false 반환
```

위와 같은 문제 때문에 equals를 재정의한 클래스 모두에서 hashCode도 재정의함  
그렇지 않으면 hashCode 일반 규약을 어됨  

:star:**Object명세에서 규약내용**
- equals 비교에 사용되는 정보가 변경되지 않았다면, 애플리케이션이 실행되는 동안 그 객체의 hashCode메서드는 몇 번을 호출해도 일관되게 항상 같은 값을 반환해야 한다. (일관성)
- equals(Object)가 두 객체를 같다고 판단했다면, 두 객체의 hashCode는 똑같은 값을 반환해야 한다. (동치성)
- equals(Object)가 두 객체를 다르다고 판단했더라도, 두 객체의 hashCode가 서로 다른 값을 반환할 필요는 없다.
  단, 다른 객체에 대해서는 다른 값을 반환해야 해시테이블의 성능이 좋아진다.

특히, 논리적으로 같은 객체는 같은 해시코드를 반환해야 한다.  
Object의 기본 hashCode는 논리적으로 같은 두 객체를 전혀 다르다고 판단하여, 규약과 달리 (무작위처럼 보이는) 서로 다른 값을 반환한다.  

```java
//같은 x, y값들이 들어있는 point 인스턴스들의 hashcode결과, 다 다른 결과가 나옴
points.stream().sorted(Comparator.comparing(Point::hashCode)).forEach(p->System.out.println(p.hashCode()));
unitCircle.stream().sorted(Comparator.comparing(Point::hashCode)).forEach(p->System.out.println(p.hashCode()));

558638686
1023892928
1149319664
1747585824

990368553
1096979270
1324119927
2003749087
```

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "제니");

m.get(new PhoneNumber(707, 867, 5309));  // null, 제니 X
//HashCode를 재정의하지 않았기 때문에 서로 다른 해시코드를 반환(두번째 규약 못지킴)
//HashMap은 해시코드가 다른 엔트리끼리는 동치성 비교를 시도조차 하지않음.
```

- 적법하지만 사용하면 안되는 hashCode 예시
```java
@Override
public int hashCode() {
  return 42;
}
```

모든 객체에 똑같은 값을 반환하므로 모든 객체가 해시테이블의 버킷 하나에 담김.  
해시 테이블에 같은 버킷값이 들어오면 연결리스트형식으로 엔트리들이 연결된다.  

해시테이블은 O(1)에 가까운데 연결리스트형식이 되어버려 하나의 버킷에 모두 담겨 O(n)으로 동작하게 되어 성능이 느려진다.  

좋은 해시 함수는 서로 다른 인스턴스에 다른 해시코드를 반환한다.  

---
:star:**좋은 hashCode를 작성하는 요령**  
1. int 변수 result를 선언한 후 값 c로 초기화한다. 이때 c는 해당 객체의 첫 번째 핵심 필드를 단계 2.a 방식으로 계산한 해시코드다(여기서 핵심 필드란 equals 비교에 사용되는 필드를 말한다.(아이템 10)
2. 해당 객체의 나머지 핵심 필드 f 각각에 대해 다음 작업을 수행한다.  
  a. 해당 필드의 해시코드 c를 계산한다.  
    1. 기본 타입 필드라면, Type.hashCode(f)를 수행한다. 여기서 Type은 해당 기본 타입의 박싱 클래스다.  
    2. 참조 타입 필드면서 이 클래스의 equals 메서드가 이 필드의 equals를 재귀적으로 호출해 비교한다면, 이 필드의 hashCode를 재귀적으로 호출한다. 계산이 더 복잡해질 것 같으면, 이 필드의 표준형을 만들어 그 표준형이 hashCode를 호출한다. 필드의 값이 null이면 0을 사용한다(다른 상수도 괜찮지만 전통적으로 0을 사용한다.)  
    3. 필드가 배열이라면, 핵심 원소 각각을 별도 필드처럼 다룬다. 이상의 규칙을 재귀적으로 적용해 각 핵심 원소의 해시코드를 계산한 다음, 단계 2.b 방식으로 갱신한다. 배열에 핵심 원소가 하나도 없다면 단순히 상수(0을 추천한다)를 사용한다. 모든 원소가 핵심 원소라면 Arrays.hashCode를 사용한다.  
  b. 단계 2.a에서 계산한 해시코드 c로 result를 갱신한다. 코드로는 다음과 같다. result = 31 * result + c;  
3. result를 반환한다.

---
:star:**주의사항**
1. 파생 필드는 해시코드 계산에서 제외해도 된다.
2. equals 비교에 사용되지 않은 필드는 반드시 제외해야 한다.

단계 2.b의 곱셈 31 * result는 필드를 곱하는 순서에 따라 result값이 달라지게 한다.  
그 결과 클래스에 비슷한 필드가 여러 개일 때 해시 효과를 크게 높여준다.


전형적인 hashCode 메서드
```java
@Override
public int hashCode() {
  int result = Short.hashCode(areaCode);
  result = 31 * result + Short.hashCode(prefix);
  result = 31 * result + Short.hashCode(lineNum);
  return result;
}
```
위와 같은 코드는 기능면에서 충분히 좋으나, 해시 충돌이 더욱 적은 방법을 꼭 써야 한다면 구아바의 com.google.common.hash.Hashing을 참고  

- Objects의 hash함수 사용
성능에 민감하지 않지만 더 쉬운 방법    
Objects의 hash는 임의의 개수만큼 객체를 받아 해시코드르 계산해 주는 정적 메서드다.  
```java
@Override
public int hashCode() {
  return Objects.hash(lineNum, prefix, areaCode);  //성능이 살짝 아쉬움
}
```

- 클래스가 불변이고 해시코드 계산 비용이 크다면 캐싱 방식을 고려
해시의 키로 사용되지 않는다면, hashCode가 처음 불릴 때 계산하는 지연초기 전략도 있다.  
필드를 지연 초기화하려면 그 클래스를 스레드 안전하게 만들도록 신경써야 한다.(아이템 83)  
```java
private int hashCode;  //자동으로 0으로 초기화된다.

@Override public int hashCode() {
  int result = hashCode;
  if (result = 0) {
    result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    hashCode = result;
  }
  return result;
}
```
:star:**주의사항**
- 성능을 높이기 위해 해시코드를 계산할 때 핵심 필드를 생략해선 안됨. 해시코드 성능은 빨라지지만, 해시테이블 성능이 떨어질 수 있음.  
- hashCode가 반환하는 값의 생성규칙을 API 사용자에게 자세히 공표하지 말자. 그래야 클라이언트가 이 값에 의지하지 않게 되고, 추후에 계산 방식을 바꿀 수도 있다.  

---
### 결론

equals를 재정의 할 때는 hashCode도 반드시 재정의 하라.  
AutoValue 프레임워크를 사용해서 자동으로 equals와 hashCode 만들어서 사용해라.  

---
[참조]
- [이펙티브-자바-아이템-11.-equals를-재정의하려거든-hashCode도-재정의하라](https://velog.io/@lychee/%EC%9D%B4%ED%8E%99%ED%8B%B0%EB%B8%8C-%EC%9E%90%EB%B0%94-%EC%95%84%EC%9D%B4%ED%85%9C-11.-equals%EB%A5%BC-%EC%9E%AC%EC%A0%95%EC%9D%98%ED%95%98%EB%A0%A4%EA%B1%B0%EB%93%A0-hashCode%EB%8F%84-%EC%9E%AC%EC%A0%95%EC%9D%98%ED%95%98%EB%9D%BC)