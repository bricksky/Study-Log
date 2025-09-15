## 1. 의존 객체 주입의 중요성

아이템 5는 객체가 필요로 하는 자원(의존성)을 직접 생성하거나 정적으로 참조하는 대신, 외부에서 주입받는 설계 방식을 권장힌다. 이는 객체 지향 설계의 핵심 원칙인 느슨한 결합(loose coupling)과 높은 응집도(high cohesion)를 달성하여 코드의 유연성, 테스트 용이성, 재사용성을 극대화하는 데 목적이 있다. 

---

</br>

## 2) 문제점: 자원을 직접 명시하는 방식의 단점

자원을 직접 생성하거나 정적 유틸리티 클래스(예: 싱글턴, 정적 팩토리)를 통해 참조하면 다음과 같은 문제점이 발생한다:

- **유연성 부족**:
    - 클래스 내부에서 자원을 직접 생성하거나 정적으로 참조하면, 해당 자원을 교체하려면 클래스 코드를 수정해야한다. 예를 들어, 영어 사전(`EnglishLexicon`)을 한국어 사전(`KoreanLexicon`)으로 변경하려면 클래스 내부의 정적 참조를 수정해야 한다.
    - 이는 코드의 확장성과 유지보수성을 떨어뜨린다.
- **테스트 어려움**:
    - 단위 테스트에서 가짜(mock) 객체를 주입하거나 다른 설정을 적용하기 어렵다..
    - 이는 테스트 코드 작성 시 추가적인 리팩터링을 요구하거나 테스트 자체를 어렵게 만든다.
- **재사용성 저하**:
    - 특정 자원에 강하게 결합된 클래스는 다른 환경(예: 다른 데이터베이스, 다른 API 클라이언트)에서 재사용하기 어렵다.
    - 이는 코드의 모듈성을 해치고, 다양한 컨텍스트에서 클래스를 활용하기 어렵게 만든다.

**나쁜 예시**:

```java
// 정적 참조를 사용한 좋지 않은 예
public class SpellChecker {
    private static final Lexicon dictionary = new EnglishLexicon(); 
    // 특정 구현에 강하게 결합
    
    private SpellChecker() {} // 싱글턴 패턴
    public static final SpellChecker INSTANCE = new SpellChecker();

    public boolean isValid(String word) {
        return dictionary.isValid(word);
    }
}
```

위 코드의 문제점

- **EnglishLexicon에 강하게 결합되어 있어 다른 사전으로 교체하려면 클래스 코드를 수정해야 함**
    - 코드에서 dictionary 필드는 static final로 선언되어 EnglishLexicon의 인스턴스를 직접 생성하고 참조합한다. 이는 SpellChecker 클래스가 EnglishLexicon이라는 특정 구현에 강하게 결합(hard-coupled)되어 있음을 의미하는데,
        
        만약 다른 사전(예: KoreanLexicon, FrenchLexicon)을 사용하려면, SpellChecker 클래스 내부의 `private static final Lexicon dictionary = new EnglishLexicon();` 코드를 직접 수정해야 한다. 이는 개방-폐쇄 원칙(OCP, Open-Closed Principle)을 위반한다. OCP는 클래스가 확장에는 열려 있어야 하지만 수정에는 닫혀 있어야 한다고 명시니다.
        
        즉, 새로운 언어 사전을 추가하려면 SpellChecker 클래스의 소스 코드를 변경하고 재컴파일해야 하며, 이는 유지보수 비용을 증가시키고 코드 변경에 따른 버그 발생 가능성을 높인다.
        

- 테스트에서 MockLexicon 같은 가짜 객체를 주입할 방법이 없음
    - SpellChecker는 dictionary를 정적으로 참조하므로, 테스트 환경에서 EnglishLexicon을 대체할 방법이 없다. 단위 테스트에서는 실제 구현(EnglishLexicon) 대신 가짜 객체(Mock 또는 Stub)를 주입해 테스트를 단순화하고 외부 의존성을 제거하는 것이 일반적인데, 이 코드에서는 dictionary가 static final로 고정되어 있어, 테스트용 MockLexicon을 주입할 수 없다.
    - 실제 EnglishLexicon 객체가 외부 자원(예: 파일, 데이터베이스, 네트워크)에 의존한다면, 테스트 실행 시 외부 자원에 접근해야 하므로 테스트가 느려지고 불안정해진다.

