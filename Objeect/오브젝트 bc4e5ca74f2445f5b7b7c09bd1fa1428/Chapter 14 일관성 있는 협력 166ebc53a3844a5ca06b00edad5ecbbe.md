# Chapter 14 일관성 있는 협력

애플리케이션을 개발하다보면 유사한 요구사항을 반복적으로 추가하거나 수정하게 되는 경우가 있다.

이런 객체의 협력구조가 서로 다른 경우에는 코드를 이해하기 어렵고 코드 수정으로 인해 버그가 발생할 위험도 높아진다. 그리고 유사한 요구사항을 계속 추가하다 보면 일관성은 떨어지게 되어있다.

이럴때 일관성을 높이려면 협력패턴을 사용하는 것이다.

유사한 패턴을 따른다면 시스템을 이해하고 확장하기 위해 요구되는 정신적인 부담을 크게 줄일수 있다.

협력패턴은 적용한 코드는 아래에서 확인할 수 있다.

## 1. 핸드폰 과금 시스템 변경하기

| 유형 | 형식 | 예 |
| --- | --- | --- |
| 고정요금 방식 | A초당 B원 | 10초당 18원 |
| 시간대별 방식 | A시부터 B시까지 C초당 D원
B시부터 C시까지 C초당 E원 | 00시부터 19시까지 10초당 18원
19시부터 24시까지 10초당 15원 |
| 요일별 방식 | 평일에는 A초당 B원
공휴일에는 A초당 C원 | 평일에는 10초당 38원
공휴일에는 10초당 19원 |
| 구간별 방식 | 초기 A분 동안 B초당 C원
A분 ~ D분까지 B초당 D원
D분 초과시 B초당 E원 | 초기 1분 동안 10초당 50원
초기 1분 이후 10초당 20원 |

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled.png)

새로운 기본정책을 적용해 가능한 모든 경우의 수를 나타낸 것이다.

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%201.png)

구현해볼 클래스 구조도이다.

1) 고정 요금 방식 구현

```java
public class FixedFeePolicy extends BasicRatePolicy{
	private Money amount;
	private Duration seconds;

	public FixedFeePolicy(Money amount, Duration seconds){
		this.amount = amount;
		this.seconds = seconds;
	}

	@Override
	protected Money calculateCallFee(Call call){
		return amount.times(call.getDuration().getSeconds() / seconds.getSeconds());
	}
}
```

2) 시간대별 방식 구현하기

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%202.png)

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%203.png)

11 장에 구현한 시간대별 방식을 적용하면 버그가 생기게 된다. 

```java
public class DateTimeInterval{
	private LocalDateTime from;
	private LocalDateTime to;

	public static DateTimeInterval of(LocalDateTime from, LocalDateTime to){
		return new DateTimeInterval(from, to);
	}

	public static DateTimeInterval toMidnight(LocalDateTime from){
		return new DateTimeInterval(
			from,
			LocalDateTime.of(from.toLocalDate(), LocalTime.of(23, 59, 59, 999_999_999)));
	}

	public static DateTimeInterval fromMidnight(LocalDateTime to){
		return new DateTimeInterval(
			LocalDateTime.of(to.toLocalDate(), LocalTime.of(0, 0)),
			to);
		)
	}

	public static DateTimeInterval during(LocalDate date){
		return new DateTimeInterval(
			LocalDateTime.of(date, LocalTime.of(0, 0)),
			LocalDateTime.of(date, LocalTIme.of(23, 59, 59, 999_999_999)));
		)
	}

	private DateTimeInterval(LocalDateTime from, LocalDateTime to){
		this.from = from;
		this.to = to;
	}

	public Duration duration(){
		return Duration.between(from, to);
	}

	public LocalDateTime getFrom(){
		return from;
	}

	public LocalDateTime getTo(){
		return to;
	}
}
```

```java
public class Call{
	// private LocalDateTime from;
	// private LocalDateTime to;
  // ->
	private DateTimeInterval interval;

	public Call(LocalDateTime from, LocalDateTime to){
		this.interval = DateTimeInterval.of(from, to);
	}

	public Duration getDuration(){
		return interval.duration();
	}

	public LocalDateTime getFrom(){
		return interval.getFrom();
	}

	public LocalDateTime getTo(){
		return interval.getTo();
	}

	public DateTimeInterval getInterval(){
		return interval;
	}
}
```

