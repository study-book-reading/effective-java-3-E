## 아이템 try-finally보다는 try-with-resources를 사용하라


자바 라이브러리에는 close 메서드를 호출해 직접 닫아줘야 하는 자원이 많음.  
InputStream, OutputStream, java.sql.Connection 등이 있다.  

자원 닫기는 클라이언트가 놓치기 쉬워서 예측할 수 없는 성능 문제로 이어지기도 함.  

이런 자원 중 상당수가 안전망으로 finalizer를 활용하고 있지만 finalizer는 그리 믿을만하지 못하다.(이미 deprecated 돼서 삭제 예정)  

전통적으로 자원이 제대로 닫힘을 보장하는 수단으로 try-finally가 쓰였음.  

```java
static String firstLineOfFile(String path) throws IOException {
  BufferedReader br = new BufferedReader(new FileReader(path));
  try {
    return br.readLine();
  } finally {
    br.close();
  }
}
```

자원을 하나 더 사용한다면 얼마나 복잡해질까?

```java
static void copy(String src, String dst) throws IOException {
  InputStream in = new FileInputStream(src);
  try {
    OutputStream out = new FileOutputStream(dst);
    try {
      byte[] buf = new byte[BUFFER_SIZE];
      int n;
      while ((n = in.read(buf)) >= 0)
    } finally {
      out.close(); //복잡해져서 close를 실수해버렸다.  
    }
  } finally {
    in.close();
  }
}
```

사실 2007년 당시 자바 라이브러리에서 close 메서드를 
제대로 구현한 비율은 겨우 1/3정도다. (이게 말이됨?)  

#### try-finally를 사용하면 안되는 이유

- 여러 자원 사용시 코드가 복잡해진다.
- 두 번째 예외가 첫 번째 예외를 삼킬 경우 디버깅이 어렵다.
- GC가 리소스를 회수하기 전에 프로그램이 close메서드를 호출하여 리소스를 OS로 다시 반납하지 못한다면, 리소스를 해제하는 데 필요한 정보가 손실되어 리소스 누수가 있을 수 있음.(운영체제는 여전히 사용중인 것으로 간주)

이러한 문제들은 자바7의 **try-with-resources**로 해결 가능.

---
### try-with-resources란?

하나 또는 그 이상의 자원을 선언하는 try구문.

여기서 말하는 자원이란 프로그램(OS접근)사용 후 그 프로그램과 함께 닫혀야 하는 객체를 말함.  
try-with-resources 구문은 그 구문의 끝에 해당 자원의 닫힘을 보장.  

이 구조를 사용하려면 해당 자원이 AutoCloseable 인터페이스를 구현해야 함.  
AutoCloseable 인터페이스는 Closeable을 상속하고 있고, 단순히 void를 반환하는 close 메서드 하나만 덩그러니 정의한 인터페이스다.  

- try-with-resources 사용
```java
static String firstLineOfFile(String path) throws IOException {
  try (BufferedReader br = new BufferedReader(new FileReader(path))) {
    return br.readLine();
  }
}
```
- 두 개 자원 사용  
(세미콜론으로 구분되어 사용)  
(뒤따라 오는 코드 블록이 정상적으로 종료되거나 예외로 인해 종료되면 객체들의 close 메서드가 자동으로 반대의 순서대로 호출됨.)
```java
static void copy(String src, String dst) throws IOException {
  try (InputStream in = new FileInputStream(src);
        OutputStream out = new FileOutputStream(dst)) {
    byte[] buf = new byte[BUFFER_SIZE];
    int n;
    while((n = in.read(buf)) >= 0)
      out.write(buf, 0, n);
  }
}
//out먼저 close를 호출하고, in이 close됨.
```

코드가 짧아지고 가독성이 좋아졌다.  

firstLineOfFile 메서드에서 readLine과 close호출 양쪽에서 예외가 발생하면, close에서 발생한 예외는 숨겨지고 readLine에서 발생한 예외가 기록됨.  
이처럼 실전에서는 프로그래머에게 보여줄 예외 하나만 보존되고 여러 다른 예외는 숨겨질 수도 있다.  
스택 추적 내역에 '숨겨짐(suppressed)' 꼬리표를 달고 출력된다.  
자바7에서 Throwable에 추가된 getSuppressed 메서드를 이용하면 프로그램 코드에서 가져올 수도 있음.  

보통의 try-finally처럼 try-with-resources도 catch 절을 사용할 수 있음.  
catch절 덕분에 try문 중첩이 필요없이 다수의 예외 처리 가능.  

```java
static String firstLineOfFile(String path, String defaultVal) {
  try (BufferedReader br = new BufferedReader(new FileReader(path))) {
    return br.readLine();
  } catch (IOException e) {
    return defaultVal;
  }
}
```

---
#### 결론
자원회수시 try-with-resources를 사용하자.  
코드 가독성에도 좋고 예외 정보도 훨씬 유용하다.  
정확하고 쉽게 자원을 회수할 수 있다.  
(try-finally는 자원회수시 실용적이지 못하다.)

[참조]
- https://youn0111.tistory.com/35?category=994087
- https://peterica.tistory.com/305
- https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html