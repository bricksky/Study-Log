<aside>

쿠버네티스 클러스터를 운영할 때 가장 중요한 질문은 **“누가 어떤 권한으로 시스템에 접근하는가?”** 이다.
쿠버네티스 API 서버를 보호하는 3단계 보안 메커니즘과 그 중심에 있는 **RBAC**(Role-Based Access Control)에 대해 정리해보자.

</aside>

</br>

## 1. 쿠버네티스 API 접근 제어의 3단계 흐름

---

쿠버네티스 API 서버에 요청이 들어오면, 요청은 독립적으로 처리되는 것이 아니라, 아래 3단계를 통과해야한다.

<img width="976" height="443" alt="image" src="https://github.com/user-attachments/assets/36a21d60-9e62-4fd1-af4a-bf2c0af3f2cd" />


1. **Authentication (인증)**
    - **"당신은 누구인가?"**
        - 요청자가 유효한 사용자인지 확인한다.
        - ID/비밀번호, Bearer Token, X.509 인증서 등을 검증
2. **Authorization (인가)**
    - **"당신은 이 행동을 할 권한이 있는가?"**
        - 여기가 바로 **RBAC**가 작동하는 지점이다.
        - 인증된 사용자가 특정 리소스(Pod, Service 등)에 대해 특정 작업(Get, List, Create 등)을 수행할 수 있는지 판단한다.
3. **Admission Control (입장 제어)** 
    - **"권한은 있지만, 정책에 맞는 요청인가?"**
        - 권한과는 별개로, 요청 내용이 클러스터 정책(예: 리소스 할당량(Quota) 초과 여부, 특정 레지스트리 사용 강제 등)에 적합한지 최종 확인한다.

</br>
</br>

## 2. 정체성의 차이: User vs ServiceAccount

---

쿠버네티스에서 '**누가**'를 정의할 때, 일반 사용자와 시스템 계정을 엄격히 구분한다.

| **구분** | **User (일반 사용자)** | **ServiceAccount (서비스 계정)** |
| --- | --- | --- |
| **개념** | 클러스터 외부의 관리자/개발자 | Pod 내부 프로세스가 API와 통신할 때 사용 |
| **관리** | 쿠버네티스가 직접 관리하지 않음 (OIDC, 인증서 등 외부 연동) | 쿠버네티스 API 리소스로 직접 생성 및 관리 |
| **범위** | 보통 클러스터 전체 수준 | 특정 네임스페이스에 종속됨 |
| **특징** | 클러스터 내부에 'User' 객체가 존재하지 않음 | 클러스터 내부에 객체가 존재하며 토큰이 자동 생성됨 |

</br>

**💡 ServiceAccount가 필요한 이유**
포드 내에서 실행되는 앱(예: 모니터링 툴, CI/CD 에이전트 등)이 쿠버네티스 API를 호출해야 할 때, 각 앱마다 
필요한 최소한의 권한만 가진 '전용 신분증'을 주기 위해서이다.

쿠버네티스에서 ServiceAccount(서비스 계정)는 사람이 아닌 '포드 내에서 실행되는 프로세스'에게 부여하는 신분증이다. 개발 중인 백엔드 어플리케이션이나 연구실에서 사용하는 모니터링 툴들이 쿠버네티스 API 서버와 통신해야 할 때, 이 SA가 없으면 "누구인지 알 수 없어" 요청이 거부된다. 

</br>
</br>

## 3. ServiceAccount(SA)의 핵심과 작동 원리

---

"누구를 위한 계정인가?"

- **User Account (사용자 계정)**
    - 개발자, 운영자 등 **사람**을 위한 계정이다. 쿠버네티스 외부(Google, GitHub)에서 관리되는 경우가 많다.
- **ServiceAccount (서비스 계정)**
    - 포드를 위한 계정이다. 쿠버네티스 내부 리소스로 관리되며, 특정 네임스페이스에 종속된다.
 

</br>

### 3-1. 왜 ServiceAccount가 중요한가?

쿠버네티스 보안의 핵심인 '최소 권한의 원칙'을 지키기 위해서이다. 포드 내에서 실행되는 앱(모니터링 툴, CI/CD 에이전트 등)이 API 서버에 접근할 때, 각 앱마다 필요한 최소한의 권한만 담긴 '전용 신분증'을 부여한다.

1. **자동화된 인증**
    
    포드가 생성될 때 별도의 ID/PW 입력 없이도 API 서버에 자신을 인증할 수 있는 토큰을 자동으로 부여받음
    
2. **권한의 분리**
    - 웹 서버 포드: "로그만 볼 수 있는 권한"
    - 모니터링 포드: "모든 노드의 상태를 볼 수 있는 권한"
    - 자동 배포 포드: "새로운 포드를 생성할 수 있는 권한"
    
    각각의 목적에 맞는 전용 SA를 만들어 권한을 쪼개 줄 수 있다
    
</br>

### 3-2. SA의 작동 원리 (토큰 메커니즘)

포드가 SA의 신분을 증명하는 과정은 다음과 같다.

1. **SA 생성**
    - `kubectl create sa [이름]` 명령어로 생성한다.
2. **포드 할당**
    - 포드 정의(YAML)의 `spec.serviceAccountName` 필드에 SA 이름을 명시한다,
3. **토큰 주입 (Volume Mount)**
    - 포드가 생성될 때, 쿠버네티스는 해당 SA의 인증 토큰(JWT)을 포드 내부의 특정 경로에 파일 형태로 마운트
    - **경로:** `/var/run/secrets/kubernetes.io/serviceaccount/token`
