객체 생성은 메모리와 CPU 자원을 소모하며, 특히 반복문이나 빈번히 호출되는 코드에서 불필요한 객체 생성은 성능 저하를 초래할 수 있다. 
이 아이템은 **객체 재사용**, **캐싱**, **효율적인 자료구조 사용**을 통해 성능을 향상시키는 방법을 설명한다.

</br>

## 1. 핵심 원칙: 객체 재사용의 중요성

- **핵심 메시지**: 동일한 기능을 하는 객체를 매번 새로 생성하기보다는 기존 객체를 재사용하는 것이 성능과 코드 품질 면에서 유리하다.
- **왜 중요한가?**
    - 객체 생성은 메모리 할당과 초기화 비용을 발생시킨다.
    - 불변 객체는 상태가 변하지 않으므로 안전하게 재사용 가능하며, 특히 JVM의 최적화(예: 문자열 풀, Integer 캐싱)와 잘 맞는다.
    - 재사용은 성능뿐만 아니라 코드의 명확성과 세련미를 높인다.

</br>

## 2. 불변 객체와 문자열 풀

### 2-1) 문제 사례: 불필요한 `String` 객체 생성

제공된 텍스트에서 강조된 잘못된 예시는 다음과 같다:

```java
String s = new String("bikini"); // 따라 하지 말 것!
```

- **문제점**:
    - `new String("bikini")`는 매 호출마다 새로운 `String` 객체를 힙에 생성.
    - `"bikini"` 자체는 JVM의 문자열 풀(string pool)에 저장된 불변 객체로, 이미 존재하는 객체를 재사용할 수 있다.
    - 반복문이나 빈번히 호출되는 메서드에서 이 코드를 사용하면 수백만 개의 불필요한 객체가 생성될 수 있다.
- **개선된 코드**:

```java
String s = "bikini";
```

- **개선 효과**:
    - 문자열 리터럴(`"bikini"`)은 문자열 풀에 저장되어 동일한 가상 머신 내 모든 코드가 동일한 객체를 공유한다.
    - 메모리 사용량과 GC 부담을 줄이며, 동일성 비교(`==`)에서도 일관된 결과를 보장한다.
- **추가 팁**:
    - 문자열 풀은 메모리 효율성을 높이지만, 고유 문자열이 많아지면 풀의 크기가 커질 수 있으므로 주의.

</br>

## 3. 정적 팩토리 메서드 활용

불변 클래스에서 생성자 대신 **정적 팩토리 메서드**(아이템 1)를 사용하면 객체 생성을 제어하고 캐싱된 객체를 반환할 수 있다.

**예시: `Boolean.valueOf`**

- **잘못된 코드**:

```java
Boolean b = new Boolean("true"); // 새로운 객체 생성
```

- 매 호출마다 새로운 `Boolean` 객체를 생성하며, 자바 9에서 이 생성자는 사용 자제(deprecated)로 지정됨.

- **개선된 코드**:

```java
Boolean b = Boolean.valueOf("true"); // 캐싱된 객체 반환
```

- `Boolean.valueOf`는 `Boolean.TRUE` 또는 `Boolean.FALSE`를 반환하여 객체 생성을 방지.
- **장점**:
    - 객체 생성 비용 감소.
    - 캐싱된 객체는 동일성 비교(`==`)에서도 유리.
    - 가변 객체라도 변경되지 않는 경우 재사용 가능(예: `Collections.unmodifiableList`).
- **현대 자바에서의 고려사항**:
    - 자바 9 이후 `Boolean` 생성자는 비추천되며, `valueOf` 사용이 표준.
    - 자바 17 이상에서는 `record` 클래스를 활용해 불변 객체를 간결히 설계할 수 있음.

<aside>


### 3-1) record 클래스란?

record는 데이터 중심의 불변 객체를 간결하게 정의하기 위한 자바의 구문이다. 주로 "데이터 홀더" 역할을 하는 클래스를 작성할 때 보일러플레이트 코드를 줄여준다. record는 불변(immutable)이며, 자동으로 다음 요소를 제공힌다:

- **생성자**: 모든 필드를 초기화하는 생성자.
- **getter 메서드**: 각 필드에 대한 접근자 메서드(예: name()).
- **equals(), hashCode(), toString()**: 적절히 구현된 메서드.
- **private final 필드**: 선언된 필드를 자동으로 final로 처리해 불변성 보장.

### 3-2) **record 선언 예시**

```java
public record Person(String name, int age) {}
```

- 이 코드는 다음을 자동으로 제공:
    - name과 age를 초기화하는 생성자.
    - name()과 age() getter 메서드.
    - equals(), hashCode(), toString() 구현.
- 추가로 커스텀 메서드나 정적 팩토리 메서드를 정의할 수 있음.

### 3-3) **record의 특징**