- 싱글턴 패턴으로 인해 단일 인스턴스만 존재하므로, 여러 환경에서 다른 설정을 적용하기 어려움
    - SpellChecker는 싱글턴 패턴을 사용해 단일 인스턴스(INSTANCE)만 생성되도록 설계되었다. 이는 SpellChecker가 애플리케이션 전체에서 단 하나의 EnglishLexicon 인스턴스만 사용할 수 있음을 의미하는데, 실제 애플리케이션에서는 동일한 클래스(SpellChecker)를 서로 다른 설정(예: 영어 사전, 한국어 사전)으로 여러 인스턴스를 생성해 사용해야 할 때가 많다.
        
        예를 들어:
        
        - 다국어 지원 애플리케이션에서 영어와 한국어 사전을 동시에 사용해야 할 수 있다.
        - 테스트 환경과 프로덕션 환경에서 서로 다른 Lexicon 구현체를 사용해야 할 수 있다.
        
        싱글턴 패턴은 이러한 유연성을 제공하지 못하며, 모든 요청이 동일한 SpellChecker 인스턴스와 그에 연결된 EnglishLexicon을 사용해야 힌다.
        

</br>


## 3) 해결책: 의존 객체 주입(DI, Dependency Injection)

의존 객체 주입은 클래스가 필요로 하는 자원을 외부에서 주입받는 방식이다. 
이는 생성자, setter 메서드, 또는 팩토리 메서드를 통해 구현할 수 있다. 

- **유연성**:
    - 자원을 외부에서 주입받으므로, 런타임에 자원을 쉽게 교체하거나 변경할 수 있다. 
    예를 들어, `EnglishLexicon` 대신 `KoreanLexicon`을 주입해 동일한 `SpellChecker` 클래스를 재사용할 수 있다.
- **테스트 용이성**:
    - 테스트 환경에서 Mock 객체(예: Mockito, EasyMock)를 주입하여 단위 테스트를 쉽게 작성할 수 있다. 
    실제 데이터베이스 연결 대신 테스트용 인메모리 데이터베이스를 주입하는 것도 가능하다.
- **재사용성**:
    - 특정 자원에 의존하지 않도록 설계된 클래스는 다양한 환경에서 재사용 가능하다. 이는 모듈화된 코드를 작성하는 데 기여한다.
- **명확한 의존성 표현**:
    - 생성자나 메서드를 통해 의존성을 명시적으로 선언하므로, 클래스가 어떤 자원을 필요로 하는지 코드만 봐도 알 수 있다.

**나쁜 예시:**

```java
// 정적 참조를 사용한 좋지 않은 예
public class SpellChecker {
    private static final Lexicon dictionary = new EnglishLexicon(); 
    // 특정 구현에 강하게 결합
    
    private SpellChecker() {} // 싱글턴 패턴
    public static final SpellChecker INSTANCE = new SpellChecker();

    public boolean isValid(String word) {
        return dictionary.isValid(word);
    }
}
```

**좋은 예시**:

```java
// 의존 객체 주입을 사용한 예
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary); 
        // null 체크로 안정성 확보
    }

    public boolean isValid(String word) {
        return dictionary.isValid(word);
    }
}

// 사용 예
Lexicon englishDict = new EnglishLexicon();
SpellChecker englishChecker = new SpellChecker(englishDict);

Lexicon koreanDict = new KoreanLexicon();
SpellChecker koreanChecker = new SpellChecker(koreanDict);
```

- **인터페이스 기반 의존성**
    - SpellChecker는 Lexicon 인터페이스에 의존하며, 특정 구현체(EnglishLexicon, KoreanLexicon)에 직접 의존하지 않는다.
    - Lexicon 인터페이스는 사전의 동작을 정의하며, 구현체는 이를 구체화한다.
- **생성자 주입**:
    - SpellChecker는 생성자를 통해 Lexicon 객체를 주입받는다. 이는 필수 의존성을 명시적으로 드러내며, 객체 생성 후 의존성이 변경되지 않도록 불변성을 보장한다.
