# [Item 1] 생성자 대신 정적 팩터리 메서드를 고려하라.

## 정적 팩터리 메서드가 생성자보다 좋은 장점

```java
public static Boolean valueOf(boolean b) {
		return b ? Boolean.TRUE : Boolean.FALSE;
}
```

### 이름을 가질 수 있다.

생성자에 넘기는 매개변수와 생성자 자체만으로는 반환될 객체의 특성을 제대로 설명하지 못한다.

한 클래스에 시그니처가 같은 생성자가 여러 개 필요한 상황에는 정적 팩터리 메서드로 바꾸고 차이를 잘 드러내는 이름을 짓자.

### 호출될 때 마다 인스턴스를 새로 만들 필요가 없다.

불변 클래스는 인스턴스를 미리 만들어 놓거나 새로 생성한 인스턴스를 캐싱하여 재활용할 수 있다.

**인스턴스 통제 클래스 장점**

- 클래스를 싱글턴으로 만들 수 있다.
- 인스턴스화 불가로 만들 수 있다.
- 동치인 인스턴스가 단 하나뿐임을 보장할 수 있다. ( a == b 일 때만 a.equals(b) 가 성립한다.)

### 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.

### 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.

```java
public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
    Enum<?>[] universe = getUniverse(elementType);
    if (universe == null)
        throw new ClassCastException(elementType + " not an enum");

    if (universe.length <= 64)
        return new RegularEnumSet<>(elementType, universe);
    else
        return new JumboEnumSet<>(elementType, universe);
}
```

구현 클래스를 공개하지 않고도 다양한 객체를 반환할 수 있다. 클라이언트는 이 두 클래스의 존재를 모른다.

### 정적 팩터리 메서드를 작성하는 시점에서는 반환할 객체의 클래스가 존재하지 않아도 된다.

## 정적 팩터리 메서드의 단점

### 상속을 하려면 `public` 이나 `protected` 생성자가 필요해서, 정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다.

`Collections` 같은 유틸리티 구현 클래스들은 상속이 불가능하다. (`private` 생성자만 있으므로) 따라서 상속보다 컴포지션을 사용하도록 해야 한다.

### 정적 팩터리 메서드는 프로그래머가 찾기 어렵다.

생성자처럼 API 설명에 명확히 들어나지 않으니 API 문서(자바독)를 잘 써놓고, 메서드 이름도 널리 알려진 규약을 따라 짓자.

## 정적 팩터리 메서드에 흔히 사용하는 명명 방식

### `from` : 매개변수를 하나 받아 해당 타입 반환한다.

```java
Date date = Date.from(instant);
```

### `of` : 여러 매개변수를 받아 적합한 타입 반환한다.

```java
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
```

### `valueOf` : `from`, `of`의 자세한 버전이다.

```java
String str = String.valueOf("hello")
```

### `instance` or `getInstance` : 인스턴스를 반환하지만, 같은 인스턴스임을 보장하진 않는다.

```java
StackWalker luke = StackWalker.getInstance(options);
```

### `create` or `newInstance` : 매번 새로운 인스턴스를 생성해 반환한다.

```java
Object newArray = Array.newInstance(classObject, arrayLen);
```

### `getType` : 생성할 클래스가 아닌 다른 클래스에 팩터리 메서드를 정의할 때 쓴다.

```java
FileStore fs = File.getFileStore(path);
```

### `newType` : 생성할 클래스가 아닌 다른 클래스에 팩터리 메서드를 정의할 때 쓴다.

```java
BufferReader br = Files.newBufferedReader(path);
```

### `type` : `getType`과 `newType`의 간결한 버전이다.

```java
List<Complaint> litany = Collections.list(legacyLitany);
```

<aside>
💡 무작정 `public` 생성자를 제공하는 습관을 고치고, 정적 팩터리를 사용하는 것이 유리하다면 적극 사용하자.

</aside>

# [Item 2] 생성자에 매개변수가 많다면 빌더를 고려하라.

## 점층적 생성자 패턴

- 코드
    
    ```java
    public class NutritionFacts {
    		private final int servingSize;
    		private final int servings;
    		...
    		
    		public NutritionFacts(int servingSize, int servings) {
    					this(servingSize, servings, 0);
    		}
    		public NutritionFacts(int servingSize, int servings, int calories) {
    					this(servingSize, servings, calories, 0);
    		}
    		...
    ```
    

