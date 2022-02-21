# 4장. 유스케이스 구현하기

`육각형 아키텍처 스타일에서 유스케이스 구현하기`

- 어플리케이션, 웹, 영속성 계층이 현재 아키텍처에서 아주 느슨하게 결합
- 육각형 아키텍처는 도메인 중심의 아키텍처에 적합하므로 도메인 엔티티를 만들고 엔티티를 중심으로 유스케이스 구현

## 1. 도메인 모델 구현하기

한 계좌에서 다른 계좌로 송금하는 유스케이스

입금과 출금을 할 수 있는 Account 엔티티와 출금 계좌에서 돈을 출금해서 입금 계좌로 돈을 입금

```java
package buckpal.account.domain;

@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Account {

	@Getter private final AccountId id;

	@Getter private final Money baselineBalance;

	@Getter private final ActivityWindow activityWindow;

	public static Account withoutId(
					Money baselineBalance,
					ActivityWindow activityWindow) {
		return new Account(null, baselineBalance, activityWindow);
	}

	public static Account withId(
					AccountId accountId,
					Money baselineBalance,
					ActivityWindow activityWindow) {
		return new Account(accountId, baselineBalance, activityWindow);
	}

	public Optional<AccountId> getId(){
		return Optional.ofNullable(this.id);
	}

	public Money calculateBalance() {
		return Money.add(
				this.baselineBalance,
				this.activityWindow.calculateBalance(this.id));
	}

	public boolean withdraw(Money money, AccountId targetAccountId) {

		if (!mayWithdraw(money)) {
			return false;
		}

		Activity withdrawal = new Activity(
				this.id,
				this.id,
				targetAccountId,
				LocalDateTime.now(),
				money);
		this.activityWindow.addActivity(withdrawal);
		return true;
	}

	private boolean mayWithdraw(Money money) {
		return Money.add(
				this.calculateBalance(),
				money.negate())
				.isPositiveOrZero();
	}

	public boolean deposit(Money money, AccountId sourceAccountId) {
		Activity deposit = new Activity(
				this.id,
				sourceAccountId,
				this.id,
				LocalDateTime.now(),
				money);
		this.activityWindow.addActivity(deposit);
		return true;
	}

	@Value
	public static class AccountId {
		private Long value;
	}

}
```

- Account 엔티티
    - 실제 계좌의 현재 스냅샷을 제공
- ActivityWindow
    - 한 계좌에 대한 모든 활동(activity)들을 항상 메모리에 한꺼번에 올리는 것은 현명하지 않으므로  지난 며칠 혹은 몇 주간의 범위 활동만 보유
- baselineBalance
    - 첫번째 활동 전의 잔고
    - 현재 총 잔고는 기준 잔고(baseBalance)에 활동 창의 모든 활동들의 잔고를 합한 값
- withdraw (입금), deposit(출금)
    - 새로운 활동을 활동창에 추가
    - 출금 전에는 잔고를 초과하는 금액을 출금할 수 없도록 비즈니스 규칙 검사
    

## 유스케이스 둘러보기

```java
01. 입력을 받는다.
02. 비즈니스 규칙을 검증한다.
03. 모델 상태를 변경한다.
04. 출력을 반환한다.
```

- 입력을 받는다.
    - `01. 입력을 받는다` 에서는 `입력 유효성 검증` 을 하지 않는다. 유스케이스 코드에서는 입력 유효성을 검증하지 않고, 도메인 로직에만 신경써야 함
    - 따라서 입력 유효성 검증은 다른 곳에서 처리함
- 모델 상태를 변경한다.
    - 도메인 객체의 상태를 바꾸고 영속성 어댑터를 통해 구현된 포트로 이 상태를 전달해서 저장
- 출력을 반환한다
    - 아웃 고잉 어댑터에서 온 출력값을 유스케이스를 호출한 어댑터로 반환할 출력 객체로 변환

```java
package buckpal.account.application.service;

@RequiredArgsConstructor
@Transactional
public class SendMoneyService implements SendMoneyUseCase {

	private final LoadAccountPort loadAccountPort;
	private final AccountLock accountLock;
	private final UpdateAccountStatePort updateAccountStatePort;

	@Override
	public boolean sendMoney(SendMoneyCommand command) {
		// TODO: 비즈니스 규칙 검증
		// TODO: 모델 상태 조작
		// TODO: 출력 값 반환
	}
}
```

- 서비스
    - 인커밍 포트 인터페이스인 SendMoneyUseCase 구현 (돈 송금하기)
    - 아웃고잉 포트 인터페이스인 LoadAccountPort 호출 (계좌 불러오기)
    - 데이터베이스 상태 업데이트를 위해 UpdateAccountStatePort 호출