두 단계로 나눠 구현할 필요가 있다.

- 통화 기간을 일자별로 분리한다.
- 일자별로 분리된 기간을 다시 시간대별 규칙에 따라 분리한 후 각 기간에 대해 요금을 계산한다.

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%204.png)

예) 19시부터 24시까지 10초당 15원의 요금을 부과

1월 1일 10시부터 1월 3일 15시까지 3일에 걸쳐 통화

intervals 변수에게 splitByDay 메서드를 호출해 날짜별로 통화시간을 분리한다.

1월 1일 10시 ~ 24시, 1월 2일 0시 ~ 24시, 1월 3일 0시 ~ 15시로 분리한다.

1월 1일 10시 ~ 24시의 경우 19시 기준으로 나눠 1월 1일 10시 ~ 1월 1일 19시, 1월 1일 19시 ~ 24시로 나눈다.

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%205.png)

나눈 결과이다.

```java
public class TimeOfDayDiscountPolicy extends BasicRatePolicy{
	private List<LocalTime> starts = new ArrayList<>();
	private List<LocalTime> ends = new ArrayList<>();
	private List<Duration> durations = new ArrayList<>();
	private List<Money> amount = new ArrayList<>();

	@override
	protected Money calculateCallFee(Call call){
		Money result = Money.ZERO;
		for(DateTimeInterval interval. call.splitByDay()){
			for(int loop = 0; loop < starts.size(); loop++){
				result.plus(amounts.get(loop).times(
					Duration.between(from(interval, starts.get(loop)), to(interval, ends,get(loop)))
						.getSeconds() / durations.get(loop).getSeconds()));
			}
		}
		return result;
	}

	private LocalTime from(DateTimeInterval interval, LocalTime from){
		return interval.getFrom().toLocalTime().isBefore(from) ?
			from : interval getFrom().toLocalTime();
	}

	private LocalTime to(DateTimeInterval interval, LocalTime to){
		return interval.getTo().toLocalTime().isAfter(to) ?
				to: interval.getTo().toLocalTime();
	}
}
```

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%206.png)

시간대별로 요금방식을 구성하는 4개의 리스트이다.

```java
public class Call{
	public List<DateTimeInterval> splitByDay(){
		return interval.splitByDay();
	}
}
```

```java
public class DateTimeInterval{
	public List<DateTimeInterval> splitByDay(){
		if(days() > 0){
			return splitByDay(days());
		}
		return Arrays.asList(this);
	}

	private long days(){
		return Duration.between(from.toLocalDate().asStartOfDay(), to.toLocalDate().atStartOfDay())
			.toDays();
	}

	private List<DateTimeInterval> splitByDay(long days){
		List<DateTimeInterval> result = new ArrayList<>();
		addFirstDay(result);
		addMiddleDays(result, days);
		addLastDay(result);
		return result;
	}

	private void addFirstDay(List<DateTimeInterval> result){
		result.add(DateTimeInterval.toMidnight(from));
	}

	pirvate addMiddleDays(List<DateTimeInterval> result, long days){
		for(int loop = 1; loop < days; loop++){
			result.add(DateTimeInterval.during(from.toLocalDate().plusDays(loop)));
		}
	}

	private void addLastDay(List<DateTimeInterval> result){
		result.add(DateTimeInterval.fromMidnight(to));
	} 
}
```

1. 요일별 방식 구현하기

```java
public class DayOfWeekDiscountRule{
	private List<DayOfWeek> dayOfWeeks = new ArrayList<>();
	private Duration duration = Duration.ZERO;
	private Money amount = Money.ZERO;

	pulbic DayOfWeekDiscountRule(List<DayOfWeek> dayOfWeeks, Duration duration, Money amount){
		this.dayOfWeeks = dayOfWeeks;
		this.duration = duration;
		this.amount = amount;
	}

	public Money calculate(DateTimeInterval interval){
		// 요일 조건을 만족시킬 경우
		if(dayOfWeeks.contains(interval.getFrom,.getDayOfWeek())){
			return amount.times(interval.duration().getSeconds() / duration.getSecond());
		}
		return Money.ZERO;
	}
}
```

