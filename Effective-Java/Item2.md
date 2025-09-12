## **1. 문제 상황: 매개변수가 많은 생성자의 단점**

- **점층적 생성자 패턴(Telescoping Constructor Pattern)**:
    - 객체를 생성할 때 매개변수의 개수에 따라 여러 생성자를 제공하는 방식.
    - 예: `new NutritionFacts(240, 8, 100, 0, 35, 27);`
    - **문제점**:
        - 매개변수의 의미를 파악하기 어렵다 (코드 가독성 저하).
        - 매개변수의 순서가 바뀌어도 컴파일 오류가 발생하지 않아 런타임 오류 가능성 증가.
        - 매개변수가 많아질수록 생성자 오버로딩이 복잡해진다.
        - 클라이언트 코드가 장황해지고 유지보수가 어려워진다.
- **자바빈 패턴(JavaBeans Pattern)**:
    - Setter 메서드를 사용해 객체를 초기화하는 방식.
    - 예:
        
        ```java
        NutritionFacts facts = new NutritionFacts();
        facts.setServingSize(240);
        facts.setServings(8);
        facts.setCalories(100);
        ...
        ```
        
    - **문제점**:
        - 객체가 불완전한 상태로 생성될 수 있다 (일관성 유지 어려움).
        - Setter 메서드 호출 순서에 따라 객체 상태가 달라질 수 있다.
        - 불변 객체(Immutability)를 만들 수 없다.

</br>

## **2. 해결책: 빌더 패턴(Builder Pattern)**

빌더 패턴은 점층적 생성자 패턴과 자바빈 패턴의 단점을 보완하며, 매개변수가 많은 객체를 깔끔하고 안전하게 생성할 수 있는 방법이다.

- **빌더 패턴의 특징**:
    - 클라이언트는 빌더 객체를 통해 필요한 매개변수를 설정한 뒤, 최종적으로 `build()` 메서드를 호출해 객체를 생성.
    - 필수 매개변수는 빌더의 생성자를 통해 설정하고, 선택적 매개변수는 메서드 체이닝(Method Chaining)을 통해 설정.
    - 가독성이 높고, 불변 객체를 보장하며, 유연한 객체 생성 가능.
- **빌더 패턴 구현 예시**:
    
    ```java
    public class NutritionFacts {
        private final int servingSize;  // 필수
        private final int servings;     // 필수
        private final int calories;     // 선택
        private final int fat;          // 선택
        private final int sodium;       // 선택
        private final int carbohydrate; // 선택
    
        public static class Builder {
            // 필수 매개변수
            private final int servingSize;
            private final int servings;
    
            // 선택 매개변수 - 기본값으로 초기화
            private int calories = 0;
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
    
            public Builder sodium(int val) {
                sodium = val;
                return this;
            }
    
            public Builder carbohydrate(int val) {
                carbohydrate = val;
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
            sodium = builder.sodium;
            carbohydrate = builder.carbohydrate;
        }
    }
    
    ```
    
- **사용 예시**:
    
    ```java
    NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
        .calories(100)
        .sodium(35)
        .carbohydrate(27)
        .build();
    ```
    
    - 필수 매개변수(`servingSize`, `servings`)는 빌더 생성자에서 설정.
    - 선택 매개변수는 메서드 체이닝으로 설정.
    - 코드가 간결하고 가독성이 높으며, 매개변수의 의미가 명확해짐.
    

### **2-1) 메서드 체이닝이란?**

메서드 체이닝은 메서드 호출 후 자신(this)를 반환하도록 설계된 메서드를 연속적으로 호출하여, 한 줄의 코드로 여러 작업을 수행하는 방식이다. 빌더 패턴에서는 선택적 매개변수를 설정하는 메서드들이 this(즉, 빌더 객체 자신)를 반환하도록 구현되어, 여러 설정을 체인 형태로 연결할 수 있다.

```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
    .calories(100)
    .sodium(35)
    .carbohydrate(27)
    .build();
```

여기서 메서드 체이닝은 .calories(100), .sodium(35), .carbohydrate(27) 부분에서 나타난다.

**1. 빌더 객체 생성**

`new NutritionFacts.Builder(240, 8)`

- NutritionFacts.Builder 생성자를 호출하여 필수 매개변수(servingSize, servings)를 설정한 빌더 객체를 생성한다.
- 이 시점에서 빌더 객체는 servingSize=240, servings=8, 나머지 선택 매개변수(calories, fat, sodium, carbohydrate)는 기본값 0으로 초기화된 상태이다.

**2. 메서드 체이닝으로 선택 매개변수 설정**

`.calories(100).sodium(35).carbohydrate(27)`