점층적 생성자 패턴은, 매개 변수의 개수가 많아지면 확장이 어렵다. 코드 또한 작성하거나 읽기 어렵다.

## 자바빈즈 패턴

- 코드
    
    ```java
    public class NutritionFacts {
    		private int servingSize = -1;
    		private int servings = -1;
    		private int calories = 0;
    		...
    		public NutritionFacts() {}
    
    		public void setServingSize(int val) { servingSize = val; } // 여러 setter 메서드 들
    		public void setServings(int val) { servings = val; }
    		public void setCalories(int val) { calories = val; }
    ```
    

인스턴스를 만들기 쉽고, 더 읽기 쉬운 코드가 되었지만 객체 하나를 만드는 데 메서드를 여러개 호출해야 하고, 객체가 완전히 생성되기 전까지는 일관성이 없다.

일관성이 무너지는 문제 때문에 자바빈즈 패턴에서는 클래스를 불변으로 만들 수 없다.

## 빌더 패턴 - 점층적 생성자 패턴과 자바빈즈 패턴의 장점만을 취한

- 코드
    
    ```java
    public class NutritionFacts {
    
      private final int servingSize;
      private final int servings;
      private final int calories;
      private final int fat;
      private final int sodium;
      private final int carbohydrate;
    
      public static class Builder {
    
        private final int servingSize; // 필수 매개변수
        private final int servings;
    
        private int calories = 0; // 선택 매개변수
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;
    
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
        ...
    
        public NutritionFacts build() {
          return new NutritionFacts(this);
        }
      }
    
      private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
      }
    }
    ```
    
    ```java
    NatritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
                      .calories(100)
                      .sodium(35)
                      .carbohydrate(27)
                      .build();
    ```
    

빌더의 세터 메서드들은 빌더 자신을 반환하므로, 연쇄적으로 호출된다. 이런 방식을 **플루언트 API** 또는 **메서드 연쇄**라고 한다.

빌더 패턴은 상당히 유연한데, 빌더 하나로 여러 객체를 순회하면서 만들 수 있고, 빌더에 넘기는 매개변수에 따라 다른 객체를 만들 수도 있다.

**장점**

- 쓰기 쉽고, 읽기 쉽다.
- 기본 값으로 설정하고 싶다면 메서드를 호출하지 않으면 된다.
- 계층적으로 설계된 클래스와 함께 쓰기 좋다.

**단점**

- 빌더 생성 시 생기는 장황한 코드. 매개변수가 4개 이상은 되어야 값어치를 한다.

<aside>
💡 생성자나 정적 팩터리가 처리해야 할 매개변수가 많다면 빌더 패턴을 선택하자.

</aside>

# [Item 3] `private` 생성자나 열거 타입으로 싱글턴임을 보증하라.

## 싱글턴

인스턴스를 오직 하나만 생성할 수 있는 클래스이다. 하지만 클래스를 싱글턴으로 만들면 테스트가 어려워질 수 있다.

### `public static final` 필드 방식의 싱글턴

```java
public class Elvis {
		public static final Elvis INSTANCE = new Elvis();  // public static final 필드 방식

		private Elvis() { ... }
		...
```

`private` 생성자는 `Elvis.INSTANCE`를 초기화 할 때 딱 한번 호출된다. 

간결하고, `public static` 필드가 `final`인 것으로 보아 싱글턴임을 한눈에 알 수 있다.

### 정적 팩터리 방식의 싱글턴

```java
public class Elvis {
		private static final Elvis INSTANCE = new Elvis();
		private Elvis() { ... }

		public static Elvis getInstance() { return INSTANCE; } // 정적 팩터리 방식
		...
```

**장점** - 이러한 장점들이 굳이 필요 없다면 `public static final` 방식을 쓰자.

- 다른 인스턴스도 반환하도록 코드를 수정할 수 있다. 확장이 쉽다.
- 정적 팩터리를 제네릭 싱글턴 팩터리로 만들 수 있다.
- 정적 팩터리의 메서드 참조를 `Supplier`로 사용할 수 있다. → `Elvis::getInstance` 대신 `Supplier<Elvis>`

> 두 방식 모두 예외가 있는데, 권한이 있는 클라이언트는 리플렉션 API인 `AccessibleObject.setAccessible`을 사용해 `private` 생성자를 호출할 수 있다고 한다. → 생성자에서 처리해주자.
> 