```java

public class DayOfWeekDiscountPolicy extens BasicRatePolicy{
	private List<DayOfWeekDiscountRule> rules = new ArrayList<>();
		
	public DayOfWeekDiscountPolicy(List<DayOfWeekDiscountRule> rules){
		this.rules = rules;
	}

	@override
	protected Money calculateCallFee(Call call){
		Money result = Money.ZERO;
		for(DateTimeInterval interval : call.getInterval().splitByDay()){
			for(DayOfWeekDiscountRule rule: rules){
				result.plus(rule.calculate(interval));
			}
		}
		return result;
	}
}
```

1. 구간별 방식 구현하기

```java
public class DurationDiscountRule extends FixedFeePolicy{
	private Duration from;
	private Duration to;

	public DurationDiscountRule(Duration from, Duration to, Money amount, Duration seconds){
		super(amount, seconds);
		this.from = from;
		this.to = to;
	}

	public Money calculate(Call call){
		if(call.getDuration().compareTo(to) > 0){
			return Money.ZERO;
		}
		if(call.getDuration().compareTo(from) < 0){
			return Money.ZERO;
		}

		// 부모 클래스 Phone의 클래스를 파라미터로 받는다
		Phone phone = new Phone(null);

		phone.call(new Call(call.getFrom().plus(from),
				call.getDuration().compareTo(to) > 0 ? call.getFrom().plus(to) : 
				call.getTo()));

		return super.calculateFee(phone);
	}
}
```

이제 여러개의 룰을 이용해 할인 정책을 구현할 수 있다.

```java
public class DurationDiscountPolicy extends BasicRatePolicy{
	private List<DurationDiscountRule> rules = new ArrayList<>();

	public DurationDiscountPolicy(List<DurationDiscountRule> rules){
		this,rules = rules;
	}

	@overried
	protected Money calculateCallFee(Call call){
		Money resukt = Money.ZERO;
		for(DurationDiscountRule rule: rules){
			result.plus(rule.calculate(call));
		}
		return result;
	}
}
```

이 클래스는 할일 요금 정책을 정상적으로 계산하고 클래스 하나당 하나의 책임을 가지게 한다.

하지만 이 설계가 훌륭하다 할 수 없다. 이유는 기존 설계가 어떠한 가이드도 제공을 하지 않아 기존 클래스와 일관성이 떨어지 때문이다.

> 코드 재사용을 위한 상속은 해롭다.
DurationDiscountRule 클래스가 상속을 잘못 사용한 경우라는 사실을 눈치챘을 것이다.
이 코드는 이해하기도 어려운데 FixedFeePolicy의 calculateFee 메서드를 재사용하기 위해
DurationDiscountRule의 calculate 메서드 안에 Phone과 Call의 인스턴스를 생성하는것은 꽤나 부자연스러워 보인다. 이것은 상속을 위해 설계된 클래스가 아닌 재사용을 위해 억지로 코드를 비튼 결과이다.
> 

## 2.  설계에 일관성 부여하기

<aside>
🔖 기본 지침
1. 변하는 개념을 변하지 않는 개념으로부터 분리하라.
  각 조건문을 개별적인 객체로 분리했고 이 객체들과 일관성있게 협력하기 위해 타입계층을 구성했  다. 그리고 이 타입 계층의 클라이언트로부터 분리하기 위해 역할을 도입하고 최종적으로 이 역할을 추상클래스와 인터페이스로 구현했다.

2. 변하는 개념을 캡슐화 해라
  Movie로 부터 할인 정책을 분리한 후 추상 클래스인 DiscountPolicy를 부모로 삼아 상속 계층을 구성한 이유가 바로 Movie로 부터 구체적인 할인 정책들을 캡슐화 하기 위해서다.

</aside>

조건 로직 대 객체 탐색