![image](https://user-images.githubusercontent.com/37948906/154954560-c5a6b8d6-3ea2-43c0-a895-eda1fbe537a7.png)

## 입력 유효성 검증

- 어댑터에서 유스케이스에 입력을 전달하기 전에 입력 유효성을 전달한다면? 모든 어댑터에서 유효성 검증을 구현해야 함.
- 따라서 어플리케이션 계층에서 유효성 검증을 해야 함. but 유스케이스 클래스에서 입력 유효성을 검증하지는 않음

### 생성자 내에서의 유효성 검증

- 생성자 내에서 유효성 검증을 한다면? (인커밍 포트 패키지에 위치한 SendMoneyCommand)

```java
@Getter
public class SendMoneyCommand {

    private final AccountId sourceAccountId;
    private final AccountId targetAccountId;
    private final Money money;

    public SendMoneyCommand(
            AccountId sourceAccountId,
            AccountId targetAccountId,
            Money money) {
        this.sourceAccountId = sourceAccountId;
        this.targetAccountId = targetAccountId;
        this.money = money;
        requireNonNull(sourceAccountId);
        requireNonNull(targetAccountId);
        requireNonNull(money);
        requireGraterThan(money, 0);
    }
}
```

자바에 Bean Validation API를 활용하면 필드 애너테이션으로 간단하게 사용할 수 있다.

```java
@Value
@EqualsAndHashCode(callSuper = false)
public
class SendMoneyCommand extends SelfValidating<SendMoneyCommand> {

    @NotNull
    private final AccountId sourceAccountId;

    @NotNull
    private final AccountId targetAccountId;

    @NotNull
    private final Money money;

    public SendMoneyCommand(
            AccountId sourceAccountId,
            AccountId targetAccountId,
            Money money) {
        this.sourceAccountId = sourceAccountId;
        this.targetAccountId = targetAccountId;
        this.money = money;
        this.validateSelf();
    }
}
```

```java
package buckpal.common;

public abstract class SelfValidating<T> {

  private Validator validator;

  public SelfValidating() {
    ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
    validator = factory.getValidator();
  }

  protected void validateSelf() {
    Set<ConstraintViolation<T>> violations = validator.validate((T) this);
    if (!violations.isEmpty()) {
      throw new ConstraintViolationException(violations);
    }
  }
}
```

### 생성자에서 유효성 검증을 했을 때의 장점

- 유효하지 않은 상태의 객체를 만드는 것이 불가능하다
    - 클래스가 불변이기 때문에 생성자의 인자 리스트에 클래스의 각 속성에 해당하는 파라미터를 포함
    - 생성자가 파라미터의 유효성 검증을 하고 있기 때문에 유효하지 않은 객체를 만드는 것은 불가능함

### 만약 빌더를 사용한다면?

```java
new SendMoneyCommandBuilder()
		.sourceAccountId(new AccountId(41L))
		.targetAccountId(new AccountId(42L))
		// ... 다른 여러 필드를 초기화
		.build();
```

유효성 검증 로직은 생성자에 그대로 있기 대문에 빌더가 유효하지 않은 상태의 객체를 생성하지 못하도록 막을 수 있다.

그런데 만약 SendMoneyCommandBuilder에 새로운 필드를 추가해야 한다면?

빌더를 호출하는 코드에 새로운 필드를 추가하는 것을 잊고 만다면, 컴파일러에서는 유효하지 않은 불변 객체를 만들려는 시도에 대해 경고해주지는 못하지만 **단위테스트 혹은 런타임에서 유효성 검증 로직이 동작해서 누락된 파라미터에 대한 에러를 던질 것이다.**

**But 빌더 말고 생성자를 직접 사용했다면 컴파일 에러에 따라 나머지 코드에 즉각 변경사항을 반영**할 수 있었을 것이다. (요즘은 IDE에서 파라미터명 힌트도 주기 때문에 생성자에 많은 인자가 들어가도 어떤 필드가 누락되었는지 잘 알 수 있다.)


<img src="https://user-images.githubusercontent.com/37948906/154954600-c713846a-570e-4cc5-a754-8d16e83c3495.png" width="500">

## 유스케이스마다 다른 입력 모델

- `계좌 등록하기`와 `계좌 정보 업데이트하기` 두 기능의 입력 모델에 ID의 유무 차이가 있다면 유효성 검사는 어떻게 해야할까?
- 각 유스케이스 전용 입력 모델은 유스케이스를 훨씬 정확하게 만들고 다른 유스케이스와의 결합도를 제거해서 불필요한 부수효과가 발생하지 않게 한다.

## 비즈니스 규칙 검증하기

- 비즈니스 규칙 검증
    - 도메인 모델의 현재 상태에 접근한다.
    - 유스케이스의 맥락 속에서 의미적인(sementical) 유효성을 검증
    - ex) 출금 계좌는 초과 출금되어서는 안된다
- 입력 유효성 검증
    - 구문상의(syntactical) 유효성을 검증
    - ex) 송금되는 금액은 0보다 커야 한다.

```java
public class Account {
		// ...

		public boolean withdraw(Money money, AccountId targetAccountId) {
				if (!mayWithdraw(money)){
						return false;
				}
				// ...
		}
}
```

