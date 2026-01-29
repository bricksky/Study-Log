### "공정한 성능 비교를 위하여" : 단일 모듈에서 멀티모듈 아키텍처로

> 🔗 **관련 PR**: [Kafka, Redis 기반 실시간 위치 스트리밍 인프라 및 기능 구현](https://github.com/ICN-Soongsil/itrc-project/pull/8)





> 이전 프로젝트인 매일메일을 진행하며 멀티모듈 아키텍처를 처음 접했을 때는 이를 단순히 프로젝트를 깔끔하게 유지하기 위한 정리 기법 정도로만 이해했다. 당시에는 구조의 복잡성을 감당할 엄두가 나지 않아 도입을 미루었으나, 이번 연구 과제를 수행하며 단일 모듈 구조의 치명적인 한계를 마주했다. RDBMS, Kafka, Redis Streams라는 세 가지 기술 스택의 성능을 정밀하게 대조해야 하는 상황에서, 모든 라이브러리가 뒤섞인 환경은 실험의 신뢰성을 위협하는 변수 덩어리였기 때문이다. 이에 코드의 물리적 격리를 통해 실험의 순수성을 확보하고자 멀티모듈로 아키텍처를 전환한 과정을 정리한다.

---

</br>

**리팩토링 요약**

- 배경: 기술 스택별 독립 실행 환경 확보 및 성능 측정 시 외부 변수 차단 필요
- 핵심: 관심사 분리를 통한 모듈별 물리적 격리 및 독립적인 런타임 환경 구축
- 결과: 기술 스택 간 간섭 차단, 연구 데이터의 객관성 확보, 유지보수 효율성 향상

---

</br>

### 1. 문제 상황: "실험 도구의 혼재와 데이터 오염"

연구를 진행함에 따라 도메인 로직과 각기 다른 기술 스택의 설정 코드가 하나의 소스 폴더 안에 공존하게 되었다. 이러한 구조는 다음과 같은 기술적 한계를 드러냈다.

단일 모듈 구조의 한계:

- 의존성 전이와 클래스패스 오염: RDBMS 방식만 테스트할 때도 Kafka 브로커 관련 라이브러리가 메모리에 로드되어 불필요한 자원을 점유했다.
- 설정 간섭의 위험: 특정 스택을 위한 유효성 검사나 직렬화 설정이 다른 스택의 실행 환경에 영향을 주어 성능 지표를 왜곡할 가능성이 있었다.
- 빌드 효율 저하: 특정 구현체만 수정했음에도 관계없는 다른 모듈까지 포함된 전체 프로젝트를 다시 빌드해야 하므로 개발 생산성이 낮아졌다.

</br>

### 2. 물리적 기반 구축

멀티모듈 설계를 위해 가장 먼저 settings.gradle에 모듈명을 등록했다. 이는 프로젝트의 전체 지도를 그리는 작업이다.

- **Step 1. 모듈 구성 정의 (settings.gradle)**

```java
`rootProject.name = 'itrc-project'

include 'itrc-common'
include 'itrc-api-kafka'
include 'itrc-api-rdbms'
include 'itrc-api-redis-streams'`
```

- **Step 2. 물리적 디렉토리 생성**

설정 파일 선언만으로는 Gradle이 구조를 인식하지 못하므로, 터미널을 통해 실제 폴더와 각 모듈의 build.gradle 파일을 생성했다.

```bash
# 하위 모듈 폴더 생성
mkdir itrc-common itrc-api-kafka itrc-api-rdbms itrc-api-redis-streams

# 각 모듈을 프로젝트로 인식시키기 위한 최소 설정 파일 생성
touch itrc-common/build.gradle itrc-api-kafka/build.gradle itrc-api-rdbms/build.gradle itrc-api-redis-streams/build.gradle
```

</br>

### 3. 루트 프로젝트 설정: "컨트롤 타워의 구축"

최상위 build.gradle은 하위 모든 모듈이 공통으로 사용할 기술 스택의 버전과 기본 도구를 중앙에서 관리한다.

- **루트 build.gradle 설정**

```java
`plugins {
    id 'java'
    id 'org.springframework.boot' version '4.0.2' apply false
    id 'io.spring.dependency-management' version '1.1.7'
}

allprojects {
    group = 'ICN'
    version = '0.0.1-SNAPSHOT'
    repositories { mavenCentral() }
}

subprojects {
    apply plugin: 'java'
    apply plugin: 'org.springframework.boot'
    apply plugin: 'io.spring.dependency-management'

    java { toolchain { languageVersion = JavaLanguageVersion.of(17) } }

    dependencies {
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }
}`
```
</br>

### 4. 코드 이관과 공통 모듈 설계: "데이터 규격의 통일"

가장 먼저 수행한 것은 모든 실험의 기준점이 되는 데이터 규격인 LocationRequest를 공통 모듈로 이관하는 것이었다.

- **Step 1. 공통 모듈 설정 (itrc-common/build.gradle)**

이 모듈은 실행 파일이 아닌 라이브러리 역할을 수행하므로 bootJar를 비활성화했다.

```java
`plugins {
    id 'java-library'
    id 'org.springframework.boot'
    id 'io.spring.dependency-management'
}

dependencies {
    // api를 사용하여 이 모듈을 참조하는 다른 모듈에 의존성을 전파
    api 'org.springframework.boot:spring-boot-starter-validation'
    api 'com.fasterxml.jackson.core:jackson-databind'
}

bootJar { enabled = false }
jar { enabled = true }`
```

- **Step 2. 코드 이관**
    - 기존 위치: src/main/java/ICN/itrc_project/dto/LocationRequest.java
    - 변경 위치: itrc-common/src/main/java/ICN/itrc_project/dto/
    - 처리 결과: 루트 경로의 기존 src 폴더를 삭제하여 모든 코드가 모듈 내부에 존재하도록 강제했다.

</br>

### 5. 실행 모듈 구현과 트러블슈팅: "의존성 격리"

각 실행 모듈은 자신에게 꼭 필요한 부품만 장착한다. itrc-api-kafka 모듈을 구축하며 발생한 의존성 인식 문제를 해결한 최종 코드다.

- **itrc-api-kafka/build.gradle 최종 설정**

```java
dependencies {
    implementation project(':itrc-common') // 공통 DTO 연결

    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.springframework.boot:spring-boot-starter-kafka'

    // 테스트 환경: 인식 오류 방지를 위해 버전을 직접 명시
    testImplementation 'org.springframework.kafka:spring-kafka-test'
    testImplementation 'org.testcontainers:junit-jupiter:1.19.3'
    testImplementation 'org.testcontainers:redis:1.19.3'
}
```

**트러블슈팅 포인트:**

- 메서드 인식 오류: bootJar나 implementation 메서드를 찾지 못하는 에러는 해당 플러그인이 plugins 블록에 명시되지 않아 발생했다. 각 모듈에 필요한 플러그인을 명시적으로 선언하여 해결했다.
- 저장소 및 버전 오류: Testcontainers 라이브러리 인식 실패를 해결하기 위해 루트에서 mavenCentral 저장소를 재확인하고, BOM 방식 대신 버전을 직접 기입하여 실험 환경의 안정성을 확보했다.

---

</br>

### 6. 왜 이렇게 설계했는가?

- **실험 변수의 완벽한 통제:**

멀티모듈을 통해 의존성을 격리함으로써, RDBMS 모듈 실행 시 Kafka 엔진이 자원을 점유하는 등의 외부 요인을 원천 차단했다. 이는 숭실대학교 연구 과제가 추구하는 객관적인 벤치마킹의 토대가 된다.

- **데이터 규격의 중앙 집중화:**

itrc-common에 위치한 LocationRequest를 모든 API 모듈이 강제로 참조하게 함으로써, 데이터 포맷 차이로 인한 성능 변수를 제거했다. 이제 모든 실험체는 동일한 입력을 바탕으로 내부 처리 메커니즘의 차이만 드러내게 된다.

</br>

### 7. 결론

<img width="6647" height="3400" alt="ICN-kafka_redis-2026-01-27-082000" src="https://github.com/user-attachments/assets/ff04e636-28cd-43be-9e55-5db00cc2341f" />


이번 멀티모듈 전환은 단순히 코드를 정리하는 작업을 넘어, 연구의 목적을 가장 공학적으로 달성하기 위한 기반 시스템을 구축하는 과정이었다. 복잡한 Gradle 설정과 플러그인 충돌을 해결하며 얻은 이 구조는 향후 진행될 수많은 성능 측정 테스트의 신뢰성을 보장하는 강력한 증거가 될 것이다. 이제 격리된 각 실험실에서 본격적으로 데이터의 흐름을 분석할 준비를 마쳤다.