```java
public class ReservationAgency{
	public Reservation reserve(Screening screening, Customer customer, int audienceCount){
		for(DiscountCondition conition : movie.getDiscountConditions()){
			if(condition.getType() == DiscountConditionType.PERIOD){
				// 기간 조건인 경우
			}else{
				// 회차 조건인 경우
			}
		}

		if(discountable){
			switch(movie.getMovieType()){
				case AMOUNT_DISCOUNT : 
					// 금액 할인 정책인 경우
				case PERCENT_DISCOUNT : 
					// 비율 할인 정책인 경우
				case NONE_DISCOUNT :
					// 할인 정책이 없는 경우
			}
		}else{
			// 할인 적용이 불가능한 경우
		}
	}
}
```

위 코드는 추가 사항이 있다면 새로운 else절이나 case를 추가 할것이다.

객체지향은 조금 다른 접근방법을 취한다

```java
public class Movie{
	private DiscountPolicy discountPolicy;

	public Money calculateMovieFee(Screening screening){
		return fee.minus(discountPolicy.calculateDiscountAmount(screening));
	}
}
```

다형성은 바로 이런 조건 로직을 객체 사이의 이동으로 바꾸기 위해 객체지향이 제공하는 설계 기법이다.

```java
public abstract class DiscountPolicy{
	private List<DiscountCondition> conditions = new ArrayList<>();

	public Money calculateDIscountAmount(Screening screening){
		for(DiscoountCondition each: conditions){
			if(each.isSatisfiedBy(screening)){
				return getDiscountAmount(screening);
			}
		}
		return screening.getMovieFee();
	}
}
```

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%207.png)

하지만 협력에 참여하는 주체는 구체적인 객체이다. 이 객체들은 협력 안에서 DiscountPlicy와 DiscountCondition을 대체할 수 있어야 한다.

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%208.png)

인터페이스를 이용해 이 타입 계층을 구현한 것이다.

객체 사이의 이동으로 대체하기 위해서는 커다란 클래스를 더 작은 클래스들로 분리해야 한다. 그렇게 하기 위해 어떤 기준을 따르는게 좋을까? 가장 중요한 것은 변경의 이유와 주기이다. 클래스는 명확히 단 하나의 이유에 의해서만 변경 되어야 하고 클래스 안의 모든 코드는 함께 변경 되어야 한다. 간단히 말해 단일 책임 원칙을 따르도록 클래스를 분리해야 한다.

작은 클래스로 변경을 하면 일관성을 지키기 수월해진다.

캡슐화 다시 살펴보기

캡슐화는 데이터 은닉이 아니다. 소프트웨어 안에서 변할수 있는 모든 개념을 감추는 것이다.

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%209.png)

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%2010.png)

**데이터 캡슐화** : Movie 클래스의 인스턴스 변수 title의 가시성은 private이기 때문에 외부에서 직접 접근할 수 없다. 이 속성에 접근할 수 있는 유일한 방법은 메서드를 이용하는것 뿐이다. **다시말해 클래스는 내부에 관리하는 데이터를 캡슐화 한다.**

**메서드 캡슐화** : DiscountPolicy 클래스에서 정의돼 있는 getDiscountAmount 메서드의 가시성은 protected다. 클래스의 외부에서는 이 메서드에 직접 접근할 수 없고 클래스 내부와 서브클래스에서만 접근이 가능하다. 따라서 클래스 외부에 영향을 미치지 않고 메서드를 수정할수 있다. 다시말해 클래스 내부 행동을 캡슐화 하고 있는것이다.

**객체 캡슐화** : Movie 클래스는 DiscountPolicy 타입의 인스턴브 변수 discountPolicy를 포함한다. 이 인스턴스 변수는 private 가시성을 가지기 때문에 Movie와 DiscountPolicy 사이의 관계를 변경하더라도 외부에는 영향을 미치지 않는다. 다시 말해서 객체와 객체 사이의 관계를 캡슐화 한다. 눈치가 빠른 사람이라면 객체 캡슐화가 합성을 의미 한다는것을 눈치 챘을 것이다.

