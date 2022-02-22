# 3장. 코드 구성하기

# 패키지 구조

## 1. 계층으로 구성하기

```java
buckpal
|-- domain
|   |-- Account
|   |-- Activity
|   |-- AccountRepositoryy
|   |-- AccountService
|-- persistence
|   |-- AccountRepositoryImpl
|-- web
    |-- AccountController
```

웹 계층(web), 도메인 계층(domain), 영속성 계층(persistence)로 구분했다.

### 1.1 계층으로 구성한 패키지 구조가 최적의 구조가 아닌 이유

**이유1.** 어플리케이션의 기능 조각(functional slice)이나 **특성(feature)을 구분 짓는 패키지 경계가 없다.**

- 추가적인 구조가 없다면, 서로 연관되지 않은 기능들끼리 예상하지 못한 부수효과를 일으킬 수 있는 클래스들의 묶음으로 변모할 수 있다.

**이유2.** 어플리케이션이 **어떤 유스케이스들을 제공하는지 파악할 수 없다.**

- 서비스 내의 어떤 메서드가 특정 기능에 대한 책임을 수행하는지 찾아야 한다.

**이유3. 패키지 구조를 통해 아키텍처를 파악할 수 없다.**

- 어떤 기능이 웹 어댑터에서 호출되는지, 영속성 어댑터가 도메인 계층에 어떤 기능을 제공하는지 한눈에 볼 수 없다.

## 2. 기능으로 구성하기

```java
buckpal
|-- account
    |-- Account
    |-- AccountController
    |-- AccountRepository
    |-- AccountRepositoryImpl
    |-- SendMoneyService
```

account 패키지로 묶고 계층 패키지를 없앴다.

- 장점
    - package-private 접근 수준으로 각 기능 사이의 불필요한 의존성을 방지할 수 있다.
- 단점
    - 가시성을 떨어뜰인다.
    - package-private 접근 수준을 이용해 도메인 코드가 실수로 영속성 코드에 의존하는 것을 막을 수 없다.

## 3. 아키텍처적으로 표현력 있는 패키지 구조

```java
buckpal
|-- account
    |-- adapter
    |   |-- in
    |   |   |-- web
    |   |       |-- AccountController
    |   |-- out
    |   |   |-- persistence
    |   |       |-- AccountPersistenceAdapter
    |   |       |-- SpringDataAccountRepository
    |-- domain
    |   |-- Account
    |   |-- Activity
    |-- application
        |-- SendMoneyService
        |-- port
            |-- in
            |   |-- SendMoneyUseCase
            |-- out
            |   |-- LoadAccountPort
            |   |-- UpdateAccountStatePort
```

### 3.1 핵사고날 아키텍처 패키지 구조

Account와 관련된 유스케이스는 모두 account 패키지 안에 있다.

- domain
    - 도메인 모델 (Account)
- application
    - 도메인 모델을 둘러싼 서비스 계층 (SendMoneyService)
    - 인커밍 포트 인터페이스 (SendMoneyUseCase)
    - 아웃고잉 포트 인터페이스 (LoadAccountPort, UpdateAccountStatePort)
- adapter
    - 어플리케이션 계층의 인커밍 포트를 호출하는 인커밍 어댑터 (Controller)
    - 어플리케이션 계층의 아웃고잉 포트에 대한 구현을 제공하는 아웃고잉 어댑터 (PersistenceAdapter, Repository)

### 3.2 헥사고날 아키텍처 구조의 장점

장점1. 이러한 패키지 구조는 모델-코드 갭(아키텍처-코드 갭)을 효과적으로 다룰 수 있다.


> 💡 [모델-코드 갭(model-code gap)](https://www.ben-morris.com/most-architecture-diagrams-are-useless/#:~:text=George%20Fairbanks%20identified%20what%20he,always%20be%20mapped%20into%20code.)
> 아키텍처 모델에는 항상 코드에 매핑할 수 없는 추상적인 개념, 기술 선택 및 설계 결정이 혼합되어 있다. 최종 결과는 모델이 정한 구성 요소의 배열과 반드시 일치하지 않는 소스 코드가 될 수 있다.

장점2. 패키지간 접근을 제어할 수 있다.

- package-private인 adapter 클래스
    - 모든 클래스는 application 패키지 내의 포트 인터페이스를 통해 바깥에 호출되기 때문에 adapter는 모두 package-private 접근 수준으로 둬도 된다.
    - 어플리케이션 계층에서 어댑터로 향하는 우발적 의존성은 있을 수 없다.
- public이어야 하는 application, domain의 일부 클래스
    - application의 port(in, out)
        - `SendMoneyUseCase`, `LoadAccountPort`, `UpdateAccountStatePort`
    - 도메인 클래스
        - `Account`, `Activity`
- package-private이어도 되는 서비스 클래스
    - 인커밍 포트 인터페이스 뒤에 숨겨지는 서비스는 public일 필요가 없다.
        - `GetAccountBalanceService`

## 4. 의존성 주입의 역할

- 클린 아키텍처의 본질적인 요건
    
    `어플리케이션이 인커밍/아웃고잉 어댑터에 의존성을 갖지 않아야 한다.`
    
- 의존성 역전 원칙 이용
    - 어플리케이션 계층에 인터페이스(port)를 만들고 어댑터에 해당 인터페이스를 구현한 클래스를 둔다.
    - 모든 계층에 의존성을 가진 중립적인 컴포넌트를 하나 두고, 이 컴포넌트가 아키텍처를 구성하는 대부분의 클래스를 초기화하는 역할을 한다.
    - 웹 컨트롤러가 서비스에 의해 구현된 인커밍 포트를 호출한다. 서비스는 어댑터에 의해 구현된 아웃고잉 포트를 호출한다.
    
    ![image](https://user-images.githubusercontent.com/37948906/154954236-0972c09e-d6c7-41d1-ad79-eedb73822d85.png)

    - AccountController
        - SendMoneyUseCase 인터페이스가 필요하므로 의존성 주입을 통해 SendMoneyService 클래스의 인스턴스를 주입
    - SendMoneyService
        - LoadAccount 인터페이스로 가장한 AccountPersistenceAdapter 클래스의 인스턴스 주입
        
    
    ## 5. 유지보수 가능한 소프트웨어를 만드는 데 어떻게 도움이 될까?
    
    - 코드에서 아키텍처의 특정 요소를 찾으려면 이제 아키텍처 다이어그램의 박스 이름을 따라 패키지 구조를 탐색하면 된다.
    - 이를 통해 의사소통, 개발, 유지보수 모두가 조금 더 수월해진다.