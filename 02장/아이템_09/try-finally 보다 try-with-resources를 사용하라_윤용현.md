# Try-finally 보다는 Try-with-resources를 사용하라

---

## Try-finally는 더 이상 자원을 회수하는 최선의 방책이 아니다

전통적으로 자원이 제대로 닫힘을 보장하는 수단으로 Try-finally가 쓰였다.

```java
static String firstLineOfFIle(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br. readLine();
    } finally {
        br.close();
    ｝
}
```

자원을 하나 더 사용한다면? 아래 코드를 보면 알 수 있듯 try 문이 중첩되어 코드가 더러워진다.

```java
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}
```

뿐만 아니라, 예외는 try 블록과 finally 블록 모두에서 발생할 수 있다.

- 기기에 물리적인 문제가 생겨 firstLineOfFile 메서드 안의 readLine 메서드가 예외를 던지면, close 메서드도 예외를 던질 것이다.
- 이 경우 readLine에서 발생한 예외는 숨겨지고, close에서 발생한 예외가 기록된다.
- 이런 상황이라면 두 번째 예외가 첫 번째 예외를 완전히 집어삼켜 버린다. 스택 추적 내역에는 첫 번째 예외에 관한 정보가 남지 않게 된다.

<br><br><br>

## Try-with-resources를 사용하자

```java
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
}
```

- **이 구조를 사용하려면 해당 자원이 AutoCloseable 인터페이스를 구현해야 한다.**
- 복수의 자원을 처리함에 훨씬 읽기 수월할 뿐만 아니라 문제를 진단하기도 쉽다.

<br><br><br>

## Try-with-resources를 catch 절과 함께 쓰는 경우

catch 절을 사용하여 try 문을 더 중첩하지 않고도 다수의 예외를 처리할 수 있다.

```java
static String firstLineOfFile(String path, String defaultval) {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return defaultVal;
    ｝
}
```

<br><br><br>

## 요약

- 꼭 회수해야 하는 자원을 다룰 때는 try-finally 말고, try-with-resources를 사용하자.
- 코드는 더 짧고 분명해지고, 만들어지는 예외 정보도 훨씬 유용하다.
- try-finally로 작성하면 실용적이지 못할 만큼 코드가 지저분해지는 경우라도, try-with-resources로는 정확하고 쉽게 자원을 회수할 수 있다.