**서브타입 캡슐화** : Movie는 DiscountPolicy에 대해서는 알고있지만 AmountDiscountPolicy와 PercentDIscountPolicy에 대해서는 알지 못한다. 그러나 실제로 실행 시점에는 이 클래스들의 인스턴스와 협력할 수 있다. 이것은 기반 클래스인 DiscountPolicy와의 추상적인 관계가 AmountDiscountPolicy와 PercentDiscountPolicy의 존재를 감추고 있기 때문이다. 다시 말해 서브타입의 종류를 캡슐화 하고 있는것이다. 눈치가 빠른 사람이라면 서브타입 캡슐화가 다형성의 기반이 된다는 것을 알 수 있을 것이다.

다양한 캡슐화 방법이 존재하지만 일관성있게 만들기 위해 가장 일반적으로 사용하는 방법은 서브타입캡슐화와 객체 캡슐화를 조합하는 것이다. ( 위 사진 )

변경의 이유에 따라 캡슐화 할 수 있는 다양한 방법이 궁금하다면 디자인 패턴을 살펴보길 바란다.

## 3. 일관성 있는 기본 정책 구현하기

### 변경 분리하기

일관성 있는 협력을 만들기 위한 첫 단계: 변하는 개념과 변하지 않는 개념 분리

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%2011.png)

앞에서 구현한 통화 요금 과금 정책을 위와같이 정리할 수 있다.

- 기본 정책은 한 개 이상의 규칙으로 구성된다.
- 하나의 규칙은 적용조건과 단위요금의 조합이다.

이 정리에서 변하는 개념과 변하지 않는 개념을 분리할 수 있다.

- 변하는 개념: 적용 조건
- 변하지 않는 개념: 규칙

### 변경 캡슐화 하기

협력을 일관성 있게 만들기 위해서는 변경을 캡슐화 해서 파급효과를 줄여야 한다.

변하지 않는 부분 ‘규칙’ 에서 변하는 부분 ‘적용 조건’(변경)을 분리해서 추상화 한 후 시간대별, 요일별, 구간별 방식을 추상화의 서브타입으로 만든다. (서브타입 캡슐화)

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%2012.png)

- 변하지 않는 부분: BasicRatePolicy, FeeRule, FeeCondition
- 변하는 부분: TimeOfDayFeeCondition, DayOfWeekFeeCondition, DurationFeeCondition

변하는 부분이 FeeCondition으로 캡슐화됨

### 협력 패턴 설계하기

추상화만으로 협력을 하면 재사용 가능한 협력 패턴이 된다.

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%2013.png)

BasicRatePolicy가 FeeRule에서 요금을 계산하라는 메시지를 요청함으로서 협력이 구성된다.

FeeRule은 하위의 FeeCondition들에게 요금 계산에 필요한 시간 구간을 요청한다.

### 추상화 수준에서 협력 패턴 설계하기

1. 적용조건을 표현하는 추상화 인터페이스
    
    ```java
    public interface FeeCondition {
        List<DateTimeInterval> findTimeIntervals(Call call);
    }
    ```
    
2. 단위요금 과 적용조건 을 저장하고 요금을 계산하는 클래스
    
    ```java
    public class FeeRule {
        private FeeCondition feeCondition;
        private FeePerDuration feePerDuration;
    
        public FeeRule(FeeCondition feeCondition, FeePerDuration feePerDuration) {
            this.feeCondition = feeCondition;
            this.feePerDuration = feePerDuration;
        }
    
        public Money calculateFee(Call call) {
            return feeCondition.findTimeIntervals(call)
                    .stream()
                    .map(each -> feePerDuration.calculate(each))
                    .reduce(Money.ZERO, (first, second) -> first.plus(second));
        }
    }
    ```
    
3. 일정 기간동안의 요금을 계산하는 단위 시간당 요금 클래스
    
    ```java
    public class FeePerDuration {
        private Money fee;
        private Duration duration;
    
        public FeePerDuration(Money fee, Duration duration) {
            this.fee = fee;
            this.duration = duration;
        }
    
        public Money calculate(DateTimeInterval interval) {
            return fee.times(Math.ceil((double)interval.duration().toNanos() / duration.toNanos()));
        }
    }
    ```
    