- 각 메서드(calories, sodium, carbohydrate)는 빌더 객체의 필드를 설정한 뒤, 자신(this)를 반환한다. 예를 들어:
    
    `public Builder calories(int val) {    calories = val;    return this; *// 빌더 객체 자신을 반환*}`
    
    - calories(100)을 호출하면:
        - 빌더 객체의 calories 필드가 100으로 설정됨.
        - return this로 인해 같은 빌더 객체가 반환됨.
    - 따라서 .을 사용해 바로 다음 메서드(sodium(35))를 호출할 수 있음.
- 이 과정이 연속적으로 이어져, .sodium(35)는 sodium을 35로 설정하고, .carbohydrate(27)는 carbohydrate를 27로 설정한다.
- 각 메서드가 빌더 객체(this)를 반환하므로, 이러한 호출이 체인처럼 연결된다.

**3. 최종 객체 생성**

`.build();`

- 체이닝의 마지막 단계에서 build() 메서드를 호출하여 최종적으로 NutritionFacts 객체를 생성
- build()는 빌더 객체에 설정된 모든 값을 사용해 NutritionFacts 객체를 반환:
    
    `public NutritionFacts build() {    return new NutritionFacts(this);}`
    
    - 이로써 cocaCola 객체는 servingSize=240, servings=8, calories=100, sodium=35, carbohydrate=27, fat=0 (기본값)으로 초기화된다.

</br>

## **3. 빌더 패턴의 장점**

### **3-1) 가독성: 메서드 이름으로 매개변수의 의미를 명확히 알 수 있다**

- **설명**:
    - 빌더 패턴에서는 각 매개변수를 설정하는 메서드 이름이 해당 매개변수의 역할을 명확히 나타낸다.
    예를 들어, `.calories(100)`은 칼로리 값을 설정한다는 것을 직관적으로 알 수 있습니다. 
    반면, 점층적 생성자 패턴에서는 `new NutritionFacts(240, 8, 100, 0, 35, 27)`처럼 매개변수의 순서와 의미를 기억해야 하므로 가독성이 떨어진다.
    - 메서드 체이닝을 통해 코드가 한 줄로 이어지며, 마치 문장처럼 읽힌다. 이는 코드 작성자와 유지보수자가 객체 생성 과정을 쉽게 이해하도록 돕는다.
- **예시**:
    
    ```java
    NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
        .calories(100)     // "칼로리를 100으로 설정"
        .sodium(35)        // "나트륨을 35로 설정"
        .carbohydrate(27)  // "탄수화물을 27로 설정"
        .build();
    ```
    
- **추가 이점**:

### **3-2) 불변성: 빌더를 통해 생성된 객체는 불변 객체로 만들 수 있다**

- **설명**:
    - 빌더 패턴은 최종 객체(`NutritionFacts`)의 필드를 `final`로 선언하여 불변(immutable) 객체로 만들 수 있다. 불변 객체는 생성 후 상태가 변경되지 않으므로 스레드 안전성과 예측 가능한 동작을 보장
    - 자바빈 패턴에서는 `setter` 메서드로 필드를 언제든 변경할 수 있어 불변성을 보장할 수 없다. 
    반면, 빌더 패턴은 `build()` 호출 후 더 이상 객체의 상태를 변경할 방법이 없다.
- **예시**:
    
    ```java
    public class NutritionFacts {
        private final int servingSize; // final로 선언해 불변
        private final int servings;
        private final int calories;
        // ... 나머지 필드도 final
    }
    ```
    
    - `NutritionFacts` 객체는 생성 후 필드 값을 변경할 수 있는 메서드(`setter`)가 없으므로 불변이다.

### **3-3) 유연성: 선택적 매개변수를 자유롭게 설정 가능**

- **설명**:
    - 빌더 패턴은 필수 매개변수만 생성자에서 강제하고, 선택적 매개변수는 메서드 체이닝으로 원하는 것만 설정할 수 있어 유연하다.
    - 점층적 생성자 패턴에서는 모든 매개변수를 지정해야 하거나, 기본값을 하드코딩한 생성자를 별도로 만들어야 한다. 빌더 패턴은 이런 제약이 없다.
- **예시**:
    
    ```java
    // 최소한의 설정만
    NutritionFacts water = new NutritionFacts.Builder(240, 8).build();
    // 모든 선택 매개변수 설정
    NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
        .calories(100)
        .sodium(35)
        .carbohydrate(27)
        .build();
    ```
    
    - `water`는 선택 매개변수를 설정하지 않고 기본값(`0`)을 사용.
    - `cocaCola`는 필요한 선택 매개변수만 설정.
- **추가 이점**:
    - 클라이언트가 필요에 따라 원하는 매개변수만 설정 가능.
    - 새로운 선택 매개변수를 추가해도 기존 클라이언트 코드에 영향을 주지 않음(확장성).

