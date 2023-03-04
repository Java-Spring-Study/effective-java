## Item 1 - 생성자 대신 정적 팩터리 메서드를 고려하라

### 이름을 가질 수 있다.

정적 팩터리 메서드를 사용하면 이름을 가지기 때문에 어떤 역할을 하는지 명확하게 표현할 수 있다.

하지만 생성자는 객체가 생성될 때 동작하는 메서드이며, 클래스명을 따라가기 때문에 어떤 동작을 하는지 표현할 수 없다.

또한 이름을 다르게 하여 같은 파라미터 타입으로 여러개의 메서드를 만들 수 있다.

### 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다.

```java
@IntrinsicCandidate
public static Boolean valueOf(boolean b) {
  return (b ? TRUE : FALSE);
}

public static Boolean valueOf(String s) {
  return parseBoolean(s) ? TRUE : FALSE;
}
```

static 메서드로 선언되었기 때문에 가질 수 있는 이점이며, 다양한 곳에서 자주 요청되는 유틸리티성 메서드에서 장점을 발휘한다.

### 반환 객체를 자유롭게 선택할 수 있는 유연한 개발이 가능하다.

```java
public abstract class EnumSet<E extends Enum<E>> extends AbstractSet<E> implements Cloneable, java.io.Serializable {
  ...
  /**
  * Creates an empty enum set with the specified element type.
  * ...
  */
  public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
    Enum<?>[] universe = getUniverse(elementType);
    if (universe == null)
      throw new ClassCastException(elementType + " not an enum");

    if (universe.length <= 64)
      return new RegularEnumSet<>(elementType, universe);
    else
      return new JumboEnumSet<>(elementType, universe);
  }
  ...
}
```

```java
class RegularEnumSet<E extends Enum<E>> extends EnumSet<E> {}
class JumboEnumSet<E extends Enum<E>> extends EnumSet<E> {}
```

사용자는 `EnumSet.noneOf`를 사용할 때 내부 구현이 어떻게 되어있는지 알 필요 없이 그저 API를 사용하면 된다. 개발자가 `noneOf`의 코드를 수정한다고 해도 사용자는 알 필요가 없다.

또한 반환 타입은 `EnumSet`을 상속하기만 하면 되기 때문에 **유연한 개발**이 가능하다. (위 코드는 원소의 개수에 따라서 다른 클래스를 반환하고 있다.)

### 정적 팩터리 메서드를 사용하기 위한 별도의 학습이 필요하다.

생성자는 객체를 생성했을 때 자동으로 동작하지만 정적 팩터리 메서드는 어떤 메서드들이 구현되어 있는지 찾아야 한다.

그렇기 때문에 API 문서화를 꼼꼼히 하고 메서드명을 정의할 때 규약을 따르는 것을 권장하고 있다.

* `from`: 매개변수를 하나 받아서 해당 타입의 인스턴스를 반환하는 형변환 메서드
  ```java
  public static Date from(Instant instant) {
    try {
      return new Date(instant.toEpochMilli());
    } catch (ArithmeticException ex) {
      throw new IllegalArgumentException(ex);
    }
  }
  ```
* `of`: 여러 매개변수를 받아서 적합한 타입의 인스턴스를 반환하는 집계 메서드
  ```java
  public static <E extends Enum<E>> EnumSet<E> of(E e) {
    EnumSet<E> result = noneOf(e.getDeclaringClass());
    result.add(e);
    return result;
  }
  ...

  public static <E extends Enum<E>> EnumSet<E> of(E e1, E e2, E e3, E e4, E e5)
  {
    EnumSet<E> result = noneOf(e1.getDeclaringClass());
    result.add(e1);
    result.add(e2);
    result.add(e3);
    result.add(e4);
    result.add(e5);
    return result;
  }

  @SafeVarargs
  public static <E extends Enum<E>> EnumSet<E> of(E first, E... rest) {
    EnumSet<E> result = noneOf(first.getDeclaringClass());
    result.add(first);
    for (E e : rest)
      result.add(e);
    return result;
  }
  ```
  번외로 `EnumSet.of`를 보면 6개부터 가변 매개변수로 동작하고, 5개 이하는 오버로딩 메서드가 동작한다. 이는 실행 속도를 높이기 위한 최적화된 방법이라고 이해하면 되겠다. (<a href="https://gist.github.com/hyuunnn/08c3d5788a9f06a34d199b977245a023">Bing GPT 답변</a>)