4. 협력의 시작점인 기본 요금 정책
    
    ```java
    public final class BasicRatePolicy implements RatePolicy {
        private List<FeeRule> feeRules = new ArrayList<>();
    
        public BasicRatePolicy(FeeRule ... feeRules) {
            this.feeRules = Arrays.asList(feeRules);
        }
    
        @Override
        public Money calculateFee(Phone phone) {
            return phone.getCalls()
                    .stream()
                    .map(call -> calculate(call))
                    .reduce(Money.ZERO, (first, second) -> first.plus(second));
        }
    
        private Money calculate(Call call) {
            return feeRules
                    .stream()
                    .map(rule -> rule.calculateFee(call))
                    .reduce(Money.ZERO, (first, second) -> first.plus(second));
        }
    }
    ```
    

위 클래스/인터페이스들은 변하지 않는 추상화다.

이 요소들의 조합으로 전체적인 협력 구조가 완성된다.

변하지 않는 추상적인 요소만으로 요금 계산에 필요한 협력 구조를 설명할 수 있다.

실제로 동작하려면 인터페이스(FeeCondition)를 구현하는 구체적인 클래스가 필요하다.

### 구체적인  협력 구현하기

- (DateTimeInterval, 참고용으로 추가)
    
    ```java
    public class DateTimeInterval {
        private LocalDateTime from;
        private LocalDateTime to;
    
        public static DateTimeInterval of(LocalDateTime from, LocalDateTime to) {
            return new DateTimeInterval(from, to);
        }
    
        public static DateTimeInterval toMidnight(LocalDateTime from) {
            return new DateTimeInterval(from, LocalDateTime.of(from.toLocalDate(), LocalTime.of(23, 59, 59, 999_999_999)));
        }
    
        public static DateTimeInterval fromMidnight(LocalDateTime to) {
            return new DateTimeInterval(LocalDateTime.of(to.toLocalDate(), LocalTime.of(0, 0)), to);
        }
    
        public static DateTimeInterval during(LocalDate date) {
            return new DateTimeInterval(
                    LocalDateTime.of(date, LocalTime.of(0, 0)),
                    LocalDateTime.of(date, LocalTime.of(23, 59, 59, 999_999_999)));
        }
    
        private DateTimeInterval(LocalDateTime from, LocalDateTime to) {
            this.from = from;
            this.to = to;
        }
    
        public Duration duration() {
            return Duration.between(from, to);
        }
    
        public LocalDateTime getFrom() {
            return from;
        }
    
        public LocalDateTime getTo() {
            return to;
        }
    
        public List<DateTimeInterval> splitByDay() {
            if (days() > 0) {
                return split(days());
            }
    
            return Arrays.asList(this);
        }
    
        private long days() {
            return Duration.between(from.toLocalDate().atStartOfDay(), to.toLocalDate().atStartOfDay()).toDays();
        }
    
        private List<DateTimeInterval> split(long days) {
            List<DateTimeInterval> result = new ArrayList<>();
            addFirstDay(result);
            addMiddleDays(result, days);
            addLastDay(result);
            return result;
        }
    
        private void addFirstDay(List<DateTimeInterval> result) {
            result.add(DateTimeInterval.toMidnight(from));
        }
    
        private void addMiddleDays(List<DateTimeInterval> result, long days) {
            for(int loop=1; loop < days; loop++) {
                result.add(DateTimeInterval.during(from.toLocalDate().plusDays(loop)));
            }
        }
    
        private void addLastDay(List<DateTimeInterval> result) {
            result.add(DateTimeInterval.fromMidnight(to));
        }
    
        public String toString() {
            return "[ " + from + " - " + to + " ]";
        }
    }
    ```
    