만약 도메인 엔티티에서 비즈니스 규칙을 검증하기가 여의치 않다면 유스케이스 코드에서 도메인 엔티티를 사용하기 전에 해도 된다.

```java
@RequiredArgConstructor
@Transactional
public class SendMoneyService implements SendMoneyUseCase {
		// ...
		
		@Override
		public boolean sendMoney(SendMoneyCommand command){
					requireAccountExists(command.getSourceAccountId());
					requireAccountExists(command.getTargetAccountId());
		}
}
```

유효성 검증하는 코드를 호출하고, 유효성 검증이 실패할 경우 유효성 검증 전용 예외를 던진다.

## 풍부한 도메인 모델 vs 빈약한 도메인 모델

- 풍부한 도메인 모델(rich domain model)
    - 엔티티에서 가능한 많은 도메인 로직이 구현
    - 엔티티들은 상태를 변경하는 메서드를 제공하고, 비즈니스 규칙에 맞는 유효한 변경만을 허용
    - 많은 비즈니스 규칙이 유스케이스 구현체 대신 엔티티에 위치
- 빈약한 도메인 모델(anemic domain model)
    - 엔티티에 상태를 표현하는 필드와 이 값을 읽고 바꾸기 위한 getter, setter 메서드만 포함하고 어떤 도메인 로직도 가지고 있지 않음
    - 도메인 로직은 유스케이스 클래스에만 구현
    - 비즈니스 규칙을 검증하고, 엔티티의 상태를 바꾸고, 데이터 베이스 저장을 담당하는 아웃고잉 포트에 엔티티를 전달할 책임 역시 유스케이스 클래스에 있음

## 유스케이스마다 다른 출력 모델

- 입력과 비슷하게 출력도 가능하면 각 유스케이스에 맞게 구체적일수록 좋다
- 출력은 호출자에게 꼭 필요한 데이터만 들고 있어야 한다
- 유스케이스들 간에 같은 출력 모델을 공유하면 유스케이스들도 강하게 결합됨
- 같은 이유로 도메인 엔티티를 출력 모델로 사용하고 싶은 유혹도 견뎌야 한다.

## 읽기 전용 유스케이스

- 읽기전용 유스케이스를 구현하는 방법

```java
package buckpal.application.service;

@RequiredArgsConstructor
class GetAccountBalanceService implements GetAccountBalanceQuery {
	private final LoadAccountPort loadAccountPort;

	@Override
	public Money getAccountBalance(AccountId, accountId){
		return loadAccountPort.loadAccount(accoutId, LocalDateTime.now()).calculateBalance();
	}
}
```

- 쿼리 서비스 구현
    - GetAccountBalanceQuery 인커밍 포트 구현
    - 데이터베이스로부터 실제 데이터를 로드하기 위해 LoadAccountPort라는 아웃고잉 포트를 호출
- 읽기 전용 쿼리는 쓰기가 가능한 유스케이스(또는 ‘커멘드’)와 코드 상에서 명확하게 구분
    
    ⇒ SQS(Command-Query Separation)
    ⇒ CQRS(Command-Query Responsibility Segregation)

- GetAccountBalanceService 서비스에서는 아웃고잉 포트로 쿼리를 전달하는 것 외에 다른 일을 하지 않는다. 여러 계층에 걸쳐 같은 모델을 사용하면 지름길을 써서 클라이언트가 아웃고잉 포트를 직접 호출하게 할 수도 있음

> CQRS란? 흔히 말하는 CRUD(Create, Read, Update, Delete)에서 CUD(Command)와 R(Query)를 구분하자는 것.
> **CQRS의 장점**
> 1. Read와 CUD 각각에 더 최적화된 Database 구성을 통해서 성능을 더 향상시킬 수 있다.
> 2. Read와 CUD에서 필요한 데이터 형식이 다를 수 있고, 특히 Read는 aggregation(집계 함수) 등의 부가적인 attribute들이 Entity에 필요하게 될 수 있다. R과 CUD를 분리함으로써 R로 인해 Entity의 구조가 변경되는 것을 막을 수 있다.
> 3. R과 CUD를 분리함으로써 과도하게 복잡한 모델을 덜 복합하게 만듦으로서 시스템 복잡도를 줄일 수 있다.
> 출처: [Log.bluayer](https://bluayer.com/37)
    

## 유지보수 가능한 소프트웨어를 만드는데 어떻게 도움이 될까?

- 도메인 로직을 우리가 원하는 대로 구현할 수 있도록 허용하지만, 입출력 모델을 독립적으로 모델링한다면 원치 않는 부수효과를 피할 수 있다.
- 유스케이스 간에 모델을 공유하는 것보다 별도 모델을 만들고, 엔티티를 매핑하는 등의 추가 작업을 해주어야 한다.
- 유스케이스별로 모델을 만들면 유스케이스를 명확하게 이해할 수 있고 장기적으로 유지보수하기도 더 쉽다
- 꼼꼼한 입력 유효성 검증, 유스케이스 별 입출력 모델은 지속가능한 코드를 만드는데 큰 도움이 된다.