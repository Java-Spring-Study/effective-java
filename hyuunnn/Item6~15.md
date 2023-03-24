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

TODO: `finalizer` 공격 코드 만들어보기... 이어서 내용 추가하기..

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

## Item 10 - equals는 일반 규약을 지켜 재정의하라

값 객체에 들어있는 값이 같은지 비교할 때 `equals`를 재정의한다. 같은 객체인지 확인하는 코드는 `Object`의 `equals`가 사용되며, 일반적으로 사용하는 자바 라이브러리들은 대부분 이미 정의되어 있다. (`Set`은 `AbstractSet`이 구현한 `equals` 사용, `List`는 `AbstractList`가 구현한 `equals` 사용 등)

`equals` 사용할 때 규약에 맞게 구현되어 있다고 가정하고 동작하기 때문에 규칙을 지켜서 신중하게 만들어야 한다.

### 반사성(reflexivity)

`null`이 아닌 참조 값 x에 대해 `x.equals(x)`는 true다.

즉 자기 자신을 비교했을 때 true가 나와야한다는 의미이다.

### 대칭성(symmetry)

`null`이 아닌 참조 값 x에 대해 `x.equals(y)`가 true면 `y.equals(x)`도 true다.

```java
public class CaseInsensitiveString {
  private final String s;

  public CaseInsensitiveString(String s) {
    this.s = Objects.requireNonNull(s);
  }

  @Override
  public boolean equals(Object o) {
    if (o instanceof CaseInsensitiveString)
      return s.equalsIgnoreCase(
          ((CaseInsensitiveString) o).s);

    if (o instanceof String)
      return s.equalsIgnoreCase((String) o);
    return false;
  }
}
```

```java
CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String s = "polish";

System.out.println(cis.equals(s)); // true
System.out.println(s.equals(cis)); // false
```

출력 결과를 보면 대칭성을 위반하고 있다. 그 이유는 `equals` 구현에 문제가 있는데, `CaseInsensitiveString` 타입이 아닐 때 `String` 타입인지 1번 더 확인하고 있다.

이때 `s.equals(cis)`에서는 `CaseInsensitiveString`의 존재를 모르기 때문에 `false`를 반환한다. 즉 `CaseInsensitiveString`과 `String`을 연동하면 안되며 둘은 다른 객체로 분리하게 만들어야 한다.

```java
@Override
public boolean equals(Object o) {
    return o instanceof CaseInsensitiveString &&
        ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
```

### 추이성(transitivity)

`null`이 아닌 참조 값 x에 대해 `x.equals(y)`, `y.equals(z)`가 true면 `x.equals(z)`도 true다.

TODO: 설명

### 일관성(consistency)

`null`이 아닌 참조 값 x에 대해 `x.equals(y)`를 호출했을 때 항상 일관되게(true면 true, false면 false) 출력되어야 한다. 

같은 두 객체가 수정되지 않는 한 항상 일관된 값을 출력해야 한다.

책에서는 `java.net.URL`의 `equals`에서 주어진 URL의 IP 주소를 이용해 비교한다고 했을 때 네트워크 상태에 따라 항상 같다고 보장할 수 없는 것이다. (`getHostAddress()`의 값을 받지 못하는 상태)

그래서 항상 존재하는 값, 객체만을 사용하여 결정적인 계산을 수행해야 한다.

### null-아님

`null`이 아닌 참조 값 x에 대해 `x.equals(null)`은 false다.

### 정리

1. `==` 연산자를 사용하여 입력이 자기 자신인지 확인 후 그렇다면 `true`를 반환한다. (반사성)

2. `instanceof` 연산자로 입력이 올바른 타입인지 확인한다. 

3. 입력을 올바른 타입으로 형변환한다. (`instanceof`에 의해 올바른 타입으로 형변환)

4. 입력 객체와 자기 자신의 대응되는 필드들이 모두 일치하는지 하나씩 검사한다. 

### 주의

1. `equals`를 재정의할 때 hashCode도 재정의하자.

2. `equals`의 매개변수 타입은 반드시 `Object`여야 한다.
    ```java
    public boolean equals(MyClass o) {
        ...
    }
    ```
    위 코드는 `equals`를 재정의가 아닌 다중정의를 했다고 볼 수 있다. (`MyClass` 타입이 아닌 객체가 들어왔을 때 새롭게 정의한 `equals`는 동작하지 않는다.)


## Item 11 - equals를 재정의하려거든 hashCode도 재정의하라

