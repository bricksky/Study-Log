## **0. 싱글턴 패턴이란?**

### **0-1) 싱글턴 패턴의 정의**

- 싱글턴(Singleton) 패턴은 **클래스의 인스턴스가 오직 하나만 생성**되도록 보장하고, 그 인스턴스에 전역적으로 접근할 수 있는 방법을 제공하는 디자인 패턴이다.
- 주로 **애플리케이션 전체에서 공유되는 단일 객체**가 필요한 경우 사용됩니다.

### **0-2) 싱글턴 패턴의 특징**

- **단일 인스턴스**: 클래스의 인스턴스가 하나만 존재하도록 제한.
- **전역 접근**: 정적 메서드나 필드를 통해 어디서든 해당 인스턴스에 접근 가능.
- **제한된 생성**: 생성자를 private으로 만들어 외부에서 임의로 객체를 생성하지 못하도록 함.
- **지연 초기화(Lazy Initialization)**: 필요할 때까지 인스턴스를 생성하지 않을 수 있음(구현 방식에 따라 다름).

### **0-3) 싱글턴 패턴의 사용 사례**

- **적합한 경우**:
    - 애플리케이션에서 단일 인스턴스만 필요한 경우 (예: 설정 관리, 캐시, 스레드 풀).
    - 리소스 사용을 최소화하려는 경우 (예: 데이터베이스 연결).
- **부적합한 경우**:
    - 상태를 가지는 객체에 싱글턴을 사용하면 예기치 않은 부작용 발생 가능.
    - 테스트가 어려워질 수 있음 (의존성 주입이 더 나은 경우가 많음).

### **0-4) 싱글턴 패턴의 문제점**

- **테스트 어려움**: 전역 상태를 가지므로 단위 테스트에서 상태 공유로 인해 예상치 못한 동작 가능.
- **의존성 은닉**: 클래스 내부에서 싱글턴을 참조하면 의존성이 코드에 명시적으로 드러나지 않음.
- **멀티스레드 문제**: 스레드 안전성을 보장하지 않으면 동시성 문제가 발생 가능.
- **확장성 제약**: 싱글턴은 상속이 어렵고, 인스턴스를 하나로 제한하므로 유연성이 떨어질 수 있음.

</br>


## **1. 싱글턴 구현 방법**

### **1-1) public static final 필드 방식**

- **설명**:
    - 생성자를 `private`으로 선언해 외부에서 인스턴스 생성을 막음.
    - `public static final` 필드로 단일 인스턴스를 제공.
    - 클래스 로딩 시점에 인스턴스가 즉시 생성됨 (Eager Initialization).
- **예시 코드**
    
    ```java
    public class Elvis {
        public static final Elvis INSTANCE = new Elvis();
    
        private Elvis() {
            // 외부에서 생성 방지
        }
    
        public void leaveTheBuilding() {
            System.out.println("Whoa baby, I'm outta here!");
        }
    }
    ```
    
- **사용 예시**:
    
    ```java
    Elvis elvis = Elvis.INSTANCE;
    elvis.leaveTheBuilding(); // 출력: Whoa baby, I'm outta here!
    ```
    
- **private 생성자의 역할**:
    - `private Elvis()`는 클래스의 생성자를 외부에서 호출하지 못하도록 제한한다.
    - 외부 코드에서 `new Elvis()`를 호출하면 컴파일 오류가 발생 (예: Elvis() has private access in Elvis).
    - 따라서 `Elvis.INSTANCE`를 통해서만 단일 인스턴스에 접근 가능하며, 싱글턴 패턴의 핵심인 **단일 인스턴스 보장**이 유지.
- **접근 가능성**:
    - 클라이언트는 `Elvis.INSTANCE`를 통해 `Elvis` 객체에 접근 가능.
    - 하지만 new Elvis()로 새 인스턴스를 만들 수 없음
- **장점**:
    - 구현이 매우 간단하고 직관적.
    - API가 명확히 싱글턴임을 드러냄 (`public static final` 필드).
    - 클래스 로딩 시 인스턴스 생성으로 스레드 안전성 보장.
- **단점**:
    - **지연 초기화 불가**: 클래스 로딩 시 무조건 인스턴스 생성, 초기화 비용이 큰 경우 비효율적.
    - **리플렉션 취약점**: 리플렉션을 통해 `private` 생성자를 호출할 수 있음.
    - **직렬화 문제**: `Serializable` 구현 시 역직렬화로 새 인스턴스 생성 가능.

