## 1. 생성자 대신 정적 팩터리 메서드를 고려하라

자바에서 객체를 생성하는 가장 기본적인 방법은 생성자(Constructor) 호출이다.

하지만 《이펙티브 자바》에서는 “생성자 대신 정적 팩터리 메서드를 고려하라”고 조언하는데, 왜 일까?

### 2. 생성자(Constructor)

생성자는 **클래스 이름과 동일한 특별한 메서드이**다.

```java
Book book = new Book("Effective Java", 30000);
```

- 반환 타입이 없음
- `new` 키워드와 함께 호출
- 오버로딩 가능하지만, 이름을 따로 붙일 수 없음

즉, 생성자는 단순하고 직관적이지만 **표현력이 부족**할 때가 있다.

### 3. 정적 팩터리 메서드란?

간단히 말해, **`new`로 직접 객체를 생성하지 않고, 클래스에 정의된 `static` 메서드를 통해 객체를 반환하는 방식**

```java
// 정적 팩터리 메서드 예시
Ball ball = Ball.of(position, number);

// 내부 구현
public class Ball {
    private Ball(int position, int number) { ... }
    public static Ball of(int position, int number) {
        return new Ball(position, number);
    }
}
```

즉, `Ball.of(...)`는 내부에서 `new Ball(...)`을 호출한 뒤 객체를 반환한다.

여기서 핵심은 "객체 생성을 메서드에 위임한다"는 것.

## 4. 왜 정적 팩터리 메서드인가? (5가지 장점)

---

### 4-1) 이름을 가질 수 있다

생성자는 클래스명과 같아야 해서 의도를 드러내기 어렵지만, 정적 메서드는 이름으로 의미를 명확히 표현할 수 있다

```java
BigInteger prime = BigInteger.probablePrime(32, new Random());

// vs

BigInteger prime = new BigInteger(32, new Random());
```

정적 메서드의 경우 "32비트 크기의 소수를 생성"한다는 의도가 명확하다.

### 4-2) 호출될 때마다 새로운 객체를 만들 필요가 없다

**`Boolean`은 불변(immutable)** + 값 종류가 단 2개(true/false)라서, 매번 새 인스턴스를 만들 필요가 없다.

JDK는 이를 활용해 **미리 만들어둔 두 객체**만 계속 돌려준다. 

- `Boolean.TRUE` (true 값의 단일 객체)
- `Boolean.FALSE` (false 값의 단일 객체)

그리고 아래 정적 팩터리들이 이 둘을 그대로 반환하는 방식

- `Boolean.valueOf(boolean)`
- `Boolean.valueOf(String)` (문자열이 `"true"`(대소문자 무시)에 해당하면 TRUE, 아니면 FALSE)

**예시 1) `valueOf`는 항상 같은 객체를 준다**

```java
Boolean a = Boolean.valueOf(true);
Boolean b = Boolean.valueOf(true);
Boolean c = Boolean.valueOf(false);

System.out.println(a == b);      // true  (같은 TRUE 인스턴스)
System.out.println(a == Boolean.TRUE);  // true
System.out.println(c == Boolean.FALSE); // true
```

- `==` 비교가 `true`인 이유: **동일 객체(싱글턴) 재사용**
- 매 호출마다 `new`로 만들지 않으니 **GC 부담 감소 + 캐시 적중으로 성능 이점**

---

**예시 2) `new Boolean(...)`은 피하자**

```java
Boolean x = new Boolean(true);
Boolean y = Boolean.valueOf(true);

System.out.println(x == y);     // false (서로 다른 객체)
System.out.println(x.equals(y)); // true  (논리값은 같음)
```

- `new`를 쓰면 **항상 새로운 객체**가 생겨 `==`가 실패한다.
- 불변·값객체에는 굳이 새로 만들 이유가 없다.

### 4-3)  반환 타입의 하위 타입 객체를 반환할 수 있다

**<생성자와의 차이>**

생성자는 호출하는 클래스의 인스턴스를 직접 생성하고 반환합니다. 즉, `new MyClass()`를 호출하면 정확히 `MyClass` 타입의 객체가 반환되며, 그 하위 타입이나 다른 구현체를 반환할 수 없다. 

이로 인해 생성자는 반환 타입의 유연성이 제한된다.

```java
public class MyClass {
    public MyClass() {
        // MyClass 객체를 생성
    }
}
```

여기서 new MyClass()는 반드시 MyClass 타입의 객체만 반환한다.

**<정적 팩토리 메서드의 유연성>**

반면, **정적 팩토리 메서드**는 반환 타입을 인터페이스나 상위 클래스 타입으로 선언하고, 실제로는 그 하위 타입이나 구현체를 반환할 수 있다. 이는 클라이언트 코드가 내부 구현 세부사항을 몰라도 되도록 캡슐화를 강화하고, 설계 유연성을 높이는 데 유용하다.