### **`readResolse` 메서드**

모든 인스턴스 필드를 `final`로 선언하고, `readResolse`메서드를 제공해야 직렬화된 인스턴스를 역직렬화할 때마다 새로운 인스턴스가 만들어지는 걸 방지할 수 있다.

- 코드
    
    ```java
    private Object readResolve() {
    		return INSTANCE;
    }
    ```
    

### `Enum` 타입 선언

- 코드
    
    ```java
    public enum Elvis {
    		INSTANCE;
    		
    		public void leaveTheBuilding() { ... }
    }
    ```
    

간결하고 추가 노력 없이 직렬화 할 수 있다.

단, 만들려는 싱글턴이 `Enum` 외의 클래스를 상속해야 한다면 이 방법은 사용할 수 없다. 열거 타입이 다른 인터페이스를 구현하도록 선언할 수 없기 때문이다.

# [Item 4] 인스턴스화를 막으려거든 `private` 생성자를 사용하라.

정적 메서드와 정적 필드만을 담은 클래스를 만들고 싶을 때는 어떻게 해야 할까?

이런 클래스는 인스턴스로 만들어 쓰려고 설계하지 않았기 때문에, 인스턴스화를 막아야 한다. 생성자를 명시하여 기본 생성자가 자동으로 생기지 않게 하자.

## 추상 클래스로 만들기 - 인스턴스화를 막을 수 없다

하위 클래스를 만들어 인스턴스화 할 수 있으므로, 해법이 아니다. 게다가 사용자가 상속해서 클래스를 사용해야 하는 것인지 오히려 헷갈리게 만든다.

## `private` 생성자 추가하기

- `java.util.Collections`
    
    ```java
    public class Collections {
        // Suppresses default constructor, ensuring non-instantiability. 사용하지 않는 생성자 이므로, 주석을 달아놓자.
        private Collections() {
    				// 에러를 던져서 클래스 안에서도 실수로 호출하지 않게 하면 좋다.
        }
    ```
    

이 방식을 사용하면, 상속이 불가능하다. 모든 생성자는 상위 클래스의 생성자를 호출하기 때문이다.

# [Item 5] 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

### 정적 유틸리티를 잘못 사용한 예

```java
public class SpellChecker {
		private static final Lexicon dictionary = new English();

		private SpellChecker() {}
```

### 싱글턴을 잘못 사용한 예

Lexion → English, Korean

```java
public class SpellChecker {
		private final Lexicon dictionary = new English();

		private SpellChecker(...) {}
		public static SpellChecker INSTANCE = new SpellChecker(...);
```

→ 두 방식 모두 유연하지 않고, 테스트하기 어렵다. (자원을 직접 명시하고 있다.)

→ 간단히 `final` 한정자를 제거하여, 다른 사전으로 교체하는 메서드를 추가하는 방법은 해법이 아니다. 이는 오류를 내기 쉽고, 멀티스레드 환경에서 쓸 수 없다.

→ 사용하는 자원에 따라 동작이 달라지는 클래스는 정적 유틸리티 클래스나 싱글턴 방식이 적합하지 않다.

## 인스턴스를 생성할 때 생성자에 필요한 자원을 넘겨주는 방식 - 의존 객체 주입 패턴

```java
public class SpellChecker {
		private final Lexion dictionary;

		public SpellChecker(Lexion dictionary) {
				this.dictionary = Objects.requireNonNull(dictionary);
		}
```

이제 dictionary 자원이 몇 개이든, `Lexion`을 상속한 클래스라면 상관없이 잘 동작한다. 불변 또한 보장한다. 테스트도 쉬워진다.

## 생성자에 자원 팩터리를 넘겨주는 방식

### 팩터리란?

호출할 때 마다 특정 타입의 인스턴스를 반복해서 만들어주는 객체. `Supplier<T>` 인터페이스가 팩터리를 표현한 좋은 예다.

<aside>
💡 클래스가 내부적으로 하나 이상의 자원에 의존하고, 그 자원이 클래스 동작에 영향을 준다면 싱글턴과 정적 유틸리티 클래스 사용은 지양하자. 대신 필요한 자원(혹은 그 자원을 만들어주는 팩터리)를 생성자(혹은 정적 팩터리 혹은 빌더)에 넘겨주자.

</aside>
