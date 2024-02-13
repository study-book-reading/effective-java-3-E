## 아이템 12. toString을 항상 재정의하라

### toString() 이란
- `toString()`이란 Object 클래스의 메서드이다.
- 자바의 모든 객체는 Object 클래스를 상속하기 때문에 모든 객체는 `toString()`메서드를 가지고 있다
- `toString()`은 아래와 같은 방식으로 객체를 표현하는 문자열을 리턴한다.
  ```java
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }
    ```
  - `클래스_이름@16진수` 로_표시한_해시코드를 반환한다.

### toString()은 언제 사용할까?
- `toString()`은 기본적으로 객체를 표현하는 문자열을 리턴해주기 때문에 개발할 때 사용한다.

1. 콘솔로 객체를 확인할 때
    1. `System.out.println()`
    2. `System.out.print()`
    3. 객체에 문자열을 연결(`SomeObject + ""`)
    4. assert 구문 사용 시
    5. 등등...
2. 디버깅 할 때
3. 로깅할 때

### toString()을 재정의해야 하는 이유
- 디버깅, 로깅, 콘솔로 확인 시 `클래스_이름@16진수` 로 표시한 해시코드와 같은 방식은 정보를 파악하기 어렵다
- 사용하려는 객체에 toString()을 재정의하여 더 유익한 정보를 확인할 수 있도록 해야한다.

#### toString()은 어떻게 재정의해야 하는가?
- 간결하면서 사람이 읽기 쉬운 형태의 유익한 정보를 반환하도록 재정의해야 한다.
```java
    PhoneNumber@adbbd -> 012-1234-5678   
    Car@442           -> Car{name=sun, position=2}
```
- 객체가 가진 주요 정보를 모두 반환
- 전화번호처럼 포맷이 정해져 있는 경우, 아래 코드와 같이 재정의하고 주석을 달아서 문서화를 할 수 있다.
```java
/** 
 * 전화번호의 문자열 표현을 반환합니다.
 * 이 문자열은 XXX-YYYY-ZZZZ 형태의 11글자로 구성됩니다.
 * XXX는 지역코드, YYYY는 접두사, ZZZZ는 가입자 번호입니다.
*/
@Override
public String toString() {
    return String.format("%03d-%04d-%04d", areaCode, prefix, lineNum);
}
```
- 책에서는 포맷을 정해준다면 앞으로의 유지보수에도 항상 이 포맷을 사용해야 하기 때문에 포맷에 얽매일 수 있다고 한다.
- 포맷 여부와 상관없이 재정의 하고 싶으면 아래와 같은 방식으로 `toString()`을 재정의를 권장한다.

```java
@Override
public String toString() {
    return "PhoneNumber{" +
                "areaCode='" + areaCode + '\'' +
                ", prefix='" + prefix + '\'' +
                ", lineNum='" + lineNum + '\'' +
                '}';
}
```
- 하지만, 구현 하려는 객체의 특성에 맞게 `toString()`을 구현해 주는 것이 가독성에 좋다

- `toString()`으로 제공한 정보를 객체로 변환해주는 정적 팩토리 메소드를 제공할 수 있다.
```java
public static PhoneNumber of(String phoneNumberString) {
            String[] split = phoneNumberString.split("-");
            PhoneNumber phoneNumber = new PhoneNumber(
            Short.parseShort(split[0]),
            Short.parseShort(split[1]),
            Short.parseShort(split[2]));
            return phoneNumber;
        }
```

```java
PhoneNumber phoneNumber = PhoneNumber.of("707-867-5309");
        System.out.println(phoneNumber);
        System.out.println(jenny.equals(phoneNumber));
        System.out.println(jenny.hashCode() == phoneNumber.hashCode());
```
- 하지만 각각의 정보를 받을 수 있도록 getter 메소드를 제공하는 것이 사용자 입장에서 편하다.

#### toString()을 재정의할 때 주의할 점
- 책에서는 toString()을 재정의할 때는 객체 내부 정보를 모두 출력하는 것이 좋다고 한다.
- 하지만, 개인정보가 민감한 경우 외부에 공개할 수 있는 데이터만 `toString()`에 재정의 해야 한다
  - 사용자의 주문 내역 등 로그 정보가 탈취될 수도 있다는 가정하에 로그조차 남기면 안되는 회사도 있다

- 유틸리티 클래스는 `toString()`을 사용할 이유가 없고,Enum은 아래 코드처럼 `toString()` 이미 정의 되어 있으므로 `toString()`을 재정의하지 않아도 된다.
```java
public abstract class Enum<E extends Enum<E>> implements Comparable<E>, Serializable {

    private final String name;

    public String toString() {
        return name;
    }
}
```
