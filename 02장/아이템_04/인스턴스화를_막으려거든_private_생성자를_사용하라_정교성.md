## 아이템 04 인스턴스화를 막으려거든 private 생성자를 사용하라

정적 메서드와 정적 필드만을 담은 클래스를 만들 경우  

예시) 
- java.lang.Math, java.util.Arrays처럼 기본 타입 값이나 배열 관련 메서드들을 모아놓음.  
- java.util.Collections처럼 특정 인터페이스 구현 객체를 생성해주는 정적 메서드(혹인 팩터리)를 모아놓음.
- final 클래스와 관련한 메서드들을 모아놓음. final 클래스를 상속해서 하위 클래스에 메서드를 넣는 건 불가능하기 때문

정적 멤버만 담은 유틸리티 클래스는 인스턴스로 만들어 사용하도록 설계된 게 아니기 때문에 private 생성자를 명시하여 인스턴스화를 막아라.

```java
public class UtilityClass {
    //인스턴스 방지용 <- 주석 명시를 통해 의도를 명확히 함
    private UtilityClass() {    //private으로 하위 클래스가 접근 불가하여 상속 불가
        throw new AssertionError(); //클래스 내부에서 생성자 호출 예방
    }
}
```






