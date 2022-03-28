# 11장. 의식적으로 지름길 사용하기

- [11장. 의식적으로 지름길 사용하기](#11장-의식적으로-지름길-사용하기)
  - [왜 지름길은 깨진 창문과 같을까?](#왜-지름길은-깨진-창문과-같을까)
  - [깨끗한 상태로 시작할 책임](#깨끗한-상태로-시작할-책임)
  - [핵사고날 아키텍처에서의 지름길](#핵사고날-아키텍처에서의-지름길)
    - [Use Case 간에 모델 공유](#use-case-간에-모델-공유)
    - [Input / Output 모델로 도메인 객체 사용하기](#input--output-모델로-도메인-객체-사용하기)
  - [들어오는 포트 생략](#들어오는-포트-생략)
  - [Application Service 생략](#application-service-생략)

먼저 지름길이 무조건 나쁜것은 아니다. 때때로는 일정을 맞추기 위해서 의식적으로 지름길을 택하고 `기술부채` 후에 이를 보정하는 방식을 택할 수 있다. 다만, 지름길이라는 것을 모르고 사용한다면 추후에 큰 문제가 될 수 있다.

## 왜 지름길은 깨진 창문과 같을까?

> Broken Windows란 깨진 유리창 이론에서 깨진 유리창에 해당한다.
깨진 유리창을 방치해두면 그 지점을 중심으로 범죄가 확산된다는 사회 무질서와 관련된 이론이다.
>
> 참고: [https://ko.wikipedia.org/wiki/깨진_유리창_이론](https://ko.wikipedia.org/wiki/%EA%B9%A8%EC%A7%84_%EC%9C%A0%EB%A6%AC%EC%B0%BD_%EC%9D%B4%EB%A1%A0)

프로그래밍에서 지름길을 택하는 것은 깨진 유리창과 유사하다.

- 저품질의 코드베이스에서 작업을 하는 경우 더 저품질의 코드를 작성할 가능성이 높다.
- 코딩 규칙을 많이 위반한 코드베이스에서 작업을 할수록 또 다른 코딩 규칙을 위반할 가능성이 높다.
- 많은 지름길을 사용한 코드베이스에서 작업을 할수록 또 다른 지름길을 사용할 가능성이 높다.

## 깨끗한 상태로 시작할 책임

코드 작업이 깨진 유리창의 차와 같지는 않지만 코드를 작성하는 프로그래머는 무의식적으로 깨진 유리창 심리의 대상이 될 수 있다. 즉, 지름길을 사용한 코드가 프로젝트 내에 있다면 무의식적으로 지름길 방식으로 코드를 작성할 수 있다는 것이다. 이와 같은 이유로 프로젝트를 시작부터 깔끔하게 유지하는 것이 중요하다.

특히 지름길이 많은 코드베이스를 팀의 새로운 개발자가 유지보수한다고 생각해보자. 이 경우는 더더욱 지름길을 사용할 가능성이 높아진다.

이를 위해서 Micheal Nygard가 제안한 것과 같이 Architecture Decision Records `ADRs`와 같은 폼으로 추가되는 지름길을 의식적으로 관리할 수 있어야한다.

Architecture Decision Records: [http://thinkrelevance.com/blog/2011/11/15/documenting-architecture-decisions](http://thinkrelevance.com/blog/2011/11/15/documenting-architecture-decisions)

## 핵사고날 아키텍처에서의 지름길

### Use Case 간에 모델 공유

`4장. Implementing a Use Case`에서 다른 Use Case에서는 다른 input, output 모델을 사용해야한다고 했다. 다른 Use Case에서 같은 input, output 모델을 사용하는 경우에는 두 Use Case 간에 결합도를 증가시킨다.

![Share_Use_Case_Model](https://user-images.githubusercontent.com/30178507/160421107-51d0c66d-a431-46ec-a90e-86b5b4b0f4f1.png)

위 그림에서 SendMoneyUseCase와 RevokeActivityUseCase는 SendMoneyCommand input 모델을 공유하고 있다. 이는 만약 SendMoneyCommand에 변경이 있는 경우에 SendMoneyUseCase와 RevokeActivityUseCase 둘 다에 영향을 미친다. output 모델을 공유하는 경우도 마찬가지이다.

예외적으로 Use Case 간에 input, output 모델을 공유해도 되는 경우가 있을 수 있다. 이는 기능적인 범위가 Use Case 간에 일치하는 경우에는 가능하다. 단, 현재는 범위가 일치하더라도 추후에 분리될 가능성이 있다면 이는 input, output 모델을 공유하는 것은 결국 지름길이 된다.

만약 처음에 동일한 기능으로 잡혀있다면 같은 input, output 모델을 사용하여 개발하고 추후에 분리되야하는 상황에서 분리하여 작업하는 것도 좋다고 생각한다. 무조건적으로 분리하는 것보다도...

> 모든 기능은 정책적으로 얼마든지 변경이 가능하기 때문에 결국은 모든 기능마다 분리해서 작업하는 것이 맞지 않을까 싶다.
>
> 그나마 CRUD의 작업만 하는 어드민 작업에서는 공유해도 무방하지 않을까 싶다.

### Input / Output 모델로 도메인 객체 사용하기

input, output 모델로 도메인 객체를 사용하는 것은 도메인 객체가 변경될 이유를 추가하는 것과 같다. 즉, 비즈니스적인 요구사항의 변화 뿐만 아니라 내부, 외부 응답 모델의 변화로도 도메인 객체를 변화시킨다.

![Command_Domain_Object](https://user-images.githubusercontent.com/30178507/160421103-b1959ca7-8eb1-4ba4-b1f3-b1a98b3aba06.png)

만약 SendMoneyUseCase에서 로직을 처리하는데 Account 객체에서 정의된 필드 외에 다른 필드가 추가적으로 필요하다고 가정한다. 이 경우에는 Account 도메인 객체를 input 모델로 사용하고 있기 때문에 결국 Account의 도메인 로직에는 필요가 없더라도 Account에 필드를 추가해야한다.

## 들어오는 포트 생략

들어오는 포트의 생략은 도메인 로직의 진입점을 명백히 하지 못한다.

![skip_port](https://user-images.githubusercontent.com/30178507/160421097-de2f8a43-1028-4963-b397-8b2f95c00a4a.png)

들어오는 포트를 생략하면 incoming adapter와 application 레이어 사이에 추상화 계층을 하나 생략하는 것과 같다. 이는 도메인 로직의 진입점을 생략하는 것이므로 애플리케이션 내부를 더욱 잘 알아야한다는 문제가 있다.

애플리케이션 유지보수 관점에서 애플리케이션 내부에 대해 잊어먹을수도 있고 팀에 새로운 개발자가 왔을때 코드 파악에 어려움을 줄 수 있으므로 들어오는 포트를 구현하는 것이 바람직하다.

## Application Service 생략

Service 대신에 Adapter를 Use Case의 구현체로 사용하는 경우 다음과 같은 형태가 될 수 있다.

![skip_service](https://user-images.githubusercontent.com/30178507/160421092-c93bbefe-7580-4d9c-a380-c7a33fdea614.png)

간단한 CRUD Use Case에 대해서 위와 같이 구현하는 경향이 있다. 즉, 복잡한 도메인 로직 없이 CRUD만 Persistence Adapter에 요청하는 형식이다.

하지만 이는 incoming adapter와 outgoing adapter에서 모델을 같이 공유해야한다. 이는 input 모델로써 도메인 모델을 사용하게됨을 의미한다.

여기에 더해서 만약 간단한 CRUD에서 더 복잡한 도메인 로직이 추가되는 경우에 persistence adapter에 도메인 로직을 구현하는 형태로 발전할 가능성이 있다. 이는 도메인 로직이 persistence 레이어와 분산이 되므로 향후 애플리케이션의 유지보수를 힘들게 만든다.
