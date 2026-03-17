### @EnableScheduling은 왜 mail-app에만 있을까? 멀티 모듈 구조와 의존성의 방향

> 지난 주말, @orijoon98 형과 이야기를 나누던 중 형도 자신의 프로젝트에 멀티 모듈을 적용했다는 말을 꺼냈다. "멀티 모듈의 장점이 뭐야?" 라고 물어봤을 때, "구분하기 편하다", "관리가 쉽다" 정도밖에 대답하지 못했다. 마침 작업 중이던 매일메일 서버가 멀티 모듈 구조였고, 코드를 처음부터 뜯어보기로 했다. 이 글은 그 과정에서 정리한 내용이다. 결론부터 말하면, 멀티 모듈의 핵심 장점은 **각 모듈이 자신의 책임만 갖게 함으로써 변경의 영향 범위를 제한하는 것**이다.
> 

---

**요약**

- 배경: 멀티 모듈 구조에서 @EnableScheduling이 진입점 모듈에만 선언된 이유 파악
- 핵심: 의존성의 방향은 계층의 상하가 아닌 "사용하는 쪽 → 사용되는 쪽"
- 결과: 각 모듈의 역할과 책임이 명확히 분리되고, core 모듈은 독립적으로 테스트 및 재사용 가능

---

</br>

### 1. 멀티 모듈 구조란?

멀티 모듈 프로젝트는 하나의 큰 프로젝트를 기능 단위로 나눠 여러 개의 서브 모듈로 관리하는 방식이다. 매일메일 서버의 모듈 구성은 다음과 같다.

```
maeilmail-server (root)
├── mail-core     — 핵심 도메인/비즈니스 로직 (라이브러리)
├── mail-api      — REST 컨트롤러 (의존: mail-core)
├── mail-admin    — 어드민 API + 스케줄러 (의존: mail-core)
├── mail-batch    — 배치 잡 (의존: mail-core)
├── wiki-core     — wiki 도메인 로직 (라이브러리)
├── wiki-api      — wiki REST 컨트롤러 (의존: wiki-core)
└── mail-app      — Spring Boot 진입점 (의존: mail-api, wiki-api, mail-admin)
```

각 모듈의 build.gradle을 보면 의존 관계가 명확하게 드러난다.

```groovy
// mail-app/build.gradle
dependencies {
    implementation project(':mail-api')
    implementation project(':wiki-api')
    implementation project(':mail-admin')
}

// mail-api/build.gradle
dependencies {
    implementation project(':mail-core')
}
```

mail-core의 build.gradle은 아무 의존도 없다. 아무것도 사용하지 않기 때문이다.

---

</br>

### 2. 실행 가능한 모듈은 mail-app 하나뿐

멀티 모듈에서 모든 모듈이 실행 가능한 JAR를 만드는 것은 아니다. build.gradle 루트 설정을 보면 모듈 타입에 따라 빌드 방식이 다르게 적용된다.

```groovy
// application 타입 모듈 — 실행 가능한 fat JAR 생성
configureByTypeHaving(project, ['boot', 'mvc', 'application']) {
    bootJar { enabled = true }
    jar     { enabled = false }
}

// lib 타입 모듈 — 일반 JAR (클래스 파일 묶음)
configureByTypeHaving(project, ['boot', 'lib']) {
    bootJar { enabled = false }
    jar     { enabled = true }
}
```

- mail-app: bootJar 활성화 → Spring Boot가 의존하는 모든 모듈을 포함한 실행 가능한 JAR 생성
- 나머지 모듈: 일반 jar → 클래스 파일 묶음일 뿐, 단독 실행 불가

mail-core, wiki-core 등은 부품 공장이고, mail-app은 그 부품들을 조립해 완성차를 만드는 공장이다.

---

</br>

### 3. 의존성의 방향: "상위가 하위를 사용한다"

처음 멀티 모듈을 접하면 이런 의문이 생긴다.

> "가장 상위 개념인 mail-app이 하위 개념인 mail-core를 의존한다고? 하위가 상위를 의존해야 하는 거 아닌가?"
> 

여기서 "의존한다"는 말의 의미를 정확히 짚어야 한다.

> "A가 B를 의존한다" = "A가 B를 사용한다" = "A는 B 없이 동작할 수 없다"
> 

즉, 의존성의 방향은 계층의 상하 관계가 아니라 사용 관계다.

```
mail-app   →  mail-api  →  mail-core
(사용한다)     (사용한다)     (아무것도 사용 안 함)
```

mail-core는 mail-app이 존재하는지조차 모른다. 덕분에 mail-core는 실행 환경에 종속되지 않고 독립적으로 테스트하거나 다른 프로젝트에서 재사용할 수 있다.

---

</br>

### 4. @EnableScheduling은 왜 mail-app에만 있는가?

```java
@EnableScheduling
@SpringBootApplication(scanBasePackages = {MAEIL_MAIL, MAEIL_WIKI})
public class MaeilMailApplication {
    public static void main(String[] args) {
        SpringApplication.run(MaeilMailApplication.class, args);
    }
}
```

@EnableScheduling은 Spring의 스케줄링 기능을 활성화하는 스위치다. 이 어노테이션이 없으면 @Scheduled가 아무리 많이 선언되어 있어도 아무것도 실행되지 않는다.

실제 스케줄러들은 여러 모듈에 흩어져 있다.

| 모듈 | 스케줄러 | 역할 |
| --- | --- | --- |
| mail-core | SendQuestionScheduler | 평일 07:00 일간 구독자 메일 발송 |
| mail-core | SendWeeklyQuestionScheduler | 주간 구독자 메일 발송 |
| mail-core | ResendQuestionScheduler | 실패 메일 재발송 |
| mail-admin | AdminReportScheduler | 어드민 리포트 발송 |

각 라이브러리 모듈은 "언제 실행할지"를 선언만 하고, 실제로 **켜는 역할은 mail-app**이 담당한다. 이것이 가능한 이유는 @SpringBootApplication의 scanBasePackages에 maeilmail과 maeilwiki 두 패키지가 모두 등록되어 있기 때문이다. mail-app이 기동될 때 모든 모듈의 @Component, @Scheduled를 전부 인식하고 활성화한다.

```java
@SpringBootApplication(scanBasePackages = {MAEIL_MAIL, MAEIL_WIKI})
// maeilmail + maeilwiki 하위의 모든 컴포넌트 스캔
```

---

</br>

### 5. 결론: 멀티 모듈의 장점은 변경의 영향 범위를 제한하는 것

멀티 모듈 구조의 핵심은 각 모듈이 자신의 책임만 갖는 것이다.

- core: 도메인 로직만 알고, 외부 세계(HTTP, 실행 환경)를 모른다
- api / -admin: HTTP 요청을 받아 core의 로직을 호출한다
- mail-app: 모든 모듈을 조립하고, 애플리케이션 수준의 설정(@EnableScheduling 등)을 담당한다

의존성은 항상 구체적인 것(app) → 추상적인 것(core) 방향으로 흐른다. core가 app을 모르기 때문에 core는 독립적으로 테스트 가능하고, app은 core를 자유롭게 교체하거나 확장할 수 있다. 단순히 파일을 나눠놓은 것이 아니라, 변경의 영향 범위를 제한하기 위한 설계적 선택이다.