```java
List<String> list = Collections.unmodifiableList(new ArrayList<>());
```

- **리턴 타입**
    - List<String>
- **실제 반환 객체**
    - 내부적으로 Collections.UnmodifiableList라는 클래스의 인스턴스
- **의미**
    - 클라이언트는 List 인터페이스를 통해 반환된 객체를 사용하지만, 내부적으로는 수정 불가능한 리스트 
    구현체(UnmodifiableList)를 받는다. 이는 캡슐화를 통해 구현 세부사항을 숨기고, 클라이언트가 리스트를 수정하려는 시도를 방지한다.

### 4-4) 입력 매개변수에 따라 다른 클래스 객체를 반환할 수 있다

정적 팩토리 메서드는 입력 매개변수의 값에 따라 서로 다른 클래스(하위 타입)의 인스턴스를 반환할 수 있다. 
이는 다형성(Polymorphism)과 조건문을 활용해 클라이언트가 필요로 하는 적절한 객체를 동적으로 선택해 제공하는 강력한 설계 방식이다. 이를 통해 클라이언트는 내부 구현 세부사항을 알 필요 없이, 인터페이스나 상위 클래스 타입으로 객체를 사용할 수 있다.

**<핵심 개념>**

- **생성자와의 차이**
    - 생성자는 고정된 클래스의 객체를 생성하지만, 정적 팩토리 메서드는 조건에 따라 서로 다른 하위 타입의 객체를 반환할 수 있다.
- **유연성**
    - 입력 매개변수를 기반으로 적절한 구현체를 선택해 반환하므로, 클라이언트 코드가 단순해지고 구현체 변경이 쉬워진다.
- **캡슐화**
    - 내부 구현(어떤 클래스의 객체가 반환되는지)을 숨기고, 클라이언트는 반환된 객체의 인터페이스만 사용하면 된다.

**<실제 사용 사례>**

Java 표준 라이브러리에서도 이 패턴이 자주 사용된다. 예를 들어, Collections 클래스의 unmodifiableList 메서드는 입력 리스트의 타입에 따라 다른 구현체를 반환할 수 있다.

```java
List<String> list = new ArrayList<>();
List<String> unmodifiable = Collections.unmodifiableList(list);
```

- **설명**
    - 입력 리스트의 특성에 따라 Collections.UnmodifiableList 또는 다른 내부 구현체가 반환된다. 
    클라이언트는 List 인터페이스를 통해 일관되게 작업할 수 있다.

**<이점>**

1. **유연한 객체 생성**
    - 입력 매개변수에 따라 적합한 하위 타입을 선택해 반환할 수 있어, 다양한 요구사항을 충족할 수 있다.
2. **캡슐화 강화**
    - 클라이언트는 구현 클래스의 존재를 알 필요 없이 인터페이스나 상위 클래스 타입만 사용하면 된다.
    - 내부 구현이 변경되더라도 클라이언트 코드는 영향을 받지 않는다.
3. **성능 최적화**
    - 입력에 따라 더 효율적인 구현체를 선택할 수 있습니다(예: EnumSet의 RegularEnumSet vs JumboEnumSet)

### **4-5) 반환 객체의 클래스가 작성 시점에 없어도 된다**

정적 팩토리 메서드는 컴파일 시점에 반환할 클래스가 정의되어 있지 않더라도, 런타임에 동적으로 클래스를 로드해 객체를 생성하고 반환할 수 있다. 이는 플러그인 아키텍처나 확장 가능한 시스템을 설계할 때 매우 유용한데, 
대표적인 예로 Java의 서비스 로더(ServiceLoader) 메커니즘과 JDBC가 있다.

**<핵심 개념>**

- **동적 로딩**
    - 컴파일 시점에 구현체 클래스가 존재하지 않아도, 런타임에 클래스패스에 있는 구현체를 동적으로 로드해 사용할 수 있다.