- **생성자를 private이 아닌 다른 접근 제어자로 변경할 경우**
    
    만약 private Elvis()를 public, protected, 또는 기본 접근 제어자(default, 즉 아무 것도 지정하지 않음)로 변경하면, 외부에서 생성자를 호출할 수 있고,. 이 경우 싱글턴 패턴의 보장이 깨질 수 있다.
    

**public 생성자로 변경**

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();

    public Elvis() {
        // 외부에서 생성 가능
    }

    public void leaveTheBuilding() {
        System.out.println("Whoa baby, I'm outta here!");
    }
}
```

- **결과**:
    - 외부 코드에서 new Elvis()를 호출해 새로운 Elvis 인스턴스를 생성할 수 있음.
    - 예:
        
        ```java
        Elvis elvis1 = Elvis.INSTANCE; // 싱글턴 인스턴스
        Elvis elvis2 = new Elvis();   // 새로운 인스턴스 생성
        System.out.println(elvis1 == elvis2); // false (다른 인스턴스)
        ```
        
    - 이는 **싱글턴 패턴의 단일 인스턴스 보장**이 깨짐을 의미 
    Elvis.INSTANCE 외에도 다른 인스턴스가 생성될 수 있으므로, 싱글턴의 목적(단일 인스턴스 유지)이 무효화

### **1-2) 정적 팩토리 메서드 방식**

- **설명**:
    - 생성자를 `private`으로 선언하고, 정적 팩토리 메서드(`getInstance()`)를 통해 단일 인스턴스 제공.
    - 지연 초기화(Lazy Initialization) 구현 가능.
- **예시 코드**
    
    ```java
    public class Elvis {
        private static final Elvis INSTANCE = new Elvis();
    
        private Elvis() {
            // 외부에서 생성 방지
        }
    
        public static Elvis getInstance() {
            return INSTANCE;
        }
    
        public void leaveTheBuilding() {
            System.out.println("Whoa baby, I'm outta here!");
        }
    }
    
    ```
    
- **사용 예시**:
    
    ```java
    Elvis elvis = Elvis.getInstance();
    elvis.leaveTheBuilding(); // 출력: Whoa baby, I'm outta here!
    ```
    
- **장점**:
    - **유연성**: `getInstance()` 메서드 내부 로직을 변경해 싱글턴을 포기하거나 다른 인스턴스 제공 가능.
    - **지연 초기화 가능**: `INSTANCE`를 `getInstance()` 호출 시 생성하도록 수정 가능.
    - 제네릭 타입 활용 가능 (예: 싱글턴 팩토리).
- **단점**:
    - 지연 초기화 구현 시 스레드 안전성을 보장하려면 동기화(`synchronized`) 필요, 성능 저하 가능.
- **지연 초기화 예시**
    
    ```java
    public class Elvis {
        private static Elvis instance;
    
        private Elvis() {}
    
        public static synchronized Elvis getInstance() {
            if (instance == null) {
                instance = new Elvis();
            }
            return instance;
        }
    }
    ```
    

### +) 방법 1과 방법 2의 차이점

| **항목** | **방법 1: public static final 필드** | **방법 2: 정적 팩토리 메서드** |
| --- | --- | --- |
| **접근 방식** | 직접 필드 접근: `Elvis.INSTANCE` | 메서드 호출: `Elvis.getInstance()` |
| **인스턴스 노출** | `public static final` 필드로 인스턴스 직접 노출 | `private static final` 필드 + `public` 메서드로 간접 제공 |
| **지연 초기화** | 불가능 (클래스 로딩 시 즉시 초기화) | 가능 (메서드 내에서 `instance` 생성 제어) |
| **유연성** | 낮음 (인스턴스 생성 로직 변경 어려움) | 높음 (메서드 내부 로직 수정으로 싱글턴 정책 변경 가능) |
| **API 명확성** | 싱글턴 의도 명확 (`public final` 필드) | 메서드 호출로 덜 직관적일 수 있음 |
| **코드 간결성** | 더 간결 (메서드 없이 필드만 선언) | 약간 더 복잡 (메서드 추가 필요) |
| **확장 가능성** | 제한적 (필드 기반이라 확장 어려움) | 확장 가능 (예: 서브클래스별 인스턴스 반환, 제네릭 활용) |
| **스레드 안전성** | 항상 안전 (클래스 로딩 시 초기화) | 기본 구현은 안전, 지연 초기화 시 동기화 필요 |
- **접근 방식**
    - **방법 1**: 클라이언트가 Elvis.INSTANCE로 직접 필드에 접근. 코드가 간단하고 직관적.
    - **방법 2**: `Elvis.getInstance()` 메서드를 호출해야 함. 메서드 호출이 추가되지만, 인스턴스 접근을 캡슐화

.

- **유연성**
    - **방법 1**: INSTANCE가 public static final로 고정되어 있어, 인스턴스 생성 로직을 변경하거나 다른 인스턴스를 반환하기 어려움.
    - **방법 2**: `getInstance()` 메서드 내부에서 로직을 수정해 다른 인스턴스 반환, 서브클래스별 인스턴스 제공, 또는 싱글턴을 포기하는 등의 변경 가능.

### **1-3) 열거 타입(Enum) 방식**

- **설명**:
    - 자바의 `enum`을 사용해 싱글턴 구현. JVM이 `enum` 인스턴스를 하나만 생성하도록 보장.
    - 리플렉션, 직렬화, 스레드 안전성 문제를 자동 해결.
- **예시 코드**
    
    ```java
    public enum Elvis {
        INSTANCE;
    
        public void leaveTheBuilding() {
            System.out.println("Whoa baby, I'm outta here!");
        }
    }
    ```
    
- **사용 예시**:
    
    ```java
    Elvis elvis = Elvis.INSTANCE;
    elvis.leaveTheBuilding(); // 출력: Whoa baby, I'm outta here!
    ```
    
- **장점**:
    - **가장 안전**: JVM이 단일 인스턴스 보장, 리플렉션 공격 방지.
    - **직렬화 안전**: `enum`은 직렬화/역직렬화 시 추가 작업 없이 싱글턴 유지.
    - **스레드 안전**: 클래스 로딩 시 초기화로 스레드 안전성 보장.
    - 구현이 간결.
- **단점**:
    - **지연 초기화 불가**: enum은 클래스 로딩 시 초기화.
    - **상속 불가**: `enum`은 다른 클래스를 상속할 수 없음.
    - enum에 익숙하지 않은 개발자에게 생소할 수 있음.


</br>

## **2. 싱글턴 구현 시 주의사항**

- **리플렉션 공격**:
    - `private` 생성자는 리플렉션 API로 호출 가능, 싱글턴 보장 깨질 수 있음.
    - **대처 방법**:
        - 생성자에서 인스턴스 중복 생성 방지:
            
            ```java
            private Elvis() {
                if (INSTANCE != null) {
                    throw new RuntimeException("Use getInstance() to get the single instance.");
                }
            }
            ```
            
        - 하지만 `enum` 방식은 JVM이 리플렉션 공격을 원천적으로 차단.
- **직렬화 문제**:
    - 싱글턴 클래스가 `Serializable`을 구현하면, 역직렬화 시 새 인스턴스 생성 가능.
    - **대처 방법**:
        - `readResolve()` 메서드 구현:
            
            ```java
            private Object readResolve() {
                return INSTANCE;
            }
            ```
            
        - `enum` 방식은 직렬화 문제를 자동 해결.
- **멀티스레드 환경**:
    - 지연 초기화 시 동기화 필요.
    - 책에서는 **Initialization-on-demand holder idiom**을 간접 언급
        
        ```java
        public class Elvis {
            private Elvis() {}
        
            private static class SingletonHolder {
                private static final Elvis INSTANCE = new Elvis();
            }
        
            public static Elvis getInstance() {
                return SingletonHolder.INSTANCE;
            }
        }
        ```
        
        - 클래스 로딩의 지연 특성을 활용해 스레드 안전성과 지연 초기화 동시 제공.


</br>

## **3. 언제 어떤 방식을 선택할까?**

- **public static final 필드**:
    - 간단한 싱글턴이 필요하고, 지연 초기화가 필요 없을 때.
    - 예: 상수 객체, 설정 객체.
- **정적 팩토리 메서드**:
    - 지연 초기화가 필요하거나, 싱글턴 로직을 나중에 변경할 가능성이 있을 때.
    - 예: 동적 설정 로딩, 팩토리 메서드 활용.
- **열거 타입**:
    - 리플렉션, 직렬화, 스레드 안전성을 모두 보장하고 싶을 때.
    - **조슈아 블로크의 권장 방식**: 가장 간결하고 안전.
    - 예: 로깅, 데이터베이스 연결 관리.


</br>

## **4. 싱글턴의 대안**

- 싱글턴은 전역 상태를 만들므로, 가능하면 **의존성 주입(Dependency Injection)** 사용 권장.
    - 예: Spring 프레임워크에서 싱글턴 스코프 빈 사용.
- 팩토리 패턴이나 서비스 로케이터로 단일 인스턴스 제공 가능.
- 싱글턴 사용 시 테스트 어려움과 의존성 은닉 문제를 고려해야 함.
