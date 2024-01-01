## 아이템 04 인스턴스화를 막으려거든 private 생성자를 사용하라

### 개요

java.lang.Math, java.util.Arrays, util.Collections 등 정적 메서드와 정적 필드만을 담은 클래스는 
객체 지향적이지 않은 방식이긴 하지만 나름대로 쓰임새가 있다.

**정적 멤버만 담은 유틸리티 클래스는 인스턴스로 만들어 쓰려고 설계한 게 아니다.**
**하지만 생성자를  명시하지 않으면 컴파일러가 자동으로 기본 생성자를 만들어준다.**

static 메서드는 클래스를 통해 바로 접근이 가능하다. 그래서 static 메서드를 인스턴스 변수를 통해 접근하는 방법은 매우 안좋은 방법이다.

```java
public class StringUtils {
    public static boolean isBlank(String str) {
        return Objects.isNull(str) || str.isBlank();
    }
}
```

```java
public class App {
    public static void main(String[] args) {
        // 클래스를 통해 접근
        String str1 = " ";
        System.out.println("str1 문자열이 비어있는가?? : " + StringUtils.isBlank(str1));
    
        // 인스턴스를 통해 접근
        StringUtils instance = new StringUtils();
        String str2 = "a";
        System.out.println("str2 문자열이 비어있는가?? : " + instance.isBlank(str2));
    }
}

// 출력 결과
// 문자열이 비어있는가?? : true
// 문자열이 비어있는가?? : false
```

위 예제에서 인스턴스를 통해 static 메서드를 호출하는 방법의 단점은 2가지가 있다.

- 해당 메서드가 static 메서드인지 파악이 잘 안된다.
- 클래스로 바로 접근 가능함에도 쓸데없이 객체를 생성하게 된다.
 
이러한 단점들로 인해, 보통 유틸성 클래스는 생성자 사용을 막는다. 이제 생성자 사용을 막는 방법에 대해 알아보자.

### 생성자 사용 막는 방법

#### 첫번째 방법 : 추상 클래스화로 인스턴스 생성 막기 (비추천 방법)

이 방법은 스프링 API에서 생성자 사용을 막을 때 주로 사용하는 방법이다. 하지만 이 방법은 생성자 사용을 완벽히 막아주지 못한다.
해당 추상 클래스를 상속받은 하위 클래스를 생성하게 되면, 그 부모인 추상 클래스의 생성자도 같이 동작하게 된다.
생성자를 통한 객체 생성은 막을 수 있지만, 상속은 할 수 있다.
때문에, 그 하위 클래스를 통해 상위 클래스의 static 메서드를 호출할 수 있게 된다.

```java
// 추상 클래스화
public abstract class StringUtils {
    public static boolean isBlank(String str) {
        return Objects.isNull(str) || str.isBlank();
    }
}
```

```java
public class StringServeUtils extends StringUtils {
}
```

```java
public class App {
    public static void main(String[] args) {
        // 유틸 클래스를 상속받은 하위 클래스를 통해 static 메서드 호출
        StringUtils instance = new StringServeUtils();
        String str = "a";
        System.out.println("str 문자열이 비어있는가?? : " + instance.isBlank(str));
    }
}
     
// 출력 결과
// str 문자열이 비어있는가?? : true
```
다른 이용자가 보기에 상속해서 사용하라는 뜻으로 오해할 수 있다.
추상 클래스가 있으면 생성자를 막는 용도보다는 상속해서 사용하라는 용도로 생각하기 쉬워진다.

#### 두번째 방법 : 생성자를 private 으로 설정 (권장 방법)
private 생성자를 가지면, 해당 클래스 외의 곳에선 일반적으로 생성자를 통해 객체를 생성할 수 없다. 또한 상속도 막아준다. 
그리고 해당 클래스 안에서도 인스턴스화 하는 것을 막고 싶다면, private 생성자 구현부에 Error(에러)를 던져주면 될 것이다. 

참고로, 대처를 할 수 있는 예외(Exception)와는 다르게, 에러(Error)는 복구할 수 없는 심각한 오류라는 것을 나타낸다.
하지만 이에 대한 단점도 존재한다. 어차피 못 쓰는 생성자인데 작성이 되어있으니 직관적이지 못하다는 것이다. 
그래서 이 생성자에대한 주석을 명확히 달아둘 필요가 있다.

private 생성자 예제
```java
public class StringUtils {
    /**
     * 인스턴스를 생성할 수 없는 유틸 클래스입니다.
     */
    private StringUtils() {
        throw new AssertionError("해당 객체는 생성이 불가능합니다.");
    }
    
    public static boolean isBlank(String str) {
        return Objects.isNull(str) || str.isBlank();
    }
}
```

## 참고

- [2022-effective-java](https://github.com/woowacourse-study/2022-effective-java)
- [[자바, Java] 이펙티브 자바(Effective Java) - 아이템 04 인스턴스화를 막으려거든 private 생성자를 사용하라](https://tjdtls690.github.io/studycontents/java/2023-02-04-private_constructor/)