### **3-4) 안전성: 필수 매개변수를 강제하고, 유효성 검사를 빌더 내부에서 수행할 수 있다**

- **설명**:
    - 빌더 생성자를 통해 필수 매개변수를 강제적으로 설정하도록 할 수 있다. 이는 클라이언트가 필수 값을 빠뜨리는 실수를 방지한다.
    - `build()` 메서드에서 매개변수의 유효성을 검사해 잘못된 값이 들어가면 예외를 던질 수 있다.
- **예시** (유효성 검사 추가):
    
    ```java
    public static class Builder {
        private final int servingSize;
        private final int servings;
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        private int carbohydrate = 0;
    
        public Builder(int servingSize, int servings) {
            if (servingSize <= 0 || servings <= 0) {
                throw new IllegalArgumentException("servingSize and servings must be positive");
            }
            this.servingSize = servingSize;
            this.servings = servings;
        }
    
        public NutritionFacts build() {
            if (calories < 0 || fat < 0 || sodium < 0 || carbohydrate < 0) {
                throw new IllegalArgumentException("Nutritional values cannot be negative");
            }
            return new NutritionFacts(this);
        }
        // ... 나머지 메서드
    }
    
    ```
    
    - 위 코드에서:
        - 생성자에서 `servingSize`와 `servings`가 양수인지 확인.
        - `build()` 메서드에서 선택 매개변수들이 음수인지 검사.
- **추가 이점**:
    - 객체 생성 전에 유효성 검사를 완료하므로 런타임 오류를 줄임.
    - 필수 매개변수를 강제하니 누락된 값으로 인한 버그 방지.
    - 유효성 검사 로직을 빌더에 중앙화하여 유지보수 용이.

</br>

## **4. 빌더 패턴의 단점**

### **4-1) 코드 복잡성 증가: 빌더 클래스를 별도로 작성해야 하므로 코드량이 늘어남**

- **설명**:
    - 빌더 패턴을 구현하려면 내부 정적 클래스(`Builder`)를 정의하고, 각 매개변수에 대한 필드와 메서드를 작성해야 한다. 이는 점층적 생성자나 자바빈 패턴에 비해 초기 코드 작성 비용이 높다.
    - 특히, 필드가 많을수록 빌더 클래스도 커져서 코드가 장황해질 수 있다.
- **예시**:
    - 위의 `NutritionFacts` 클래스에서 `Builder` 클래스는 6개 필드와 4개의 선택 매개변수 설정 메서드, 그리고 `build()` 메서드를 포함한다. 필드가 10개, 20개로 늘어나면 `Builder` 클래스도 비례해서 커진다.
    - 비교: 점층적 생성자는 단순히 생성자 하나 추가로 해결할 수 있지만, 빌더는 클래스 구조 자체를 설계해야 함.
- **대처 방법**:
    - **Lombok 사용**: `@Builder` 어노테이션을 사용하면 빌더 클래스 코드를 자동 생성할 수 있다.
        
        ```java
        import lombok.Builder;
        
        @Builder
        public class NutritionFacts {
            private final int servingSize;
            private final int servings;
            private final int calories;
            private final int fat;
            private final int sodium;
            private final int carbohydrate;
        }
        
        ```
        
    - **작은 클래스**: 필드가 적은 클래스에서는 빌더의 복잡성이 덜 두드러짐.
- **추가 고려**:
    - 초기 코드 작성 비용은 높지만, 장기적으로 가독성과 유지보수성에서 이점이 큼.
    - 팀 프로젝트에서는 코드 표준화를 통해 복잡성을 관리 가능.

### **4-2) 성능 오버헤드: 객체 생성 과정에서 빌더 객체를 추가로 생성하므로 약간의 오버헤드가 발생**

- **설명**:
    - 빌더 패턴은 `NutritionFacts` 객체를 생성하기 전에 중간에 `Builder` 객체를 생성. 
    이 과정에서 메모리 할당과 객체 초기화가 추가로 발생하므로 약간의 성능 오버헤드가 있다.
- **예시**:
    - `new NutritionFacts.Builder(240, 8)`로 빌더 객체를 생성하고, `.calories(100).sodium(35).carbohydrate(27)`를 호출하며 빌더 객체의 상태를 변경.
    - 최종적으로 `build()`에서 `NutritionFacts` 객체를 생성.
    - 이 과정에서 빌더 객체는 메모리에 임시로 존재하며, GC(가비지 컬렉션)에 의해 정리됨.

</br>

## **5. 언제 빌더 패턴을 사용해야 하나?**

- 생성자나 정적 팩토리 메서드의 매개변수가 많을 때(보통 4개 이상).
- 선택적 매개변수가 많거나, 객체 생성이 유연해야 할 때.
- 불변 객체를 만들고 싶을 때.
- 객체 생성 과정에서 유효성 검사가 필요할 때.
