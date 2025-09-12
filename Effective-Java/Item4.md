## **1. 배경**

불필요한 인스턴스화는 리소스 낭비를 초래하고, 클래스의 의도(정적 멤버만 사용)를 벗어나는 사용을 유발한다.

</br>


## **2. 배경: 인스턴스화가 불필요한 클래스**

- **대상 클래스**:
    - **유틸리티 클래스**: 정적 메서드와 정적 필드만 포함하며, 인스턴스를 생성할 필요가 없는 클래스.
        - 예: `java.lang.Math`, `java.util.Arrays`, `java.util.Collections`.
    - **정적 멤버만 포함하는 클래스**: 객체 지향적이지 않은, 정적 메서드와 필드로만 구성된 클래스.
- **문제점**:
    - 자바에서는 생성자를 명시적으로 정의하지 않으면 컴파일러가 **기본 생성자**(public 또는 package-private)를 자동으로 제공.
    - 이로 인해 클라이언트가 의도치 않게 `new`를 사용해 인스턴스를 생성할 수 있음.
    - 예: `new Math()`를 호출해 `Math` 클래스의 인스턴스를 만들면, 의미 없는 객체가 생성되어 리소스 낭비.

</br>


## **3. 해결책: private 생성자 사용**

- **방법**:
    - 클래스의 생성자를 `private`으로 선언해 외부에서 접근하지 못하도록 제한.
    - 생성자를 명시적으로 정의함으로써 컴파일러가 기본 생성자를 자동 생성하지 않도록 함.
- **예시 코드**
    
    ```java
    public class UtilityClass {
        // 인스턴스화 방지용 private 생성자
        private UtilityClass() {
            throw new AssertionError("인스턴스화 금지");
        }
    
        // 정적 메서드 예시
        public static int add(int a, int b) {
            return a + b;
        }
    }
    ```
    
- **설명**:
    - `private UtilityClass()`는 외부에서 생성자를 호출하지 못하도록 막음.
    - 클라이언트가 `new UtilityClass()`를 시도하면 컴파일 오류 발생 (`UtilityClass() has private access`).
    - `throw new AssertionError()`는 리플렉션(Reflection)을 통해 생성자를 호출하려는 시도를 방어.

</br>


## **4. private 생성자에 AssertionError 추가의 이유**

- **목적**:
    - 클래스 내부에서 실수로 생성자를 호출하는 경우를 방지.
    - 리플렉션 API를 사용해 `private` 생성자를 강제로 호출하는 경우를 방어.
- **동작**:
    - 내부 코드에서 `new UtilityClass()`를 호출하면 `AssertionError`가 발생.
    - 리플렉션으로 생성자를 호출해도 예외가 던져져 인스턴스화 실패.
- **예시**:
    - 리플렉션 공격 시도:
        
        ```java
        import java.lang.reflect.Constructor;
        
        public class Test {
            public static void main(String[] args) throws Exception {
                Constructor<UtilityClass> constructor = UtilityClass.class.getDeclaredConstructor();
                constructor.setAccessible(true); // private 생성자 접근 허용
                UtilityClass instance = constructor.newInstance(); // AssertionError 발생
            }
        }
        
        ```
        
    - `AssertionError`로 인해 인스턴스화가 실패하며, 클래스의 의도(인스턴스화 방지)가 유지됨.

### 4-1)리플렉션 API란?

- **정의**: 리플렉션은 런타임에 클래스의 메타데이터(구조적 정보)를 조회하고, 이를 기반으로 동적으로 객체를 생성하거나 메서드를 호출, 필드 값을 변경하는 기능
- **패키지**: 자바의 java.lang.reflect 패키지와 java.lang.Class 클래스를 통해 제공.
- **핵심 기능**:
    - 클래스의 정보 조회 (필드, 메서드, 생성자, 상위 클래스 등).
    - 접근 제어자 무시 (예: private 필드나 메서드에 접근).
    - 동적 객체 생성 및 메서드 호출.
- **예시**: 컴파일 시점에는 알 수 없는 클래스의 정보를 런타임에 분석하거나, 프레임워크(예: Spring, Hibernate)에서 동적으로 객체를 생성할 때 사용.

</br>


## **5. private 생성자를 사용하지 않을 경우의 문제**

- **기본 생성자 제공**:
    - 생성자를 정의하지 않으면 컴파일러가 `public` 또는 `package-private` 기본 생성자를 자동 생성.
    - 예:
        
        ```java
        public class UtilityClass {
            public static int add(int a, int b) {
                return a + b;
            }
        }
        ```
        
        - 클라이언트가 `new UtilityClass()`를 호출해 인스턴스 생성 가능.
        - 이는 클래스의 의도(정적 멤버만 사용)를 위반하고, 불필요한 객체 생성으로 메모리 낭비.
- **의도 불명확**:
    - 클라이언트가 유틸리티 클래스를 객체 지향적으로 사용하려 시도할 수 있음.
    - 예: `UtilityClass uc = new UtilityClass(); uc.add(1, 2);` (잘못된 사용).

</br>


## **6. 주의사항**

- **상속 방지**:
    - `private` 생성자는 상속을 불가능하게 함 (하위 클래스가 `super()` 호출 불가).
    - 상속을 방지하려는 의도가 있다면, 추가로 클래스를 `final`로 선언 가능:
        
        ```java
        public final class UtilityClass {
            private UtilityClass() {
                throw new AssertionError();
            }
            // ...
        }
        ```
        
- **정적 멤버만 포함**:
    - 클래스가 인스턴스 멤버(비정적 필드/메서드)를 포함하면, 인스턴스화 방지의 의미가 약화될 수 있음.
    - 따라서 유틸리티 클래스는 정적 멤버만 포함하도록 설계.

</br>


## **7. 실제 사용 사례**

- **자바 표준 라이브러리**:
    - `java.lang.Math`: `sin()`, `cos()`, `PI` 같은 정적 메서드와 상수 제공.
    - `java.util.Arrays`: `sort()`, `asList()` 같은 정적 메서드 제공.
    - `java.util.Collections`: `emptyList()`, `unmodifiableList()` 같은 정적 메서드 제공.
- **사용 예시**:
    
    ```java
    int sum = UtilityClass.add(1, 2); // 정적 메서드 호출
    // UtilityClass uc = new UtilityClass(); // 컴파일 오류
    
    ```
    
- **의존성 주입**:
    - 유틸리티 클래스 대신, 정적 메서드의 기능을 빈(Bean)이나 서비스로 제공.
    - 예: Spring 프레임워크에서 `@Service`로 유틸리티 로직 캡슐화.