* `valueOf`: 매개변수를 받은 후 다른 타입으로 반환한다는 점에서 from, of와 유사함
  ```java
  public static String valueOf(Object obj) {
    return (obj == null) ? "null" : obj.toString();
  }
  ```
* `instance` 또는 `getInstance`: 매개변수를 받을 때 매개변수로 명시한 인스턴스를 반환하지만, 같은 인스턴스임을 보장하지는 않는다. (싱글톤 패턴에서 자주 등장하는 네이밍 규약)
  ```java
  StackWalker luke = StackWalker.getInstance(options);
  ```
* `create` 또는 `newInstance`: 새로운 인스턴스를 생성하여 반환함을 보장한다.
  ```java
  Object newArray = Array.newInstance(classObject, arrayLen);
  ```
* `getType`: 다른 클래스의 팩터리 메서드를 정의할 때 사용한다. (`getInstance`와 본질적인 기능은 같음)
  ```java
  FileStore fs = Files.getFilesStore(path);
  ```
* `newType`: 다른 클래스의 팩터리 메서드를 정의할 때 사용한다. (`newInstance`와 본질적인 기능은 같음)
  ```java
  BufferedReader br = Files.newBufferedReader(path);
  ```
* `type`: `getType`, `newType`의 간결한 버전
  ```java
  List<Complaint> litany = Collections.list(legacyLitany);
  ```

## Item 2 - 생성자에 매개변수가 많다면 빌더를 고려하라

### 점층적 생성자 패턴 (Telescoping Constructor Pattern)

```java
public class NutritionFacts {
  public NutritionFacts(int servingSize, int servings) {
    this(servingSize, servings, 0);
  }

  public NutritionFacts(int servingSize, int servings, int calories) {
    this(servingSize, servings, calories, 0);
  }

  public NutritionFacts(int servingSize, int servings, int calories, int fat) {
    this(servingSize, servings, calories, fat, 0);
  }
}
...
```

위와 같이 자기 자신을 호출하면서 늘려가는 방식을 **점층적 생성자 패턴**이라고 한다.

하지만 중간에 원치 않는 매개변수가 있어도 값을 지정(ex: 0)해야 하고, 매개변수가 많아지면 사용하기 불편하며 실수할 가능성도 높아진다.

### 자바빈즈 패턴 (JavaBeans Pattern)

```java
public class NutritionFacts {
  private int servingSize = -1;
  private int servings = -1;
  private int calories = 0;
  private int fat = 0;

  public NutritionFacts() { }
  public void setServingSize(int val) { servingSize = val; }
  public void setServings(int val) { servings = val; }
  public void setCalories(int val) { calories = val; }
  public void setFat(int val) { fat = val; }
}
```

위 코드는 이전 패턴과 다르게 기본값이 설정되어 있어서 불필요한 값을 설정할 필요가 없지만, 객체 하나를 만들 때마다 필요한 setter 메서드를 호출해야 하며 이로 인한 치명적인 단점으로 **불변하지 않다**는 것이다.

### 빌더 패턴 (Builder Pattern)

```java
public class NutritionFacts {
  private final int servingSize;
  private final int servings;
  private int calories;
  private int fat;

  public static class Builder {
    // 필수 매개변수
    private final int servingSize;
    private final int servings;

    // 선택 매개변수
    private int calories = 0;
    private int fat = 0;

    public Builder(int servingSize, int servings) {
      this.servingSize = servingSize;
      this.servings = servings;
    }

    public Builder calories(int val) {
      calories = val;
      return this;
    }

    public Builder fat(int val) {
      fat = val;
      return this;
    }

    public NutritionFacts build() {
      return new NutritionFacts(this);
    }
  }

  private NutritionFacts(Builder builder) {
    servingSize = builder.servingSize;
    servings = builder.servings;
    calories = builder.calories;
    fat = builder.fat;
  }
}
```