- **서비스 제공자 프레임워크**
    - 인터페이스(서비스)와 이를 구현하는 클래스(제공자)를 분리하고, 런타임에 적절한 제공자를 로드해 객체를 생성한다.
    
    <aside>
    💡
    
    서비스 제공자 프레임워크는 인터페이스(서비스)와 이를 구현하는 구현체(제공자)를 분리하고, 런타임에 적절한 구현체를 동적으로 로드해 사용하는 설계 패턴이다. 
    
    이 패턴은 시스템이 새로운 구현체를 쉽게 추가할 수 있도록 확장 가능하게 만들며, 클라이언트 코드가 특정 구현체에 의존하지 않도록 캡슐화를 강화한다. Java에서는 ServiceLoader 클래스가 이 프레임워크를 구현하는 핵심 도구이다.
    
    **구성 요소**
    
    서비스 제공자 프레임워크는 다음과 같은 세 가지 주요 구성 요소로 이루어져 있다
    
    1. **서비스 인터페이스(Service Interface)**
        - 서비스가 제공하는 기능의 계약을 정의하는 인터페이스이다.
        - 클라이언트는 이 인터페이스를 통해 구현체와 상호작용한다.
        - 예: java.sql.Driver (JDBC 드라이버 인터페이스), javax.servlet.Servlet (서블릿 인터페이스).
    2. **제공자 등록 API(Provider Registration API)**
        - 구현체(제공자)를 시스템에 등록하는 메커니즘이다.
        - Java에서는 ServiceLoader가 클래스패스의 META-INF/services 디렉토리에 있는 설정 파일을 읽어 구현체를 등록한다.
        - 예: META-INF/services/java.sql.Driver 파일에 MySQL 드라이버 클래스(com.mysql.cj.jdbc.Driver)를 명시.
    3. **서비스 접근 API(Service Access API)**
        - 클라이언트가 서비스 인터페이스의 인스턴스를 얻는 방법을 제공한다.
        - 일반적으로 정적 팩토리 메서드나 ServiceLoader를 통해 구현체를 동적으로 로드한다.
        - 예: ServiceLoader.load(Driver.class), DriverManager.getConnection.
    
    **동작 방식**
    
    1. **구현체 제공**
        - 서비스 인터페이스를 구현한 클래스(제공자)를 작성한다.
        - 구현체는 JAR 파일에 포함되며, META-INF/services 디렉토리에 인터페이스 이름과 구현체 클래스의 정규화된 이름을 매핑한 설정 파일을 추가한다.
        - 예: META-INF/services/com.example.MessageService 파일에 com.example.EmailService를 기록.
    2. **런타임 로딩**
        - ServiceLoader가 클래스패스를 스캔해 META-INF/services 디렉토리의 설정 파일을 읽고, 명시된 구현체를 동적으로 로드한다.
        - 클라이언트는 ServiceLoader.load를 호출해 사용 가능한 구현체를 얻는다.
    3. **클라이언트 사용**
        - 클라이언트는 서비스 인터페이스를 통해 로드된 구현체를 사용한다.
        - 구현체의 구체적인 클래스를 알 필요 없이 인터페이스의 메서드만 호출하면 된다.
    </aside>
    
- **장점**
    - 시스템을 확장 가능하게 만들고, 클라이언트 코드가 특정 구현체에 의존하지 않도록 만든다.

**예제 1: ServiceLoader를 사용한 동적 로딩**

```java
import java.sql.Driver;
import java.util.ServiceLoader;

public class Main {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:3306/mydb";
        java.util.Properties props = new java.util.Properties();
        props.put("user", "username");
        props.put("password", "password");

        // ServiceLoader를 통해 Driver 구현체를 동적으로 로드
        ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);
        for (Driver driver : loader) {
            try {
                Connection conn = driver.connect(url, props);
                System.out.println("Connected using driver: " + driver.getClass().getName());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

**설명**

- `ServiceLoader.load(Driver.class)`는 클래스패스에서 java.sql.Driver 인터페이스를 구현한 모든 클래스를 찾아 로드한다.
- 런타임에 MySQL, PostgreSQL, Oracle 등의 JDBC 드라이버가 클래스패스에 있다면, 해당 구현체가 동적으로 로드
- 클라이언트 코드는 Driver 인터페이스만 알면 되고, 구체적인 구현체(MySQL 드라이버, PostgreSQL 드라이버 등)를 알 필요가 없다!
- 이 과정에서 반환되는 Driver 객체의 클래스는 컴파일 시점에 존재하지 않을 수도 있다.

**<이점>**

1. **확장성**
    - 새로운 구현체(예: 새로운 DB 드라이버)를 추가하려면 클래스패스에 JAR 파일만 추가하면 된다. 
    기존 코드를 수정할 필요 X
    - 예: 새로운 NoSQL 데이터베이스 드라이버를 추가해도 클라이언트 코드는 변경되지 않음.
2. **캡슐화**
    - 클라이언트는 구현체 클래스를 알 필요 없이 인터페이스만 사용한다.
    - 구현체의 세부사항(예: MySQL 드라이버의 내부 동작)은 완전히 숨겨진다.
3. **유연성**
    - 런타임에 적절한 구현체를 동적으로 선택할 수 있어, 시스템의 유연성이 크게 향상된다.
4. **플러그인 아키텍처 지원**
    - 새로운 기능을 플러그인 형태로 추가할 수 있어, 시스템 확장이 쉬워진다.

**<주의사항>**

1. **의존성 관리**
    - 클래스패스에 필요한 구현체(JAR 파일)가 없으면 ServiceLoader가 해당 구현체를 로드하지 못한다.
2. **성능 고려**
    - ServiceLoader는 런타임에 클래스패스를 스캔하므로, 초기 로딩 시간이 길어질 수 있다. 캐싱 등을 고려해 최적화할 수 있다.
