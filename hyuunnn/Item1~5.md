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

또한 반환 타입은 `EnumSet`을 상속하기만 하면 되기 때문에 유연한 개발이 가능하다. (위 코드는 원소의 개수에 따라서 다른 클래스를 반환하고 있다.)

### 정적 팩터리 메서드를 사용하기 위한 별도의 학습이 필요하다.

생성자는 객체를 생성했을 때 자동으로 동작하지만 정적 팩터리 메서드는 어떤 메서드들이 구현되어 있는지 찾아야 한다.

그렇기 때문에 API 문서화를 꼼꼼히 하고 메서드명을 정의할 때 규약을 따르는 것을 권장하고 있다.

* `from`: 
* `of`: 
* `valueOf`:
* `instance` 또는 `getInstance`:
* `create` 또는 `newInstance`:
* `getType`:
* `newType`:
* `type`: 

## Item 2 - 생성자에 매개변수가 많다면 빌더를 고려하라

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

위와 같이 자기 자신을 호출하면서 늘려가는 방식을 점층적 생성자 패턴(telescoping constructor pattern)이라고 한다.

## Item 3 - private 생성자나 열거 타입으로 싱글턴임을 보증하라



## Item 4 - 인스턴스화를 막으려거든 private 생성자를 사용하라



## Item 5 - 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라


