# 3. 코드 구성하기

## 계층으로 구성하기

```jsx
buckapl
|--- domain
|    |----- Account
|    |----- Activity
|    |----- AccountRepository
|    |----- AccountService
|--- persistence
|    |----- AccountRepositoryImpl
|--- web
|    |----- AccountController
```

### 문제점

1. 계층으로 코드를 구성하면 기능적인 측면들이 섞이기 쉽다.
2. 애플리케이션의 기능 조각(functional slice)이나 특성(feature)을 구분 짓는 패키지 경계가 업다.
    - 서로 연관되지 않은 기능들끼리 예상하지 못한 부수효과를 일으킬 수 있는 클래스들의 엉망진창 묶음으로 변모할 가능성이 크다.
3. 애플리케이션이 어떤 유스케이스들을 제공하는지 파악할 수 없다.
4. 패키지 구조를 통해서는 우리가 목표로 하는 아키텍처를 파악할 수 없다.

## 기능으로 구성하기

```jsx
buckapl
|--- account
|    |----- Account
|    |----- Activity
|    |----- AccountRepository
|    |----- SendMoneyService
|    |----- AccountRepositoryImpl
|    |----- AccountController
```

### 장점

1. 패키지 경계를 package-private 접근 수준과 결합하면 각 기능 사이의 불필요한 의존성을 방지할 수 있다.

### 문제점

1. 기능을 기준으로 코드를 구성하면 기반 아키텍처가 명확하게 보이지 않는다.

## 아키텍처적으로 표현력 있는 패키지 구조

```jsx
buckapl
|--- account
|    |----- adapter
|    |      |----- in
|    |      |      |---- web
|    |      |      |     |---- AccountController
|    |      |----- out
|    |      |      |---- persistence
|    |      |      |     |---- AccountPersistenceAdapter
|    |      |      |     |---- SpringDataAccountRepository
|    |---- domain
|    |     |----- Account
|    |     |----- Activity
|    |---- application
|    |     |----- SendMoneyService
|    |     |----- port
|    |     |     |---- in
|    |     |     |     |---- SendMoneyUseCase
|    |     |     |---- out
|    |     |     |     |---- LoadAccountPort
|    |     |     |     |---- UpdateAccountStatePort
```

### 장점

1. 각 아키텍처 요소들에 정해진 위치가 있어 직관적이다.
2. 어댑터 코드를 자체 패키지로 이동시키면 필요할 경우 하나의 어댑터를 다른 구현으로 쉽게 교체할 수 있다.
3. DDD 개념을 직접적으로 대응할 수 있다.
    - 상위 레벨 패키지는 다른 바운디드 컨텍스트와 통신할 전용 진입점과 출구(포트)를 포함하는 바운디드 컨텍스트에 해당한다.

### 의문점

- 패키지가 많다는 것은 모든 것을 public으로 만들어 패키지 간의 접근을 허용해야 하는거 아닌가?
    - application 패키지 내에 있는 포트 인터페이스를 통하지 않고서 바깥으로 호출되지 않기 때문에 package-private 접근 수준으로 둬도 된다.
    

## 의존성 주입의 역할

어댑터는 그저 애플리케이션 계층에 위치한 서비스를 호출할 뿐이다. 영속성 어댑터와 같이 아웃고잉 어댑터에 대해서는 제어 흐름의 반대 방향으로 의존성을 돌리기 위해 의존성 역전 원칙을 이용해야 한다. 모든 계층에 의존성을 가진 중립적인 컴포넌트를 하나 도입하면 의존성 주입을 활용할 수 있다. 중립적인 컴포넌트는 아키텍처를 구성하는 대부분의 클래스를 초기화하는 역할을 한다.

### 예시

![image](https://user-images.githubusercontent.com/53366407/151645788-d03b79c4-72cc-43cb-ae06-11ef2c5f54f8.png)

웹 컨트롤러가 서비스에 의해 구현된 인커밍 포트를 호출한다. 서비스는 어댑터에 의해 구현된 아웃고잉 포트를 호출한다.

## 출처
- [만들면서 배우는 클린 아키텍처 - 자바 코드로 구현하는 클린 웹 애플리케이션](https://www.aladin.co.kr/shop/wproduct.aspx?ItemId=283437942)