빌더 패턴은 위에서 설명한 2가지 패턴의 장점들을 가져왔다고 볼 수 있다. `NutritionFacts` 클래스는 `build()` 메서드를 호출할 때마다 새롭게 생성되기 때문에 불변하며, **자기 자신을 반환**하기 때문에 아래 코드처럼 chaining된 방식으로 코드를 작성할 수 있다. (기본 값을 유지하고 싶다면 chaining에 추가하지 않으면 된다.) 

또한 빌더 패턴은 정형화되어 있기 때문에 lombok의 `@Builder`를 사용하면 빌더 클래스를 만들 필요가 없어진다. 

책에서 API는 시간이 지날수록 기능이 추가됨에 따라 요구하는 매개변수가 많아지는 경향이 있기 때문에 성능이 민감한 상황이 아니라면 빌더 패턴 사용을 권장하고 있다.

```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8).calories(100).fat(3).build();
```

## Item 3 - private 생성자나 열거 타입으로 싱글턴임을 보증하라



## Item 4 - 인스턴스화를 막으려거든 private 생성자를 사용하라

```java
// Collections.java
public class Collections {
  // Suppresses default constructor, ensuring non-instantiability.
  private Collections() {
  }
  ...
}

// Arrays.java
public class Arrays {

  // Suppresses default constructor, ensuring non-instantiability.
  private Arrays() {}
  ...
}

// Math.java
public final class Math {
  /**
   * Don't let anyone instantiate this class.
   */
  private Math() {}
  ...
}
```

자바의 `java.util.Collections`, `java.util.Arrays` 같은 유틸리티 클래스는 인스턴스를 생성하여 사용할 수 있게 설계된 것이 아니다. 하지만 생성자를 명시하지 않으면 컴파일러가 자동으로 기본 생성자를 만들어주는데, 이때 사용자 입장에서 동작 의도를 파악함에 있어서 어려움이 있다.

그래서 private 생성자를 추가하면 인스턴스화를 막을 수 있다.

자바에서 지원하는 공식 유틸리티 클래스들을 보면 모두 private 생성자를 활용하고 있다.

## Item 5 - 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

```java
// 정적 유틸리티를 잘못 사용한 예 - 유연하지 않고 테스트하기 어렵다.
public class SpellChecker {
  private static final Lexicon dictionary = ...;
  private SpellChecker() {} // 객체 생성 방지

  public static boolean isValid(String word) { ... }
  public static List<String> suggestions(String typo) { ... }
}
```

```java
// 싱글톤을 잘못 사용한 예 - 유연하지 않고 테스트하기 어렵다.
public class SpellChecker {
  private final Lexicon dictionary = ...;

  private SpellChecker(...) {}
  public static SpellChecker INSTANCE = new SpellChecker(...);

  public boolean isValid(String word) { ... }
  public List<String> suggestions(String typo) { ... }
}
```

예를 들어 맞춤법 검사기를 만든다고 했을 때 사전에 의존하여 만들게 된다. 하지만 위의 코드처럼 만들게 되면 영어 사전에 의존하는 기능을 일본어 사전으로 변경 해야할 때 내부 코드를 수정해야 한다. 즉 자원을 **직접 명시**했기 때문에 유연하지 않은 것이다. 

```java
public class SpellChecker {
  private final Lexicon dictionary;

  public SpellChecker(Lexicon dictionary) {
    this.dictionary = Objects.requireNonNull(dictionary);
  }

  public boolean isValid(String word) { ... }
  public List<String> suggestions(String typo) { ... }
}
```

하지만 위의 코드는 생성자에 **의존 객체 주입**을 하는 방식이기 때문에 `Lexicon`을 상속만 한다면 어떤 클래스던지 들어올 수 있다. 즉 자유롭게 교체가 가능하며, 유연한 코드를 작성할 수 있다. 또한 스프링을 공부했다면 생성자 주입이라는 용어로 자주 접했을 것이다.