`equals`는 물리적으로 다른 두 객체를 논리적으로 봤을 때 같다고 할 수 있다. 하지만 hashCode는 다른 객체라고 판단하여 다른 값을 반환한다.

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "제니");
System.out.println(m.get(new PhoneNumber(707, 867, 5309)));
```

위 코드의 결과는 `null`이 나오는데 그 이유는 값을 넣을 때, 꺼내올 때 `PhoneNumber`를 새롭게 생성하여 다른 객체이기 때문이다.

그래서 `equals`를 재정의해야 하며, `HashMap`은 hashCode 값이 다르게 나오면 동치성 비교를 시도조차 하지 않도록 최적화되어 있기 때문에 같은 hashCode 값을 반환할 수 있게 재정의해야 한다.

TODO: 더 자세하게 적기..

## Item 12 - toString을 항상 재정의하라

```java
// Object.java

/** 
 *  ...
 *  It is recommended that all subclasses override this method. (모든 하위 클래스에서 이 메서드를 재정의하라.)
 *  ...
 */
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```

`Object`에 구현된 기본 `toString` 메서드는 의미 없는 값을 출력하기 때문에 재정의해야 한다. (주석에도 재정의하라는 내용이 존재한다.)

또한 `toString`을 정의할 때 주요 정보 모두를 반환하는게 좋으며, 객체의 데이터가 방대하다면 요약하여 출력해야 한다. (ex: 전화번호부 (총 12341234개))

```java
@Override public String toString() {
    return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
}
```

```text
Assertion failure: expected {abc, 123}, but was {abc, 123}.
```

위 테스트 실패 메세지를 보면 `toString`에 모든 정보를 반환하지 않아서 어떤 값이 틀렸는지 확인할 수 없다.

## Item 13 - clone 재정의는 주의해서 진행하라

TODO

## Item 14 - Comparable을 구현할지 고려하라

sgn은 부호 함수(signum function)이며 값이 음수, 0, 양수일 때 -1, 0, 1을 반환하도록 정의했다고 가정한다.

* `sgn(x.compareTo(y)) == -sgin(y.compareTo(x))`여야 한다.

* `(x.compareTo(y) > 0 && y.compareTo(z) > 0)`이면 `x.compareTo(z) > 0`이다.

* `x.compareTo(y) == 0`이며 `sgn(x.compareTo(z)) == sgn(y.compareTo(z))`이다.

* `(x.compareTo(y) == 0) == (x.equals(y))`여야 한다. (필수는 아니지만 지키는게 좋다.)
    * `compareTo`와 `equals`의 결과를 일관되게 함으로써 타 객체와 비교할 때 일관성을 유지할 수 있다.

TODO: `Integer.compare` 구현 코드 참고하여 이후 설명 추가하기

## Item 15 - 클래스와 멤버의 접근 권한을 최소화하라

API를 통해서만 접근할 수 있게 접근 권한을 최소화하고 내부 구현을 숨기는 것이 중요하다. (캡슐화)

접근 권한이 열려있으면 그만큼 외부로부터 의도하지 않은 동작을 수행할 가능성이 높아지기 때문이다.

### 방법

#### public일 필요가 없는 클래스를 package-private으로 권한을 좁힌다.**

#### 한 클래스에서만 사용하는 package-private 탑레벨 클래스라면 클래스 안에 private static으로 중첩하자. (해당 클래스를 호출할 수 있는 구멍이 하나로 줄어들게 된다.)**

#### 공개할(public) API를 세심히 설계한 후 그 외의 모든 멤버는 private으로 만들자.**

#### public 클래스의 인스턴스 필드는 되도록 public이 아니어야 한다.**

필드가 가변 객체를 참조하거나 final이 아닌 필드를 public으로 선언하면 외부에서 값을 가져올 수 있고, 결국 불변을 보장할 수 없게 된다.

#### public static final 배열 필드를 두거나 getter 메서드를 제공해서는 안된다.

```java
public static final Thing[] VALUES = { ... };
``` 

위 코드는 final을 사용했더라도 배열이기 때문에 내부 데이터를 변경할 수 있어서 불변하지 않다.

첫 번째는 이전의 VALUES 변수를 private으로 권한을 최소화하고, VALUES 변수를 읽어올 때는 `unmodifiableList`를 사용하여 수정을 막는 방법이 있다.

```java
private static final Thing[] PRIVATE_VALUES = { ... };
public static final List<Thing> VALUES = 
    Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```

두 번째는 복사본을 반환하는 public 메서드를 추가하는 방법이다.

```java
private static final Thing[] PRIVATE_VALUES = { ... };
public static final Thing[] values() {
    return PRIVATE_VALUES.clone();
}
```

자바 9에 추가된 모듈 시스템을 사용하면 패키지들을 관리하기 때문에 패키지의 공개 유무를 설정함에 따라서 접근을 제어할 수 있다.

하지만 저자는 이런 개념이 일반적이지 않기 때문에 널리 사용된다고 예측하기에는 아직 이른 감이 있다고 하며 꼭 필요한 경우가 아니라면 사용하지 않는게 좋다고 한다.