- **불변성**: 필드는 final로 선언되어 변경 불가. 이는 아이템 6에서 강조하는 불변 객체 재사용에 적합.
- **간결성**: 클래스 정의가 간단해 코드 가독성과 유지보수성 향상.
- **자동 구현**: equals(), hashCode(), toString()이 표준적으로 구현되어 일관된 동작 보장.
- **제한사항**:
    - record는 상속 불가(암묵적으로 java.lang.Record를 상속).
    - 필드는 final이므로 가변 상태를 가질 수 없음(가변 객체가 필요한 경우 적합하지 않음).

### 3-4) **아이템 6과의 연관성**

- **불변 객체 재사용**: record는 불변이므로 안전하게 캐싱하고 재사용 가능. 예를 들어, 동일한 데이터의 record 인스턴스를 문자열 풀처럼 공유 가능.
- **정적 팩토리 메서드 활용**: record에 정적 팩토리 메서드를 추가해 캐싱 전략을 구현할 수 있음.
    
    ```java
    public record Person(String name, int age) {
        private static final Map<String, Person> CACHE = new ConcurrentHashMap<>();
        
        public static Person of(String name, int age) {
            return CACHE.computeIfAbsent(name + age, k -> new Person(name, age));
        }
    }
    ```
    
    - 위 코드에서 Person.of는 캐싱된 객체를 반환해 불필요한 객체 생성을 방지.
- **오토박싱 방지**: record 필드에 기본 타입(int, long)을 사용하면 오토박싱을 피할 수 있음.
</aside>

</br>

## 4. 비싼 객체의 캐싱

생성 비용이 높은 객체는 캐싱하여 재사용하는 것이 효율적이다. 제공된 텍스트는 **정규 표현식**(`Pattern`)을 예로

### 4-1) 문제 사례: 비효율적인 `Pattern` 사용

```java
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}
```

- **문제점**:
    - `String.matches`는 내부적으로 `Pattern.compile`을 호출하여 매번 새로운 `Pattern` 객체를 생성.
    - `Pattern.compile`은 정규 표현식을 유한 상태 머신(finite state machine)으로 변환하므로 생성 비용이 높음.
    - 생성된 `Pattern`은 한 번 사용 후 GC 대상이 됨.
- **성능 영향**:
    - 텍스트에 따르면, 길이 8의 문자열에서 비효율적인 버전은 1.1μs, 개선된 버전은 0.17μs 소요(약 6.5배 성능 차이).

### 4-2) 개선된 코드: `Pattern` 캐싱

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```

- **개선 효과**:
    - `Pattern` 객체를 정적 필드에 캐싱하여 재사용.
    - 성능 향상뿐만 아니라, `ROMAN` 필드를 명시적으로 선언해 코드의 의도를 명확히 드러냄.
- **추가 고려사항**:
    - **지연 초기화**: `ROMAN` 필드를 메서드 호출 시 초기화하면 불필요한 초기화를 피할 수 있지만, 코드 복잡도가 증가하고 성능 이점이 미미할 수 있음
    - **스레드 안전성**: `Pattern`은 불변이므로 멀티스레드 환경에서도 안전.

<aside>
💡

### **+) 정적 필드란?**

정적 필드(static field)는 자바에서 **클래스에 속하는 필드**로, 특정 인스턴스에 종속되지 않고 클래스 자체에 연결된 변수이다. 이는 static 키워드로 선언되며, 클래스가 로드될 때 한 번만 초기화되어 모든 인스턴스가 공유한다.

- **주요 특징**:
    - **클래스 수준 저장**: 인스턴스마다 별도로 존재하는 인스턴스 필드와 달리, 정적 필드는 클래스당 하나만 존재.
    - **메모리 공유**: 모든 객체가 동일한 정적 필드 값을 참조.
    - **초기화 시점**: 클래스가 JVM에 로드될 때 초기화(또는 지연 초기화 가능).
    - **접근 방식**: 클래스명.필드명으로 접근(예: RomanNumerals.ROMAN).
- **예시**:
    
    ```java
    public class Example {
        static int counter = 0; // 정적 필드
        int instanceVar = 0;    // 인스턴스 필드
    }
    ```
    
    - counter는 모든 Example 객체가 공유하며, instanceVar는 각 객체가 독립적으로 가짐.

**<제공된 코드에서 정적 필드의 역할>**

제공된 코드에서 Pattern 객체는 정적 필드로 선언된다:

```java
public class RomanNumerals {
    private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

