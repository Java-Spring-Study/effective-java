## Item 6 - 불필요한 객체 생성을 피하라

### 생성자 호출로 인한 성능 저하

```java
String s = new String("bikini");
```

위와 같은 코드를 실행했을 때 새로운 String 인스턴스가 계속 생성된다.

예를 들어 위와 같은 코드가 메서드 안에서 여러번 호출된다면 그때마다 새로운 인스턴스가 생성되는 것이다.

```java
String s = "bikini";
```

이렇게 코드를 작성하면 하나의 String 인스턴스를 사용하며, 똑같은 문자를 받았을 때 같은 객체를 재사용한다.

```java
@Deprecated(since="9", forRemoval = true)
public Boolean(boolean value) {
    this.value = value;
}

@IntrinsicCandidate
public static Boolean valueOf(boolean b) {
    return (b ? TRUE : FALSE);
}
```

**정적 팩터리 메서드**도 불필요한 객체 생성을 하지 않기 때문에 성능 이점이 존재하며, Boolean에서도 생성자를 통한 호출을 deprecated로 지정했다.

**정규표현식**을 사용할 때도 불필요한 객체가 생성될 수 있다.

```java
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
            + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```

위 코드는 호출할 때마다 `Pattern` 인스턴스가 생성되며, 1번 사용 후 버려지기 때문에 가비지 컬렉션 대상이 된다.

특히 `Pattern`은 생성 비용이 높기 때문에 재사용하는 방법을 선택해야 한다.

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})"
        + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```

`ROMAN`이라는 변수를 만들어서 재사용하기 때문에 불필요한 객체가 생성되지 않는다.

하지만 `isRomanNumeral` 메서드를 아예 사용하지 않으면 불필요한 Pattern 인스턴스가 생성되었다고 생각할 수 있다. 

지연 초기화라는 방법이 있지만 저자는 권하지 않는다고 한다. (코드를 복잡하게 만드며, 성능은 크게 개선되지 않을 때가 많다고 한다.)

### 오토박싱으로 인한 성능 저하

```java
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++)
        sum += i;

    return sum;
}
```

sum에 값을 더할 때마다 Long 인스턴스를 생성 후 더하게 되며, 성능 저하의 원인이 된다. 

**박싱된 기본 타입보다는 기본 타입을 사용하고, 의도치 않은 오토박싱이 숨어들지 않도록 주의하자.** - p34

객체 생성은 비싸기 때문에 사용하지 말라는게 아니라 불필요한 생성을 피하라는 뜻이다.

## Item 7 - 다 쓴 객체 참조를 해제하라

자바는 가비지 컬렉터에 의해 자동으로 회수하기 때문에 메모리를 직접 관리할 필요가 없다.

하지만 가비지 컬렉터가 회수해주지 못하면 메모리에 계속 남아있기 때문에 **메모리 누수**가 발생할 수 있다.

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

위 코드는 스택을 구현한 코드인데 메모리 누수가 존재한다.

스택에서 꺼낸 객체를 더 이상 사용하지 않아도 가비지 컬렉터는 회수하지 않는다.

그 이유는 참조 객체가 하나라도 있으면 해당 객체 뿐만 아니라 참조하는 모든 객체를 회수하지 못한다.   

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;
    return result;
}
```

`null`을 넣어서 더 이상 사용하지 않음을 가비지 컬렉터한테 알려주면 된다.

하지만 저자는 **객체 참조를 `null` 처리하는 일은 예외적인 경우에만 사용**하라고 한다.

위에서 만들었던 Stack 클래스는 메모리를 직접 관리하기 때문에 가비지 컬렉터 입장에서 어떤 객체를 회수해야 하는지 알 방법이 없다. (**자기 메모리를 직접 관리하는 클래스라면 프로그래머는 항시 메모리 누수에 주의해야 한다.**)

유효 범위(scope)를 최소한으로 두거나 외부에서 참조하는 동안에만 필요하다면 `WeakHashMap`, 그 외에는 백그라운드 쓰레드, `LinkedHashMap`, `java.lang.ref` 등을 활용하는 방법이 있다.

## Item 8 - finalizer와 cleaner 사용을 피하라

자바는 현재 `finalizer`를 deprecated로 지정하고 `cleaner`를 대안으로 사용하고 있다고 한다. (**`cleaner`는 `finalizer`에 비해 덜 위험하지만 여전히 예측할 수 없고, 느리고, 불필요하다.**)

`finalizer`, `cleaner`는 즉시 수행된다는 보장이 없으며, 제때 실행되어야 하는 작업은 절대 할 수 없다. (언제 수행할지는 가비지 컬렉터 구현에 따라 달라진다.)

TODO: `finalizer` 공격 코드 만들어보기

## Item 9 - try-finally 보다는 try-with-resources를 사용하라

```java
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readline();
    } finally {
        br.close();
    }
}
```

IO 예외 처리를 할 때 `try` 안에서 동작을 수행하고, 마지막으로 `finally`에서 `close()`를 호출하는 코드를 자주 볼 수 있다.

만약 위 코드에서 `try` 안에 또 다른 예외 처리를 한다면 코드가 매우 지저분하게 된다.

그리고 `readLine()`을 호출하던 중에 예외가 발생한다면 `close()`를 무시하고 넘어갈 수도 있다.

```java
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```

하지만 try-with-resources를 사용하면 `AutoCloseable`에 의해 `close()`를 자동으로 호출하기 때문에 별도의 close 처리를 하지 않아도 된다. 파이썬의 with과 비슷하다고 볼 수 있다.
