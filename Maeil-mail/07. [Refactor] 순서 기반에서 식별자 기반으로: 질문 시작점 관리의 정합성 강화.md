### **"15번째 질문"은 영원히 15번째일까? 데이터 정합성을 고려한 시작점 설계 변경**

> 🔗 **관련 PR-1**: [카테고리별 초기 문제 번호 관리 방식을 개선한다](https://github.com/maeil-mail/maeil-mail-be/pull/295)
> 
> 🔗 **관련 PR-2**: [카테고리별 시작 질문 기준을 순서에서 식별자 참조로 변경한다](https://github.com/maeil-mail/maeil-mail-be/pull/301)

백엔드 질문 카테고리의 시작 번호를 단순히 15L이라는 하드코딩된 시퀀스 값으로 관리하던 방식에서, 데이터베이스의 외래 키 제약 조건을 활용한 정책 테이블 구조로 전환했다. 초기에는 매직 넘버를 제거하는 수준의 리팩토링으로 시작했으나, 코드 리뷰 과정에서 데이터 삭제나 변경 시 발생할 수 있는 순서 밀림 문제를 식별했다. 이에 단순 순서가 아닌 특정 질문의 식별자를 직접 참조하게 함으로써, 서비스 운영의 유연성과 데이터 무결성을 동시에 확보한 과정을 기록한다.

---

**리팩토링 요약**

- **배경:** Enum에 정의된 하드코딩된 시퀀스 번호의 정합성 위기 식별
- **핵심:** CategoryPolicy 엔티티 도입 및 질문 식별자 기반의 외래 키 설정
- **결과:** 질문 데이터 삭제 시에도 시작점 유지 및 데이터베이스 계층의 참조 무결성 확보



</br>

### **1. 문제 상황: 하드코딩된 시퀀스의 불안정성**

기존 서비스에서는 백엔드 질문의 시작점을 관리하기 위해 QuestionCategory라는 Enum 클래스에 직접 시퀀스 번호를 주입하여 사용하고 있었다.

**<기존 코드>**

```java
public enum QuestionCategory {
    FRONTEND("FE", 0L), 
    BACKEND("BE", 15L);

    private final String description;
    private final Long initialSequence;

    QuestionCategory(String description, Long initialSequence) {
        this.description = description;
        this.initialSequence = initialSequence;
    }
    
    public Long getInitialSequence() {
        return initialSequence;
    }
    // ... 생략
}
```

---

> **😀 하늘님:** 15라는 값이 식별자를 의미하는 것이 아니라 순서를 의미하기 때문에 15번째 질문지가 저희가 의도했던 질문지가 아니게 될 경우에 문제가 발생할 것 같아요. 백엔드 질문지가 한 개 사라져서 15번째가 의도한 질문지가 아니게 된다면 이 요구사항을 만족하지 않다고 생각해요.
> 

이 방식은 백엔드 질문 중 초기 테스트용 데이터나 낮은 품질의 질문을 건너뛰고 15번째 질문부터 사용자에게 제공하려는 의도에서 시작되었다. 하지만 15라는 숫자는 데이터베이스 내의 절대적인 위치가 아니라 상대적인 순서를 의미한다. 만약 관리자가 앞선 순서의 질문을 하나 삭제할 경우, 기존의 16번째 질문이 15번째로 당겨지며 의도하지 않은 질문이 사용자에게 먼저 노출되는 문제가 발생한다.



</br>

### **2. 해결 설계: 정책 기반의 식별자 참조**

코드 리뷰를 통해 단순한 숫자 기반의 관리가 가져올 수 있는 위험 요소들을 구체화했다. 특히 운영 상황에 따라 질문의 시작점은 언제든 변경될 수 있으며, 이때 기존 사용자들의 경험이 훼손되지 않아야 한다는 비즈니스 요구사항을 재확인했다.

> **하늘님:** 백엔드 질문지가 15번부터 시작했던 맥락을 공유드리자면, 서비스 초기에 테스트용 질문지를 넣어놨었는데요. 사용자가 생기면서 새로운 사용자는 퀄리티가 높은 질문지부터 전달해야겠다고 판단했었습니다. 그래서 백엔드 시퀀스를 점프시키는 결정을 내렸었어요.
> 

이에 대해 나는 기술적 해결책뿐만 아니라 운영 효율성까지 고려한 새로운 설계를 제안했다.

> **나:** 단순히 매직 넘버를 없애는 기술적 이슈로만 접근했었는데, 15번이라는 숫자에 초기 콘텐츠 퀄리티 관리와 사용자 경험 확보라는 중요한 비즈니스 의도가 담겨있는지는 미처 몰랐습니다! 말씀해주신 요구사항을 충족하기 위해 별도의 설정 테이블을 만들어서 관리하는 방식을 제안드리고 싶습니다.
> 

최종적으로 도출된 설계는 시작점을 숫자가 아닌 질문의 고유 식별자로 관리하고, 이를 데이터베이스 계층에서 외래 키로 연결하는 방식이다.

</br>

**<수정 코드>**

**Step 1. Enum 클래스의 역할 정화**

Enum에서 관리하던 비즈니스 로직과 매직 넘버를 제거하고, 순수한 카테고리 구분 역할만 수행하도록 변경했다.

```java
public enum QuestionCategory {
    FRONTEND("FE"), 
    BACKEND("BE");

    private final String description;

    QuestionCategory(String description) {
        this.description = description;
    }
}
```

</br>

**Step 2. 정책 관리를 위한 엔티티와 리포지토리 생성**

카테고리별 시작 질문을 관리하는 CategoryPolicy 엔티티를 설계했다. 단순히 ID 값을 필드로 가지는 것이 아니라, Question 엔티티와 연관관계를 맺어 데이터베이스 수준에서 제약 조건을 생성했다.

```java
@Entity
public class CategoryPolicy {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, unique = true)
    private QuestionCategory category;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "start_question_id", nullable = false)
    private Question startQuestion;
}

public interface CategoryPolicyRepository extends JpaRepository<CategoryPolicy, Long> {
    Optional<CategoryPolicy> findByCategory(QuestionCategory category);
}
```



</br>

### **3. 도메인 모델과 서비스 로직의 연결**

정책이 데이터베이스로 이동함에 따라, 구독 엔티티는 스스로 시작 지점을 결정하는 대신 외부에서 결정된 값을 주입받는 방식으로 변경되었다.

**Subscribe 엔티티 수정:**

```java
// 기존: 생성자 내부에서 category.getInitialSequence() 호출
// 변경: 생성자 파라미터로 nextQuestionSequence를 직접 주입받음
public Subscribe(String email, QuestionCategory category, SubscribeFrequency frequency, Long nextQuestionSequence) {
    this.email = email;
    this.category = category;
    this.nextQuestionSequence = nextQuestionSequence;
    this.token = UUID.randomUUID().toString();
    this.frequency = frequency;
}
```
</br>

**SubscribeService 로직 통합:**

최종적으로 서비스 계층에서 정책을 조회하고, 해당 질문이 현재 전체 질문 중 몇 번째 위치인지 동적으로 계산하여 주입한다.

```java
private void subscribe(QuestionCategory category, SubscribeRequest request) {
    SubscribeFrequency frequency = SubscribeFrequency.from(request.frequency());

    // 1. 카테고리 정책 조회
    CategoryPolicy policy = categoryPolicyRepository.findByCategory(category)
            .orElseThrow(() -> new IllegalArgumentException("정책이 설정되지 않았습니다."));

    // 2. 시작 질문 식별자를 기반으로 현재 시퀀스 계산
    Long startQuestionId = policy.getStartQuestion().getId();
    long nextSequence = questionRepository.countByCategoryAndIdLessThan(category, startQuestionId);

    // 3. 계산된 시퀀스 주입 및 저장
    Subscribe subscribe = new Subscribe(request.email(), category, frequency, nextSequence);
    subscribeRepository.save(subscribe);
}
```


</br>

### **4. 왜 이렇게 설계했는가?**

**4.1 데이터베이스 계층의 보호막, 외래 키**

단순히 Long 타입의 ID 값을 저장하는 방식과 비교했을 때, 객체 참조를 통한 외래 키 설정은 강력한 안전장치가 된다. 이는 서비스 로직에서 일일이 체크하지 않아도 데이터 정합성을 보장하며, 관리자의 실수로 인한 시스템 장애 가능성을 낮춘다.

**4.2 동적 시퀀스 계산의 이점**

ID 기반으로 시퀀스를 역산하는 방식은 데이터 삭제에 매우 강하다. 만약 1번부터 14번 질문 중 일부가 삭제되더라도, count 쿼리는 실제 존재하는 데이터의 개수만을 반환하므로 사용자는 항상 의도된 질문부터 구독을 시작할 수 있다.




</br>

### **5. 결론: 견고한 아키텍처는 정합성에서 시작된다**

단순한 하드코딩 리팩토링에서 시작된 이번 작업은, 리뷰어와의 소통을 통해 시스템의 신뢰도를 높이는 아키텍처 개선으로 확장되었다. 

백엔드 개발자는 단순히 기능을 구현하는 것을 넘어, 데이터의 생명주기와 변경 가능성을 고려하여 어떤 상황에서도 무너지지 않는 데이터 구조를 설계해야 함을 다시금 체감했다.