    static boolean isRomanNumeral(String s) {
        return ROMAN.matcher(s).matches();
    }
}
```

- **정적 필드 ROMAN**:
    - private static final Pattern ROMAN: Pattern 객체를 저장하는 정적 필드.
    - static: 클래스가 로드될 때 한 번만 생성되어 모든 RomanNumerals 호출이 동일한 Pattern 객체를 공유.
    - final: 한 번 초기화된 후 변경 불가(불변성 보장).
    - private: 클래스 외부에서 직접 접근 불가, 캡슐화 유지.
- **왜 정적 필드를 사용했나?**:
    - **객체 재사용**: Pattern.compile은 생성 비용이 높아(정규 표현식을 유한 상태 머신으로 변환) 매번 호출하면 불필요한 객체 생성이 발생. 정적 필드로 캐싱하면 한 번만 생성되고 재사용됨.
    - **성능 향상**: 텍스트에 따르면, 캐싱 전 1.1μs → 캐싱 후 0.17μs로 약 6.5배 빨라짐.
    - **코드 명확성**: ROMAN이라는 명시적 이름으로 코드 의도를 드러냄(예: 정규 표현식이 로마 숫자 검증용임을 명확히 함).
    - **비싼 객체 캐싱**: Pattern처럼 생성 비용이 높은 객체를 정적 필드에 저장해 재사용.
- **불변 객체 활용**: Pattern은 불변이므로 정적 필드에 저장해도 스레드 안전(thread-safe).
- **메모리 효율**: 모든 호출이 동일한 객체를 참조하므로 메모리 낭비 방지.
</aside>

</br>

## 5. 어댑터(View) 객체의 재사용

어댑터(또는 뷰) 객체는 뒷단 객체를 감싸 다른 인터페이스를 제공하는 객체로, 상태를 거의 가지지 않으므로 재사용이 적합하다.

- **예시: `Map.keySet`**
    - `Map`의 `keySet()` 메서드는 키 집합을 `Set` 뷰로 반환.
    - 매 호출마다 새로운 `Set` 객체를 반환할 것처럼 보이지만, 실제로는 동일한 `Set` 인스턴스를 반환할 수 있음.
    - 반환된 `Set`은 가변이어도 기능적으로 동일(동일한 `Map`을 참조).
- **코드 예시**:

```java
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);
Set<String> keys1 = map.keySet();
Set<String> keys2 = map.keySet();
// keys1 == keys2가 true일 수 있음 (구현에 따라 다름)
keys1.remove("a"); // keys2와 map에도 반영됨

```

- 어댑터 객체는 새로운 인스턴스를 생성할 필요가 없으므로, 불필요한 객체 생성을 피해야 함.
- 반환된 뷰 객체가 가변이라면, 수정 시 모든 관련 객체에 영향을 미치므로 주의.
- 
</br>

## 6. 오토박싱의 함정

오토박싱(autoboxing)은 기본 타입(`int`, `long`)과 래퍼 클래스(`Integer`, `Long`)를 자동 변환하지만, 성능 문제를 유발할 수 있다.

### 6-1) 문제 사례: 오토박싱으로 인한 성능 저하

```java
private static long sum() {
    Long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++) {
        sum += i; // 오토박싱으로 Long 객체 생성
    }
    return sum;
}
```

- **문제점**:
    - `sum`이 `Long` 타입이므로, `sum += i`는 매번 새로운 `Long` 객체를 생성.
    - 약 2³¹(약 21억) 개의 `Long` 객체가 생성되어 GC 부담 증가.
    - 텍스트에 따르면, 이 코드는 6.3초 소요(개선 후 0.59초, 약 10배 차이).

- **개선된 코드**:

```java
private static long sum() {
    long sum = 0L;
    for (long i = 0; i <= Integer.MAX_VALUE; i++) {
        sum += i; // 기본 타입 사용
    }
    return sum;
}
```

- **개선 효과**:
    - 기본 타입(`long`) 사용으로 오토박싱 제거.
    - 객체 생성이 없어 메모리와 CPU 효율성 대폭 향상.

</br>

## 7. 객체 풀(Object Pool)의 신중한 사용

- **객체 풀의 정의**: 객체를 재사용하기 위해 유지하는 구조(예: 커넥션 풀, 스레드 풀).
- **유용한 경우**:
    - 데이터베이스 커넥션(`HikariCP`), 스레드 풀(`ExecutorService`) 등 생성 비용이 높은 객체.
- **주의점**:
    - 일반 객체(예: `String`, `Integer`)의 풀은 관리 비용이 더 클 수 있음.
    - 현대 JVM의 GC는 가벼운 객체를 효율적으로 처리하므로, 자체 객체 풀은 오히려 성능을 저하시킬 수 있음.
- **권장사항**:
    - 객체 풀 사용 전 성능 테스트(JMH, VisualVM)로 이점을 확인.
    - 불필요한 풀링은 코드 복잡도와 메모리 사용량을 증가시킴.

</br>

## 8. 성능 최적화와 가독성의 균형

- **과도한 최적화 주의**:
    - 불필요한 객체 생성을 피하려고 코드를 지나치게 복잡하게 만들면 가독성과 유지보수성이 저하됨.
    - 예: `StringBuilder`를 단일 문자열 연결에 사용할 필요는 없음(컴파일러가 최적화).
- **현대 JVM의 최적화**:
    - 자바 11+의 G1GC, ZGC는 가벼운 객체의 생성/회수를 효율적으로 처리.
    - Escape Analysis로 스택에 객체를 할당해 GC 부담 감소.
- **실제 테스트**:
    - 최적화 전후 성능을 프로파일링 도구로 측정.
    - 예: JMH로 `String` vs. `StringBuilder` 성능 비교.
