## 아이템 09 try-finally 보다 try-with-resources를 사용하라_김우철

### 개요

- 자바에는 close를 호출해 직접 닫아줘야 하는 자원이 있다.
  - ex) InputStream, OutputStream, java.sql.Connection

### try-finally

- 예외가 발생하더라도 전통적으로 자원이 제대로 닫힘을 보장하는 수단인 try-finally가 쓰였다.

```java
public static String inputString() throws IOException {
    BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}
```

- finally 블록은 try, catch 블록이 끝난 뒤 실행할 로직을 정의해 주는 블록이다. 
- 따라서 IOException이 발생하게 되더라도 상위 메서드로 IOException 객체를 던져준 뒤 finally 메서드를 실행해 자원을 종료하게 된다.
- 그렇다면 만약 close를 여러 번 호출해야 하는 상황이 오면 어떻게 될까?

```java
public static void inputAndWriteString() throws IOException {
    BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
    try {
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(System.out));
        try {
            String line = br.readLine();
            bw.write(line);
        } finally {
            bw.close();
        }
    } finally {
        br.close();
    }
}
```

- 코드가 굉장히 지저분해지게 될 수 있다.
- **사실 가장 큰 문제는 예외를 잡아 먹는 것이다.**

```java
public class Application {
    public static void main(String[] args) {
        try {
            check();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void check() {
        try {
            throw new IllegalArgumentException();
        } finally {
            throw new NullPointerException();
        }
    }
}
```

```java
실행 결과
java.lang.NullPointerException
at Application.check(Application.java:14)
at Application.main(Application.java:4
```

- 가장 나중에 발생한 NullPointerException()만 보인다
- 첫번째 예외(IllegalArgumentException)가 먹혔다
- 가장 처음에 발생한 예외가 디버깅시 중요한데 이처럼 예외를 잡아 먹히면 디버깅이 어려워진다
- 이를 방지하기 위해선 finally 에서도 try-catch 블록을 사용하면 된다. 그러나 코드가 지저분 해진다
```java
public class Application {
    public static void main(String[] args) {
        try {
            check();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void check() {
        try {
            throw new IllegalArgumentException();
        } finally {
            try {
                throw new NullPointerException();
            } catch (NullPointerException e) {
                System.out.println("Caught NullPointerException in finally block");
                e.printStackTrace();
            }
        }
    }
}

```

### 해결책: try-with-resources

- 이러한 try-finally 방식의 단점을 보완하기 위해 자바 7 버전부터는 try-with-resources가 도입되었다. 
- try-with-resources를 사용하기 위해서는 사용하는 자원이 AutoCloseable 인터페이스를 구현해야 한다.

```java
public interface AutoCloseable {
    void close() throws Exception;
}
```

- AutoCloseable 인터페이스는 단지 close 메서드 하나만을 정의해 놓은 간단한 인터페이스이다. 
- 자바 라이브러리와 서드파티 라이브러리의 수많은 클래스와 인터페이스는 이미 AutoCloseable을 구현하거나 확장해두었다.
- 따라서, close가 필요한 자원 클래스를 커스텀 할 일이 있다면 AutoCloseable을 반드시 구현할 것을 권장한다.

```java
public class TopLine {
    // 코드 9-1 try-finally - 더 이상 자원을 회수하는 최선의 방책이 아니다! (47쪽)
    static String firstLineOfFile(String path) throws IOException {
        try (BufferedReader br = new BadBufferedReader(new FileReader(path))) {
            return br.readLine();
        }
    }

    public static void main(String[] args) throws IOException {
        System.out.println(firstLineOfFile("pom.xml"));
    }
}
```

- 코드의 가독성 증가
- 가장 처음 발생한 예외가 먼저 보이고 후속 예외도 다 보인다
    - 이렇게 숨겨진 예외는 무시되는 것이 아니라, suppressed 상태가 되어 stackTrace 시 숨겨졌다는 메시지로 출력된다.
    - suppressed 상태가 된 예외는 자바 7부터 도입된 getSuppressed 메서드를 통해 가져와서 사용할 수 있다. 
    - 다음의 테스트 코드를 보자.

```java
public class Application {

    public static void main(String[] args) {
        try {
            check();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void check() throws Exception {
        try (IllegalArgumentExceptionThrower thrower = new IllegalArgumentExceptionThrower()) {
            throw new NullPointerException();
        }
    }

    static class IllegalArgumentExceptionThrower implements AutoCloseable {
        @Override
        public void close() throws Exception {
            throw new IllegalArgumentException();
        }
    }
}
```

```java
실행 결과
java.lang.NullPointerException
at Application.check(Application.java:17)
at Application.main(Application.java:9)

Suppressed:java.lang.IllegalArgumentException
at Application$IllegalArgumentExceptionThrower.close(Application.java:24)
at Application.check(Application.java:16)
...1more
```

- 직접 throw 한 NullPointerException이 catch되어 출력된다. 
- 이 때 close 메서드에서 던지는 IllegalArgumentException은 Suppressed: 태그 뒤로 출력되는 것을 볼 수 있다.

### try-with-resources와 catch

- try-with-resources 구조 역시 기존 try-finally 처럼 catch를 병용해서 사용할 수 있다.

```java
public static String inputString() {
    try (BufferedReader br = new BufferedReader(new InputStreamReader(System.in))) {
        return br.readLine();
    } catch (IOException e) {
        return "IOException 발생";
    }
}
```
