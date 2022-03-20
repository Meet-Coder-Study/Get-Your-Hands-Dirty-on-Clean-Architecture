# 7장. 아키텍처요소 테스트하기 

## The Test Pyramid

![](https://user-images.githubusercontent.com/30178507/158181032-672eabb8-a3fb-4874-96d2-f9dc8751e7ec.png)

> 테스트 피라미드는 2009년 Mike Cohn의 책 `Succeeding with Agile`에서 언급됐다.

테스트가 여러 단위 및 교차 단위 경계 아키텍처 경계 또는 시스템 경계를 결합하면 빌드 비용이 더 많이 들고 실행 속도가 느려지며 취약해지는 경향이 있다. 피라미드는 이러한 테스트가 더 비싸질수록 테스트가 더 많은 범위를 커버하는 것을 목표로 해야함을 알려준다. 그렇지 않으면 새로운 기능 대신 테스트를 구축하는 데 너무 많은 시간을 할애하는 경향이 있다.

기본적으로 테스트 작성 비용이 저렴하고 유지관리가 쉬운 단위 테스트가 프로젝트에서 가장 많은 테스트를 구성한다. 따라서 테스트 피라미드의 가장 기초가 된다. 일반적으로 단일 클래스를 인스턴스화하고 인터페이스를 통해 기능을 테스트한다.

통합 테스트는 테스트 피라미드의 다음 계층을 형성한다. 여러 컴포넌트 테스트를 조합하여 로직을 구성하는 클래스를 인스턴스화하고 엔트리 클래스의 인터페이스를 통해 컴포넌트의 로직을 원하는대로 조합하는지 확인한다.

시스템 테스트는 애플리케이션을 구성하는 전체 구성을 가동하여 애플리케이션의 모든 계층에서 Use Case가 제대로 동작하는지 확인한다.

## 단위 테스트

핵사고날 아키텍처 구성요소 중 단위테스트가 적절한 요소들은 도메인 객체와 유스케이스이다.

### 도메인 객체 테스트

도메인 객체는 가장 만들기 쉽고 이해하기도 쉬우며 빠르게 실행되는 테스트를 만들 수 있다. 다른 클래스에 거의 의존하지 않으므로 단위 테스트 이외에 다른 종류의 테스트는 불필요하다.

### 유스케이스 테스트

도메인 객체 다음의 아키텍처 요소는 유스케이스이다. 유스케이스도 단위 테스트 대상이다. 유스케이스에서는 보통 Out-Port를 사용하여 타 시스템 또는 DB에서 데이터를 가져오고 도메인 객체를 사용하여 로직을 구성한다. 따라서 유스케이스를 단위테스트로 작성하려면 Mock이 필수적이다. Port를 Mocking 하여 특정 메서드들이 잘 호출이 되었는지와 이들 메서드들을 조합하는 유스케이스의 동작을 테스트할 수 있다.

다만, Mock을 사용하게 되면 테스트가 코드의 행동 변경과 리펙터링에 취약해진다. 따라서 중요한 상호작용이 있는 경우에만 테스트를 하거나 아예 유스케이스도 통합테스트로 구성을 하는 것이 맞지 않을까 생각이 든다.

## 통합 테스트

통합테스트가 필요한 요소는 포트의 구현체들인 웹 어뎁터 또는 영속성 어뎁터가 된다.

### 웹 어댑터 테스트

웹 어댑터 테스트는 HTTP를 통해 입력을 받고 입력 유효성 검증, 유스케이스가 사용할 수 있는 포멧으로 변경, 다시 HTTP로 응답이라는 일련의 과정이 테스트 되어야한다.

이를 위해서 Spring Boot에서는 `@WebMvcTest`를 사용할 수 있다. 실제 유스케이스는 Mocking 하므로 내부로직을 모두 태울수는 없지만 일련의 HTTP 통신이 잘 이뤄지고 유스케이스가 정의한 입력 포멧을 제대로 전달하는 지를 테스트할 수 있다.

> Rest Docs 라이브러리를 활용한다면 웹 어댑터 테스트를 통해서 먼저 스팩을 정의하고 API 문서를 작성하는데에도 활용할 수 있어보인다.

### 영속성 어댑터 테스트

영속성 어댑터도 데이터 영속화를 위한 컨텍스트를 띄우는 과정이 필요하므로 통합테스트가 필요하다.

만약 Spring Boot와 Spring Data의 일부 프로젝트에서는 `@DataJpaTest`, `@DataRedisTest`, `@DataMongoTest`등을 사용하여 도메인 객체가 제대로 데이터베이스에 영속화되는지, 조회가 되어 도메인 객체로 잘 변환이 되는지 등을 테스트할 수 있다.

> 만약 Spring Data에서 Test 자동 설정을 지원하지 않는 프로덕트가 있다면 `ex. AWS DynamoDB 등` `@SpringBootTest`를 통해서 테스트할 수 밖에 없을것 같다.

## 시스템 테스트

시스템 테스트는 애플리케이션을 구성하는 전체 컨텍스트를 띄워 API 요청을 통해 잘 동작하는지를 검증한다. 따라서 전체 컨텍스트를 띄워야하므로 `@SpringBootTest`를 사용하여 테스트를 진행한다.

출력 포트도 컨텍스트를 띄워야하기 때문에 필요한 데이터베이스는 in-memory 데이터베이스를 활용하거나 실제 환경과 비슷하게 가져가고 싶다면 Docker 컨테이너를 띄워 테스트용으로 사용할 수 있다.

출력 포트가 타 시스템을 요청하는 경우에는 [MockServer](https://mock-server.com/)를 활용하여 요청 어댑터가 HTTP 통신을 제대로 수행하는지를 검증할수도 있을 것이다.