1. 시간대별 정책
    
    ```java
    public class TimeOfDayFeeCondition implements FeeCondition {
        private LocalTime from;
        private LocalTime to;
    
        public TimeOfDayFeeCondition(LocalTime from, LocalTime to) {
            this.from = from;
            this.to = to;
        }
    
        @Override  // 시간대별 정책으로 요금 구간을 계산
        public List<DateTimeInterval> findTimeIntervals(Call call) {
            return call.getInterval().splitByDay()
                    .stream()
                    .filter(each -> from(each).isBefore(to(each)))
                    .map(each -> DateTimeInterval.of(
                                    LocalDateTime.of(each.getFrom().toLocalDate(), from(each)),
                                    LocalDateTime.of(each.getTo().toLocalDate(), to(each))))
                    .collect(Collectors.toList());
        }
    
        private LocalTime from(DateTimeInterval interval) {
            return interval.getFrom().toLocalTime().isBefore(from) ?
                    from : interval.getFrom().toLocalTime();
        }
    
        private LocalTime to(DateTimeInterval interval) {
            return interval.getTo().toLocalTime().isAfter(to) ?
                    to : interval.getTo().toLocalTime();
        }
    }
    
    ```
    
2. 요일별 정책
    
    ```java
    public class DayOfWeekFeeCondition implements FeeCondition {
        private List<DayOfWeek> dayOfWeeks = new ArrayList<>();
    
        public DayOfWeekFeeCondition(DayOfWeek ... dayOfWeeks) {
            this.dayOfWeeks = Arrays.asList(dayOfWeeks);
        }
    
        @Override
        public List<DateTimeInterval> findTimeIntervals(Call call) {
            return call.getInterval()
                    .splitByDay()
                    .stream()
                    .filter(each ->
                            dayOfWeeks.contains(each.getFrom().getDayOfWeek()))
                    .collect(Collectors.toList());
        }
    }
    ```
    
3. 구간별 정책
    
    ```java
    public class DurationFeeCondition implements FeeCondition {
        private Duration from;
        private Duration to;
    
        public DurationFeeCondition(Duration from, Duration to) {
            this.from = from;
            this.to = to;
        }
    
        @Override
        public List<DateTimeInterval> findTimeIntervals(Call call) {
            if (call.getInterval().duration().compareTo(from) < 0) {
                return Collections.emptyList();
            }
    
            return Arrays.asList(DateTimeInterval.of(
                    call.getInterval().getFrom().plus(from),
                    call.getInterval().duration().compareTo(to) > 0 ?
                            call.getInterval().getFrom().plus(to) :
                            call.getInterval().getTo()));
        }
    }
    ```
    

변하는 부분을 변하지 않는 부분에서 변경함으로 변하지 않는 부분 (BasicRatePolicy, FeePerDuration, FeeRule) 은 재사용하면서 변하는 부분 (xxxxFeeCondition) 을 확장할 수 있다.

일관성 있는 협력은 개발자에게 확장 포인트 (FeeCondition)를 강제하기 때문에 정해진 구조를 우회하기 어렵게 만든다.

개발자는 코드의 형태로 주어진 제약 안에 머물러야 하지만 작은 문제에 집중할 수 있는 자유를 얻는다.

이렇게 일관된 구조는 협력 패턴을 이해하기만 하면 변하는 부분만 따로 떼어내어 독립적으로 이해하더라도 전체적인 구조를 쉽게 이해할 수 있다.

### 협력 패턴 맞추기

고정 요금 정책은 규칙이라는 개념이 필요없다.

하지만 기존의 협력 방식을 벗어나는것보다는 조금 이상하더라도 전체적인 일관성을 유지할수 있는 설계를 선택하는 것이 좋다.

아래와 같이 구현하면, 조금 이상하지만 기존의 협력방식을 따르게 할 수 있다.

```java
public class FixedFeeCondition implements FeeCondition {
    @Override
    public List<DateTimeInterval> findTimeIntervals(Call call) {
        return Arrays.asList(call.getInterval());
    }
}
```

개념적 무결성(일관성)을 무너뜨리는것보다 이 방법이 훨씬 좋다.

### 결과

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%2014.png)

![Untitled](Chapter%2014%20%E1%84%8B%E1%85%B5%E1%86%AF%E1%84%80%E1%85%AA%E1%86%AB%E1%84%89%E1%85%A5%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%BB%E1%84%82%E1%85%B3%E1%86%AB%20%E1%84%92%E1%85%A7%E1%86%B8%E1%84%85%E1%85%A7%E1%86%A8%20166ebc53a3844a5ca06b00edad5ecbbe/Untitled%2015.png)