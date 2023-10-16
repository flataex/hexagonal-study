# 유스케이스
1. 입력을 받는다.
2. 비즈니스 규칙을 검증한다.
3. 모델 상태를 조작한다.
4. 출력을 반환한다.

- 유스케이스는 인커밍 어댑터로부터 입력을 받는다.
- 유스케이스는 비즈니스 규칙을 검증할 책임이 있다.
- 비즈니스 규칙을 충족하면 유스케이스는 입력을 기반으로 어떤 방법으로든 모델의 상태 를 변경한다. 일반적으로 도메인 객체의 상태를 바꾸고 영속성 어댑터를 통해 구현된 포트로 이 상태를 전달해서 저장될 수 있게 한다.
- 유스케이스는 또 다른 아웃고잉 어댑터를 호출할 수도 있다.
- 아웃고잉 어댑터에서 온 출력값을, 유스케이스를 호출한 어댑터로 반환할 출력 객체로 변환한다.

```java
@RequiredArgsConstructor  
@Transactional  
public class SendMoneyService implements SendMoneyUseCase {
	private final LoadAccountPort LoadAccountPort;  
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

> 서비스는 인커밍 포트 인터페이스인 SendMoneyUseCase를 구현하고, 계좌를 불러오기 위해 아웃고잉 포트 인터페이스인 LoadAccountPort를 호출한다.
> 그리고 데이터베이스의 계좌 상태를 업데이트하기 위해 UpdateAccountStatePort를 호출한다.

### 입력 유효성 검증
호출하는 어댑터가 유스케이스에 입력을 전달하기 전에 입력 유효성을 검증하면 어떨까?
과연 유스케이스에서 필요로 하는 것을 호출자(caller)가 모두 검증했다고 믿을 수 있을까?
또, 유스케이스는 하나 이상의 어댑터에서 호출될 텐데, 그러면 유효성 검증을 각 어댑터에서 전부 구현해야 한다.

애플리케이션 계층에서 입력 유효성을 검증해야 하는 이유는, 그렇게 하지 않을 경우 애플리케이션 코어의 바깥쪽으로부터 유효하지 않은 입력값을 받게 되고, 모델의 상태를 해칠 수 있기 때문이다.

**유스케이스 클래스가 아니라면 도대체 어디에서 입력 유효성을 검증해야 할까?**

입력 모델(input model)이 이 문제를 다루도록 해보자.
‘송금하기’ 유스케이스에서 입력 모델은 예제 코드에서 본 SendMoneyCommand 클래스다.
더 정확히 말하자면 생성자 내에 서 입력 유효성을 검증할 것이다.

```java
@Getter  
public class SendMoneyCommand {
	@NotNull
	private final AccountId sourceAccountld; 
	@NotNull
	private final AccountId targetAccountld; 
	@NotNull
	private final Money;

	public SendMoneyCommand(
		Accountld sourceAccountld,
		Accountld targetAccountld,
		 Money money
	) {
		this.sourceAccountld = sourceAccountld; 
		this.targetAccountld = targetAccountld; 
		this.money = money;
		requireGreaterThan(money, 0);
	}
}
```

### 유스케이스마다 다른 입력 모델
‘계좌 등록하기’와 ‘계좌 정보 업데이트하기’라는 두 가지 유스케이스를 보자. 둘 모두 거의 똑 같은 계좌 상세 정보가 필요하다.

차이점은 ‘계좌 정보 업데이트하기’ 유스케이스는 업데이트할 계좌를 특정하기 위해 계 좌 ID 정보를 필요로 하고, ‘계좌 등록하기’ 유스케이스는 계좌를 귀속시킬 소유자의 ID 정보를 필요로 한다는 것이다.
그래서 두 유스케이스에서 같은 입력 모델을 공유할 경우 ‘계좌 정보 업데이트하기’에서는 소유자 ID에, ‘계좌 등록하기’에서는 계좌 id에 null 값 을 허용해야 한다.

불변 커맨드 객체의 필드에 대해서 null을 유효한 상태로 받아들이는 것은 그 자체로 코드 냄새이다.
하지만 더 문제가 되는 부분은 이제 입력 유효성을 어떻게 검증하느냐다.
등록 유스케이스와 업데이트 유스케이스는 서로 다른 유효성 검증 로직이 필요하다.

각 유스케이스 전용 입력 모델은 유스케이스를 훨씬 명확하게 만들고 다른 유스케이스와 의 결합도 제거해서 불필요한 부수효과가 발생하지 않게 한다.

### 비즈니스 규칙 검증하기
“출금 계좌는 초과 출금되어서는 안 된다”라는 규칙을 보자. 정의에 따르면 이 규칙은 출 금 계좌와 입금 계좌가 존재하는지 확인하기 위해 모델의 현재 상태에 접근해야 하기 때 문에 비즈니스 규칙이다.

반대로 “송금되는 금액은 0보다 커야 한다”라는 규칙은 모델에 접근하지 않= 검증될 수 있다. 그러므로 입력 유효성 검증으로 구현할 수 있다.

그러면 비즈니스 규칙 검증은 어떻게 구현할까?
가장 좋은 방법은 앞에서 “출금 계좌는 초과 인출되어서는 안 된다” 규칙에서처럼 비즈니 스 규칙을 도메인 엔티티 안에 넣는 것이다.

```java
public class Account {

	public boolean withdraw(Money, Accountld targetAccountld) { 
		if (!mayWithdraw(money)) {
			return false;
		}
		// ...
}
```

이렇게 하면 이 규칙을 지켜야 하는 비즈니스 로직 바로 옆에 규칙이 위치하기 때문에 위 치를 정하는 것도 쉽고 추론하기도 쉽다.

만약 도메인 엔티티에서 비즈니스 규칙을 검증하기가 여의치 않다면 유스케이스 코드에 서 도메인 엔티티를 사용하기 전에 해도 된다.

```java
@RequiredArgsConstructor  
@Transactional  
public class SendMoneyService implements SendMoneyUseCase {

	@Override  
	public boolean sendMoney(SendMoneyCommand command) {
		requireAccountExists(command.getSourceAccountId());
		requireAccountExists(command.getTargetAccoiintId());
		// ...
	}
}
```

### 읽기 전용 유스케이스는 어떨까?
이에 계좌의 잔액을 표시해야 한다고 가정해보자. 이를 위한 새로운 유스케이스를 구현 해야할까?

애플리케이션 코어의 관점에서 이 작업은 간단한 데이터 쿼리다.
그렇기 때문에 프로젝트 맥락에서 유스케이스로 간주되지 않는다면 실제 유스케이스와 구분하기 위해 쿼리로 구현할 수 있다.

이 책의 아키텍처 스타일에서 이를 구현하는 한가지 방법은 쿼리를 위한 인커밍 전용포트를 만들고 이를 '쿼리 서비스'에 구현하는 것이다.

```java
@RequiredArgsConstructor  
class GetAccountBalanceService implements GetAccountBalanceQuery {
	private final LoadAccountPort loadAccountPort;

	@Override  
	public Money getAccountBalance(AccountId accountld) {
		return loadAccountPort.loadAccount(accountld, LocalDateTime.now())
			.calculateBalance();
	}
}
```

쿼리 서비스는 유스케이스 서비스와 동일한 방식으로 동작한다. GetAccountBalanceQuery 라는 인커밍 포트를 구현하고, 데이터베이스로부터 실제로 데이터를 로드하기 위해 LoadAccountPort라는 아웃고잉 포트를 호출한다.