4. **API 호출**
    - 포드 안의 앱은 이 파일을 읽어 API 요청 헤더(Authorization: Bearer <TOKEN>)에 담아 보낸다.
5. **검증**
    - API 서버는 받은 토큰을 보고 "아, 이 요청은 `dev` 네임스페이스의 `log-reader` SA가 보낸 거구나!"라고 인식한다.

</br>
</br>

## 4. RBAC의 핵심 구성 요소: "누가, 무엇을, 어떻게”

---

쿠버네티스 RBAC은 크게 '**권한을 정의하는 객체**'와 '**권한을 연결하는 객체**'로 나뉜다. 이를 이해하기 위해 가장 먼저 파악해야 할 개념은 **Scope**이다.

</br>

### 4-1. 권한의 정의: Role vs ClusterRole
<img width="1024" height="582" alt="image" src="https://github.com/user-attachments/assets/cf873f15-8c13-40e1-b25f-f27003e73a65" />



이 객체들은 "어떤 리소스에 대해 어떤 동작을 할 수 있는가"라는 규칙을 정의한다.

**① Role (네임스페이스 단위)**

- **특징:** 특정 네임스페이스 안에서만 유효하다.
- **용도:** 특정 팀이나 프로젝트 그룹에 해당 공간 내의 자원 관리 권한을 줄 때 사용한다.
- **예시:** "A팀 네임스페이스 내에서만 Pod를 생성하고 삭제할 수 있음."

**② ClusterRole (클러스터 단위)**

- **특징:** 클러스터 전체에 적용되는 권한이다. 네임스페이스에 속하지 않는 리소스도 제어할 수 있습니다.
- **용도:** 노드(Node), 스토리지 클래스(StorageClass) 등 클러스터 수준 리소스 관리
    - 모든 네임스페이스에 걸친 리소스 조회 (예: 모니터링 툴)
    - 여러 네임스페이스에서 공통으로 사용할 권한 정의

</br>

### 4-2. 권한의 부여: RoleBinding vs ClusterRoleBinding

정의된 역할을 실제 사용자에게 연결해주는 '매개체'이다.

<img width="1961" height="673" alt="Group 34207" src="https://github.com/user-attachments/assets/9765f3b9-318e-44a5-8068-2dfaa17853bf" />


**① RoleBinding**

- **작동 방식:** 특정 네임스페이스 내에서 `Role` 혹은 `ClusterRole`을 사용자/서비스 계정에 연결한다.
- **핵심:** `ClusterRole`을 `RoleBinding`으로 연결하면, 해당 사용자는 해당 네임스페이스 내에서만 그 권한을 갖게 된다. (권한 재사용의 핵심!)

**② ClusterRoleBinding**

- **작동 방식:** `ClusterRole`을 사용자에게 연결하여 클러스터 전체에 대한 권한을 부여한다.
- **위험성:** 관리자 권한을 줄 때 주로 사용하므로, 보안상 매우 신중하게 생성해야 한다.

</br>

### 4-3. 규칙을 구성하는 3요소

Role 정의 시 들어가는 `rules` 섹션의 세부 항목이다.

1. **apiGroups**
    - 리소스가 속한 API 그룹.
        - Core 그룹(Pod, Service 등)은 `""`로 표기
        - Apps 그룹(Deployment 등)은 `"apps"`로 표기
2. **resources**
    - 권한을 적용할 대상.
        - `"pods"`, `"services"`, `"deployments"` 등 (복수형 사용 권장)
3. **verbs**
    - 허용할 동작.
        - **조회:** `get`, `list`, `watch`
        - **변경:** `create`, `update`, `patch`, `delete`

</br>

### 4-4. 예시 사례로 이해하는 RBAC 조합

**시나리오 A: 특정 팀 전용 개발자 (Role + RoleBinding)**

> 상황: 숭실대 '네트워킹 연구실' 프로젝트 팀원에게 ICN-Lab 네임스페이스의 포드를 관리할 권한을 준다
> 
- **Role**
    - `networking-lab` NS 내의 `pods` 리소스에 대해 `get, list, watch, update` 권한 정의.
- **RoleBinding**
    - 위 Role을 사용자 `donghyun`에게 연결.
- **결과**
    - 나(동현)은 본인 팀의 포드는 관리할 수 있지만, 다른 팀의 설정은 볼 수 없다. (완벽한 격리)

**시나리오 B: 클러스터 전체 모니터링 (ClusterRole + ClusterRoleBinding)**

> 상황: Prometheus라는 모니터링 앱이 모든 네임스페이스의 자원 상태를 수집해야 한다.
> 
- **ClusterRole**
    - 클러스터 전체의 `nodes, pods, services`에 대해 `get, list, watch` 권한 정의.
- **ClusterRoleBinding**
    - 위 ClusterRole을 `prometheus`라는 **ServiceAccount**에 연결.
- **결과**
    - Prometheus 포드는 클러스터 어디든 돌아다니며 데이터를 수집할 수 있다.

**시나리오 C: 권한의 재사용 (ClusterRole + RoleBinding)**

> 상황: "포드 읽기 전용"이라는 공통 규칙(ClusterRole)을 만들어 두고, 각 팀별로 자기 동네에서만 쓰게 하고싶음
> 
- **ClusterRole**
    - 모든 포드에 대한 `get, list` 권한 정의 (공용 템플릿).
- **RoleBinding**
    - 이 공용 `ClusterRole`을 `team-a` 네임스페이스 안에서 특정 유저에게 연결.
- **결과**
    - 사용자는 공용 규칙을 부여받았지만, 본인의 네임스페이스(`team-a`) 내부에서만 읽기 권한이 작동