- **불변성 보장**:
    - dictionary 필드는 final로 선언되어 생성자에서 초기화된 후 변경되지 않는다. 이는 스레드 안전성과 코드 안정성을 높인다.
- **Null 체크**:
    - Objects.requireNonNull(dictionary)를 사용해 주입된 의존성이 null이 아닌지 확인힌다. 이는 런타임 오류를 방지하고 코드의 안정성을 강화한다.
- **다중 인스턴스 생성 가능**:
    - SpellChecker는 싱글턴이 아니므로, 필요에 따라 여러 인스턴스를 생성해 서로 다른 Lexicon 구현체를 사용할 수 있다.

</br>


## 4. 의존 객체 주입의 구현 방식

의존 객체 주입은 다양한 방식으로 구현할 수 있으며, 각 방식은 특정 상황에 적합하다.

### **4-1) 생성자 주입**

- 가장 일반적이고 권장되는 방식.
- 객체 생성 시 필수 의존성을 주입받아 불변성을 보장.
- 의존성이 명확히 드러나며, 객체가 생성된 후에는 의존성이 변경되지 않음.

```java
public class Client {
    private final Service service;

    public Client(Service service) {
        this.service = Objects.requireNonNull(service);
    }

    public void doSomething() {
        service.execute();
    }
}
```

### **4-2) Setter 주입**

- 선택적 의존성이나 런타임에 의존성을 변경해야 할 때 사용.
- 불변성을 보장하지 못하며, 객체가 완전히 초기화되지 않은 상태로 사용될 위험이 있음.

```java
public class Client {
    private Service service;

    public void setService(Service service) {
        this.service = Objects.requireNonNull(service);
    }
}
```

### **4-3) 인터페이스 주입**

- 특정 인터페이스를 구현한 메서드를 통해 의존성을 주입.
- 드물게 사용되며, 주로 프레임워크에서 활용.

```java
public interface ServiceInjector {
    void setService(Service service);
}
```

### **4-4) 팩토리 메서드**

- 객체 생성 로직을 캡슐화하고, 의존성을 주입받아 객체를 생성.
- 복잡한 객체 생성 시 유용.

```java
public class SpellCheckerFactory {
    public static SpellChecker create(Lexicon dictionary) {
        return new SpellChecker(dictionary);
    }
}
```

### **4-5) 프레임워크 사용**

- Spring, Guice, Dagger 같은 DI 프레임워크는 의존성 주입을 자동화하여 코드 작성량을 줄이고 관리 편의성을 높임.
- 예: Spring의 `@Autowired` 또는 `@Inject` 사용.

```java
@Component
public class SpellChecker {
    private final Lexicon dictionary;

    @Autowired
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = dictionary;
    }
}
```

</br>


## 5. 정적 참조 + 싱글턴 vs 의존 객체 주입 비교

의존 객체 주입은 객체 지향 설계에서 필수적인 패턴으로, 코드의 유연성, 테스트 가능성, 재사용성을 크게 향상시킵니다. 자원을 직접 생성하거나 정적 참조로 고정하는 대신, 생성자, setter, 또는 DI 프레임워크를 통해 외부에서 의존성을 주입받아 느슨한 결합을 유지하면, 유지보수와 확장이 쉬운 코드를 작성하는 데 큰 도움이 된다.

---

| **항목** | **나쁜 예시 (정적 참조 + 싱글턴)** | **의존 객체 주입** |
| --- | --- | --- |
| **유연성** | 특정 구현(`EnglishLexicon`)에 강하게 결합, 변경하려면 코드 수정 필요 | 인터페이스(`Lexicon`)에 의존, 런타임에 구현체 교체 가능 |
| **테스트 용이성** | Mock 객체 주입 불가능, 실제 구현에 의존 | Mock 객체 주입 가능, 외부 의존성 제거 |
| **인스턴스 관리** | 단일 인스턴스만 가능(싱글턴) | 다중 인스턴스 생성 가능 |
| **확장성** | 새로운 구현체 추가 시 코드 수정 필요 | 새로운 구현체 주입으로 확장 가능 |
| **동시성** | 전역 상태로 인해 동시성 문제 가능 | 인스턴스별 상태로 동시성 문제 없음 |
| **유지보수성** | 코드 변경 빈번, 유지보수 비용 증가 | 느슨한 결합으로 유지보수 용이 |
