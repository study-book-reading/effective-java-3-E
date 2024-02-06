## 아이템 11 equals를 재정의하려거든 hashCode도 재정의하라

### 개요

- Java의 모든 객체는 hashCode 메서드를 가진다.
- Java에서는 HashMap, HashSet 등 hash를 활용한 Collection들을 자주 볼 수 있는데 모두 hashCode를 기반으로 하고 있다.

### hashCode란?

- hashCode란 객체의 주소값을 변환하여 생성한 객체의 고유한 정수값이다.

```java
public native int hashCode();
```

- Object#hashCode구현
    - native call을 통해 해당 객체의 메모리 해시 구조를 가져온다.

### hashCode의 3가지 규약

- equals를 재정의 했다면 hashCode도 반드시 재정의해야한다.
- 그렇지 않으면 hashCode의 일반 규약을 어기게 된다.
- 이 경우 HashMap, HashSet과 같이 hashCode를 기반으로 하는 클래스의 원소로 사용하는 경우 예상치 못한 버그를 발견할 수 있다.

### Object#hashCode 규약 3가지

1. equals가 변경되지 않았다면 애플리케이션이 실행되는 동안 그 객체의 hashCode는 항상 같은 값을 반환해야한다. 단, 애플리케이션이 재실행하는 경우 값이 달라질 수는 있다.
2. equals가 두 객체를 같다고 판단했다면, 두 객체의 hashCode는 똑같은 값을 반환해야한다.
3. equals가 두 객체를 다르다고 판단하더라도 두 객체의 hashCode가 서로 다른 값을 반환할 필요는 없다. 단, 다른 객체에 대해서는 다른 값을 반환해야 해시 테이블의 성능이 좋아진다.

- 두번째 규약인 equals를 재정의하여 논리적으로 같다고 정의했을때 만약 hashCode를 기본 구현으로 사용한다면 서로 다른 값을 반환한다.
- 즉, equals 재정의로 2번 규약을 어기게되는 것이다.
- 특히 hashCode는 hash를 기반으로 하는 컬랙션인 HashMap과 HashSet 등에 매우 중요하다.

```java
@Override
public int hashCode(){
        return 1;
        }
```

- 만약 위와 같이 모든 인스턴스에 같은hashCode를 가진다면 어떻게 될까?
- 모든 객체에게 똑같은 hashCode만을 전달하기 때문에 모든 객체가 해시테이블의 버킷 하나에 담긴다.
    - 링크드 리스트처럼 동작한다.
- 때문에 해시테이블의 성능이 매우 떨어지게 된다.
    - 평균 수행 시간이 O(1) → O(n)으로 느려진다.
    - 이유는 해시 충돌이 유무 때문인데 hashCode가 같다면 모든 인스턴스에서 해시충돌이 일어날 것이다.
    - 이때 HashMap에서 해시 충돌을 피하기 위한 구현인 Separate Chaining에 따라 하나의 버킷에 모든 데이터가 삽입되기 때문이다.

```java
public static void main(String[]args){
        Map<PhoneNumber, String> m=new HashMap<>();
        m.put(**new PhoneNumber**(707,867,5309),"Jenny"); // 1

        System.out.println(**m.get(new PhoneNumber(707,867,5309)**)); // 2, null 반환
        }
```
- PhoneNumber는 hashCode를 재정의 하지 않았을 때 //2 는 null을 반환한다.
- PhoneNumber 클래스는 hashCode를 재정의하지 않아 **논리적 동치(equals)**인 두 객체가 서로 다른 해시코드를 반환한다. (두번째 규약 위배)
- 두 인스턴스 **같은 버킷(hashCode())**에 있더라도 HashMap은 hashCode가 다른 entry끼리는 동치성 비교를 시도초자 하지 않도록 최적화되어 있기 때문에 null이 반환된다.

### 규약에 맞는 hashCode 메서드 짜기
- 좋은 해시 함수라면 서로 다른 인스턴스끼리 다른 해시코드를 반환해야한다.
- 가장 이상적인 해시 함수는 주어진 인스턴스들을 32비트 정수 범위에 균일하게 분배해야한다.

```java
// 코드 11-2 전형적인 hashCode 메서드 (70쪽)
    @Override public int hashCode() {
        int result = Short.hashCode(areaCode); // 1
        result = 31 * result + Short.hashCode(prefix); // 2
        result = 31 * result + Short.hashCode(lineNum); // 3
        return result;
    }
```
- 프리미티브 타입이면 해당 래퍼 타입의 hashCode 메소드를 사용
  - Double#hashCode 등
- 프리미티브 타입이 아닌 경우 해당 클래스가 가지고 있는 hashCode 메소드 사용
  - Point#hashCode
- Array는 Arrays의 hashCode 메소드를 사용
- 31일 곱하는 이유
  - 짝수를 써서 연산을 하면 뒤에 0이 채워지면서 숫자가 왼쪽으로 밀리면서 조금 많이 날아갈 수 있다.
  - 옛날에 개발자 두명이 사전에 들어 있는 모든 단어를 해시 해서 테스트한 결과 31을 썼을 때 해시 충돌이 가장 적게 나와서 31을 사용
  - 32비트 정수 안에서 표현할 수 있는 값 중 가장 적절한 분포도를 보여 주는 값이 31

```java
public final class Objects {
	public static int hash(Object... values) {
	    return Arrays.hashCode(values);
	}
}

public final class Arrays {
	public static int hashCode(Object a[]) {
      if (a == null){
        return 0;
      }
      int result = 1;
      for (Object element : a)
          result = 31 * result + (element == null ? 0 : element.hashCode());
      return result;
  }
}
```
- Objects#hash 구현

```java
    @Override public int hashCode() {
        return Objects.hash(lineNum, prefix, areaCode);
    }
```
- 인텔리제이가 만들어주는 hashCode

```java
@EqualsAndHashCode
```
- Lombok의 @EqualsAndHashCode
- 가장 편하고 신뢰할 수 있는 방법

```java
// 해시코드를 지연 초기화하는 hashCode 메서드 - 스레드 안정성까지 고려해야 한다. (71쪽)
    private volatile int hashCode; // 자동으로 0으로 초기화된다.

    @Override public int hashCode() {
        if (this.hashCode != 0) {
            return hashCode;
        }

        synchronized (this) {
            int result = hashCode;
            if (result == 0) {
                result = Short.hashCode(areaCode);
                result = 31 * result + Short.hashCode(prefix);
                result = 31 * result + Short.hashCode(lineNum);
                this.hashCode = result;
            }
            return result;
        }
    }
```
- 필드 hashCode가 0이라면 hashCode 값을 초기화하고 0이 아니라면 이미 계산된 hashCode를 반환하는 방식이다.
- 성능에 민감한 경우라면 hashCode를 lazy loading하는 방식으로 구현할 수 있다.
- 해시코드 메소드 안에 여러 스레드가 동시에 들어와 값이 틀릴 수도 있다.
- 하지만 스레드 안정성을 고려해야하고 코드가 복잡해지기 때문에 성능이 중요하지 않다면 굳이 사용할 필요는 없다.

### 주의사항
- 성능을 위해 해시코드 계산할 때 equals에 명시한 모든 필드를 생략해서는 안된다.
- hashcode 값의 생성 규칙을 API 사용자에게 자세히 공표해서는 안된다.
