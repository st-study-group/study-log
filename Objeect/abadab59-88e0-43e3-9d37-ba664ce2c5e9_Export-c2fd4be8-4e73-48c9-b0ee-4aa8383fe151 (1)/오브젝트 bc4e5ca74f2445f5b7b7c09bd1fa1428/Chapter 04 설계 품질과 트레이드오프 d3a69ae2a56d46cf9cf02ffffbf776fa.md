# Chapter 04 설계 품질과 트레이드오프

## 1. 데이터 중심의 영화 예매 시스템

## 훌륭한 객체 지향 설계

- 책임을 중심으로 객체의 행동에 초점을 맞춘 설계
- 응집도를 높이고, 결합도를 낮춰서 쉽게 변경할 수 있은 설계

## 데이터 중심의 설계

- 상태(데이터)를 중심으로 객체를 분할하는 설계
- 객체는 자신이 포함하고 있는 상태(데이터)를 조작하는데 필요한 오퍼레이션을 정의

## 데이터 중심의 설계를 하는 방법

1. 객체가 내부에 저장해야 하는 데이터를 정의한다.

```java
public class Movie {
    private String title;
    private Duration runningTime;
    private Money fee;
    private List<DiscountCondition> discountConditions;

    private MovieType movieType;
    private Money discountAmount;
    private double discountPercent;
}

public enum MovieType {
    AMOUNT_DISCOUNT,    // 금액 할인 정책
    PERCENT_DISCOUNT,   // 비율 할인 정책
    NONE_DISCOUNT       // 미적용
}
```

2. 캡슐화를 위한 getter, setter를 만든다

```java
public class Movie {
    private String title;
    private Duration runningTime;
    private Money fee;
    private List<DiscountCondition> discountConditions;

		// 이 타입으로 어떤 데이터를 사용할지를 결정
    private MovieType movieType;
    private Money discountAmount;
    private double discountPercent;

    public MovieType getMovieType() {
        return movieType;
    }

    public void setMovieType(MovieType movieType) {
        this.movieType = movieType;
    }

    public Money getFee() {
        return fee;
    }

    public void setFee(Money fee) {
        this.fee = fee;
    }

    public List<DiscountCondition> getDiscountConditions() {
        return Collections.unmodifiableList(discountConditions);
    }

    public void setDiscountConditions(List<DiscountCondition> discountConditions) {
        this.discountConditions = discountConditions;
    }

    public Money getDiscountAmount() {
        return discountAmount;
    }

    public void setDiscountAmount(Money discountAmount) {
        this.discountAmount = discountAmount;
    }

    public double getDiscountPercent() {
        return discountPercent;
    }

    public void setDiscountPercent(double discountPercent) {
        this.discountPercent = discountPercent;
    }
}
```

1. 1, 2번을 반복한다.

```java
public class DiscountCondition {
    private DiscountConditionType type;

    private int sequence;

    private DayOfWeek dayOfWeek;
    private LocalTime startTime;
    private LocalTime endTime;
}

public enum DiscountConditionType {
    SEQUENCE,       // 순번조건
    PERIOD          // 기간 조건
}
```

```java
public class DiscountCondition {
    private DiscountConditionType type;

    private int sequence;

    private DayOfWeek dayOfWeek;
    private LocalTime startTime;
    private LocalTime endTime;

    public DiscountConditionType getType() {
        return type;
    }

    public void setType(DiscountConditionType type) {
        this.type = type;
    }

    public DayOfWeek getDayOfWeek() {
        return dayOfWeek;
    }

    public void setDayOfWeek(DayOfWeek dayOfWeek) {
        this.dayOfWeek = dayOfWeek;
    }

    public LocalTime getStartTime() {
        return startTime;
    }

    public void setStartTime(LocalTime startTime) {
        this.startTime = startTime;
    }

    public LocalTime getEndTime() {
        return endTime;
    }

    public void setEndTime(LocalTime endTime) {
        this.endTime = endTime;
    }

    public int getSequence() {
        return sequence;
    }

    public void setSequence(int sequence) {
        this.sequence = sequence;
    }
}
```

```java
public class Reservation {
    private Customer customer;
    private Screening screening;
    private Money fee;
    private int audienceCount;

    public Reservation(Customer customer, Screening screening, Money fee,
                       int audienceCount) {
        this.customer = customer;
        this.screening = screening;
        this.fee = fee;
        this.audienceCount = audienceCount;
    }

    public Customer getCustomer() {
        return customer;
    }

    public void setCustomer(Customer customer) {
        this.customer = customer;
    }

    public Screening getScreening() {
        return screening;
    }

    public void setScreening(Screening screening) {
        this.screening = screening;
    }

    public Money getFee() {
        return fee;
    }

    public void setFee(Money fee) {
        this.fee = fee;
    }

    public int getAudienceCount() {
        return audienceCount;
    }

    public void setAudienceCount(int audienceCount) {
        this.audienceCount = audienceCount;
    }
}
```

```java
public class Screening {
    private Movie movie;
    private int sequence;
    private LocalDateTime whenScreened;

    public Movie getMovie() {
        return movie;
    }

    public void setMovie(Movie movie) {
        this.movie = movie;
    }

    public LocalDateTime getWhenScreened() {
        return whenScreened;
    }

    public void setWhenScreened(LocalDateTime whenScreened) {
        this.whenScreened = whenScreened;
    }

    public int getSequence() {
        return sequence;
    }

    public void setSequence(int sequence) {
        this.sequence = sequence;
    }
}
```

1. 위에서 만든 객체들을 조합해서 영화를 예매하는 클래스를 만든다.

```java
public class ReservationAgency {
    public Reservation reserve(Screening screening, Customer customer,
                               int audienceCount) {
        Movie movie = screening.getMovie();

        boolean discountable = false;
        for(DiscountCondition condition : movie.getDiscountConditions()) {
            if (condition.getType() == DiscountConditionType.PERIOD) {
                discountable = screening.getWhenScreened().getDayOfWeek().equals(condition.getDayOfWeek()) &&
                        condition.getStartTime().compareTo(screening.getWhenScreened().toLocalTime()) <= 0 &&
                        condition.getEndTime().compareTo(screening.getWhenScreened().toLocalTime()) >= 0;
            } else {
                discountable = condition.getSequence() == screening.getSequence();
            }

            if (discountable) {
                break;
            }
        }

        Money fee;
        if (discountable) {
            Money discountAmount = Money.ZERO;
            switch(movie.getMovieType()) {
                case AMOUNT_DISCOUNT:
                    discountAmount = movie.getDiscountAmount();
                    break;
                case PERCENT_DISCOUNT:
                    discountAmount = movie.getFee().times(movie.getDiscountPercent());
                    break;
                case NONE_DISCOUNT:
                    discountAmount = Money.ZERO;
                    break;
            }

            fee = movie.getFee().minus(discountAmount).times(audienceCount);
        } else {
            fee = movie.getFee().times(audienceCount);
        }

        return new Reservation(customer, screening, fee, audienceCount);
    }
}
```

## 2.  설계 트레이드 오프

### 캡슐화

- **변경 가능성**(구현)이 높은 부분은 내부에 숨기고, 상대적으로 **안정된 부분**(인터페이스)만 외부에 공개하여 변경의 여파를 통제
- 구현과 인터페이스를 분리하는 것이 핵심
- 캡슐화를 지키면, 높은 응집도와 낮은 결합도를 얻을 수 있다.

### 응집도

- 객체, 클래스에 얼마나 관련있는 책임을 할당했는가의 정도
- 하나의 변경(요구사항)에 대해 하나의 모듈만 변경된다면 응집도가 높다.
- 응집도가 높을 수록 변경 대상이 명확해 지기 때문에, 코드를 변경하기 쉽다.

### 결합도

- 서로 다른 모듈에 대해 얼마나 많은 지식을 가지고 있는지 정도
- 하나의 모듈이 변경되기 위해 다른 모듈을 더 많이 변경해야 할 수록 결합도가 높다.
내부 구현을 변경했을때, 다른 모듈에 영향이 가면 결합도가 높다.
인터페이스를 변경했을때만 다른 모듈에 영향이 가는 것은 결합도가 낮다.
- String, ArrayList 같은 표준 라이브러리나 안정된 프레임워크는 변경될 확률이 매우 적기 때문에 결합도가 높아도 상관이 없다.

## 3. 데이터 중심의 영화 예매 시스템의 문제점

**1. 캡슐화 위반**

- 캡슐화를 위해 만든 getter, setter는 아무것도 캡슐화 하지 못한다.
getter, setter는 객체가 가진 모든 상태를 외부에 퍼블릭 인터페이스로 공개하고 있다.
협력을 고려하지 않고 설계하면 객체가 사용될 문맥을 추측해야 하고, 모든 경우에 대비해서 getter, setter를 만들어야 한다. 이런 설계 방식을 추측에 의한 설계 전략이라고 한다.
- 내부 구현이 외부에 노출되어 캡슐화를 위반하기 때문에, 변경에 취약해진다.

**2. 높은 결합도**

- getter, setter로 내부 구현을 외부에 드러내면, 객체간의 결합도가 올라간다.
예시로 Movie의 상태인 fee의 타입을 변경한다면, 퍼블릭 인터페이스인 getFee 메서드도 변경되고 getFee를 사용하는 ReservationAgency 도 변경되게 된다.
- 제어 로직이 한군데로 집중되는것도 문제가 된다.
모든 객체들이 ReservationAgency에 결합되어 있기 때문에, 어떤 객체를 수정하더라도 ReservationAgency도 함께 수정해야하게 된다.

**3. 낮은 응집도**

- 책임이 다른 코드들이 한군데에 뭉쳐있어서, 하나를 변경하려고 했는데 아무 상관없는 코드가 영향을 받을 수 있게 된다.
ReservationAgency에서 할인 조건만 수정하려고 했는데 할인정책 관련 코드도 섞여있어서 수정할 필요가 없는 부분에 영향을 줄 수 있다.
- 하나의 요구사항 변경을 반영하기 위해 여러개의 모듈을 수정해야 한다.
할인 정책을 추가하는데 ReservationAgency, Movie, MovieType 세군데나 변경해야한다.

---

## 4장

### 캡슐화를 지켜라

캡슐화는 설계의 제 1원리이다. 낮은 응집도와 높은 결합도의 문제로 몸살을 앓는다면 캡슐화의 원칙을 위반했기 때문이다. private를 설정했다 해도 getter setter를 이용하여 외부로 제공한다면 이것은 캡슐화를 위반한 것이다.

위반의예시

```java
class Rectangle{
	private int left;
	private int top;
	private int right;
	private int bottom;

	public Rectangle(int left, int top, int right, int bottom){
		this.left = left;
		this.top = top;
		this.right = right;
		this.bottom = bottom;
	}

	public int getLeft() { return left; }
	public void setLeft(int left) { this.left = left; }

	public int getTop() { return top; }
	public void setTop(int top) { this.top = top; }

	public int getRight() { return right; }
	public void setRight(int right) { this.right = right; }

	public int getBottom() { return bottom; }
	public void setBottom(int bottom) { this.bottom = bottom; }

}
```

```java
class AnyClass {
	void antMethod(Rectangle rectangle, int multiple) {
		rectangle.setRight(rectangle.getRight() * multiple);
		rectangle.setBottom(rectangle.getBottom() * multiplt);
		...
	}
}
```

위 코드는 많은 문제점이 있다. 

1. 코드중복이 발생할 확률이 높다. 

<aside>
💡 ex) Rectangle에서 bottom을 증가시키기 위해서는 getBottom 메서드를 호출해서 setBottom 메서드로 넣어야 한다.

</aside>

1. 변경에 취약하다

<aside>
💡 ex) right → length 변경시에는 getRight, setRight 메서드를 getLength, setLength 메서드로 변경해야 한다.

</aside>

해결방법은 캡슐화의 강화이다. 

```java
class Rectangle {
	public void enlarge(int multiple){
		right *= multiple;
		bottom *= multiple;
	}
  ...
}
```

위 코드는 Rectangle을 변경하는 주체를 외부의 객체에서 Rectangle로 이동시켰다. 즉 자신의 크기를 Rectangle 스스로 증가시키도록 **책임을 이동**시킨것이다.

### 스스로 자신의 데이터를 책임지는 객체

상태와 행동을 객체라는 하나의 단위로 묶는 이유는 객체 스스로 자신의 상태를 처리하기 위해서이다.

객체를 설계할때 생각해야 하는 부분

**1 ) 이 객체가 어떤 데이터를 포함해야 하는가?**

**2) 이 객체가 데이터에 대해 수행해야 하는 오퍼레이션은 무엇인가? *( 오퍼레이션 : 객체의 기능 )***

따라서 이 객체가 어떤 데이터를 포함하고 수행해야 하는 오퍼레이션은 무엇인가이다.

영화 예매 시스템 예제로 돌아가 ReservationAgency로 새어나간 데이터의 책임을 실제 데이터를 포함하고 있는 객체로 옮긴다.

```java
// 어떤 데이터를 관리 해야하는지?
public class DiscountCondition {
	private DiscountConditionType type;
	private int sequence;
	private DayOfWeek dayOfWeek;
	private LocalTime startTime;
	private LocalTime endTime;
}
```

```java
// 어떤 오퍼레이션을 가져야 할지?
public class DiscountCondition {
	public DiscountConditionType getType(){
		return type; // 할인 조건이 정상적으로 들어왔는지 판단
	}

	// 기간조건 할인 여부 지정 오퍼레이션
	public boolean isDiscountable(DayOfWeek dayOfWeek, LocalTime time){
		if(type != DiscountConditionType.PERIOD){
			throw new IllegalArgumentException();
		}
		return this.dayOfWeek.equals(dayOfWeek) &&
						this.startTime.compareTo(time <= 0 &&
						this.endTime.compareTo(time) >= 0;
	}

	// 순번조건 할인 여부 지정 오퍼레이션
	public boolean isDiscountable(int sequence) {
		if(type != DiscountConditionType.SEQUENCE) {
			throw new IllegalArgumentException();
		}	
		return this.sequence == sequence;
	}
}
```

```java
// 어떤 데이터를 포함해야 하는지?
public class Movie {
	private String title;
	private Duration runningTime;
	private Money fee;
	private List<DiscountCondition> discountConditions;

	private MovieType movieType;
	private Money discountAmount;
	private double discountPercent;
}
```

```java
// 이 클래스가 가질 오퍼레이션은? -> 할인 정책을 판단하는 오퍼레이션
public class Movie {
	public MovieType getMovieType() {
		return movieType;
	}
	
	// 금액 할인 정책 오퍼레이션
	public Money calculateAmountDiscountedFee(){
		if(movieType != MovieType.AMOUNT_DISCOUNT){
			throw new IllegalArgumentException();
		}	
		return fee.mius(discountAmount);
	}

	// 퍼센트 할인 정책 오퍼레이션
	public Money calclatePercentDiscountedFee() {
		if(movieType != MovieType.PERCENT_DISCOUNT) {
			throw new IllegalArgumentException();
		}
		return fee.minus(fee.times(discoountPercent);
	}

	// 미적용 오퍼레이션
	public Money calculateNoneDiscountedFee(){
		if(movieType != MovieType.NONE_DISCOUNT) {
			throw new IllegalArgumentException();
		}
		return fee;
	}
}
```

```java
public class Movie {
	// 할인 여부를 판단하는 오퍼레이션
	public boolean isDiscountable(LocalDateTime whenScreened, int sequence){
		for(DiscountCondition condition : discountConditions){
			
			if(condition.getType() == DiscountConditionType.PERIOD){ // 할인조건이 기간조건
				if(condition.isDiscountable(whenScreened.getDayofWeek(), whenScreend.toLcalTime())){
					return true;
				}
			}else{ // 할인조건이 순번조건
				if(condition.isDiscountable(sequence)){
					return true;
				}
			}
		}
	}
}
```

```java
public class Screening {
	private Movie movie;
	private int sequence;
	private LocalDateTime whenScreend;

	public Screnning(Movie movie, int sequence, LocalDateTime whenScreened){
		this.movie = movie;
		this.sequence = sequence;
		this.whenScreened = whenScreend;
	}

  // 할인정책을 지원할 경우 할인이 가능한지 체크하는 오퍼레이션
	public Money calcculateFee(int audienceCount){
		switch(movie.getMovieType()){
			case AMOUNT_DISCOUNT:
				if(movie.isDiscountalbe(whenScreened, sequence){
					return movie.calculateAmountDiscountFee().times(audienceCount);
				}
				break;
			case PERCENT_DISCOUNT:
				if(movie.isDiscountable(whenScreened, sequence)){
					return movie.calculatePercentDiscountedFee().times(audienceCount);
				}
			case NONE_DISCOUNT:
				return movie.calculateNoneDiscountedFee().times(audienceCount);
		}

		return movie.calculateNoneDiscountedFee().times(audienceCount);
	}
}
```

```java
public class ReservationAgency {
	public Reservation reserve(Screening, Customer customer, int audienceCount) {
		// 요금을 계산
		Money fee = screening.calculateFee(audienceCount);
		// 요금을 이용해 예약을 생성한다
		return new Reservation(customer, screening, fee, audienceCount);
	}
}
```

위 객체들은 스스로를 책임진다 말할수 있다.

하지만 수정 전 코드에서 발생했던 문제 대부분이 똑같이 나타난다.

```java
public class DiscountCondition{
	private DiscountConditionType type;
	private int sequence;
	private DayOfWeek dayOfWeek;
	private LocalTime startTime;
	private LocalTime endTime;

	public DiscountConditionType getType(){ ... }
	public boolean isDiscountable(DayOfWeek dayOfWeek, LocalTime, time){ ... }
	public boolean isDiscountable(int sequence) { ... }
}
```

분명 자신의 상태를 스스로 관리하는것이 객체를 지향하는것처럼 보인다. 하지만 이상한점이 몇군대가 있다.

1. isDiscountable 클래스는 DayOfWeek의 시간 정보를 파라미터로 받는다. 그러므로 DayOfWeek클래스를 사용한다고 외부로 노출이 되는것이다.
2. isDiscountable 클래스의 sequence도 마찬가지다.

이렇게 내부정보가 외부로 퍼져나가는것을 파급효과(ripple effect)는 캡슐화의 부족함의 증거이다.

```java
public class Movie {
	private String title;
	private Duration runningTime;
	private Money fee;
	private List<DiscountCondition> discountConditions;

	private MovieType movieType;
	private Money discountAmount;
	private double discountPercent;
	
	public MovieType getMovieType() { ... }
	public Money calculateAmountDiscountedFee() { ... }
	public Money calculatePercentDiscountedFee() { ... }
	public Money calculateNoneDiscountedFee() { ... }
}
```

위에것과 다르다 생각이 들수도 있지만 같다. 내부 구현을 인터페이스에 노출시키고 있기 때문이다.

calculateAmountDiscountedFee, calculatePercentDiscountedFee, calculateNoneDiscountedFee 라는 메서드가 있다고 알리고 있다.

<aside>
💡 캡슐화의 진정한 의미
위 예제들로 캡슐화는 데이터를 감추는 것 이상의 의미를 가진다. 캡슐화는 변경될 수 있는 어떠한것이라도 감추는것을 의미한다.

</aside>

### 높은 결합도

캡슐화 위반으로 인해 내부 구현이 외부로 노출되었기 때문에 두 클래스들의 결합도는 높을 수 밖에 없다.

→ Money와 DiscountCondition의 결합도는 높다

결합도가 높으면 나오는 문제점

```java
public class Movie{
	public boolean isDiscountable(LocalDateTime whenScreened, int sequence){
		for(DiscountCondition condition : discountConditions){
			if(condition.getType() == DiscountConditionType.PERIOD){
				if(condition.isDiscountable(whenScreened.getDayOfWeek(), 
						whenScreened.toLocalTime())){
					return true;		
				}
			}else{
				if(condition.isDiscountable(sequence)){
					return true;
				}	
			}
		}
		return false;
	}
}
```

1. DiscountCondtion의 기간할인조건의 명칭이 PERIOD에서 다른 값으로 변경되면 Movie를  수정해야한다.
2. DiscountCondition의 종류가 추가되거나 삭제된다면 Movie안에 if ~ else 구문을 수정해야한다.
3. DiscountCondition의 만족 여부를 판단하는데 필요한 정보가 변경되면 Movie를 수정해야한다.
    
    movie가 수정이 되면 Screening에 대한 변경을 초래한다.
    

### 낮은 응집도

```java
public class Screening{
	public Money calculateFee(int audienceCount){
		switch (movie.getMovieType()){
			case AMOUNT_DISCOUNT:
				if(movie.isDiscountalbe(whenScreened, sequence)){
					return movie.calculateAmountDiscountedFee().times(audienceCount);
				}
				break;
			case PERCENT_DISCOUNT:
				if(movie.isDiscountable(whenScreened, sequece)){
					return movie.calculatePercentDiscountedFee().times(audienceCount);
				}
			case NONE_DISCOUNT:
				return movie.calculateNoneDiscountedFee().times(audienceCount);
		}
	
		return movie.**calculateNoneDiscountedFee**().times(audienceCount);
	}
}
```

위 코드는 할인 조건이나 종류를 변경하기 위해서는 DiscountCondition, Movie, Movie를 사용하는 Screening을 함께 수정해야한다. 

하나를 변경을 하기 위해서 코드 여러곳을 동시에 수정해야 하는것은 응집도가 낮다는 것이다.

이유 : DiscountCondition과 Movie의 내부 구현의 인터페이스가 그대로 노출되고 있고 Screening은 노출된 구현에 직접적으로 의존하고 있다. 

Movie에 있어야 할 로직들이 그대로 나타나있다.

### 데이터 중심 설계의 문제점

설계가 변경에 유연하지 못한 이유는 캡슐화를 위반했기 때문이다. 

데이터 중심의 설계가 변경에 취약한 이유는 두가지이다

<aside>
💡 1. 데이터 중심의 설계는 본질적으로 너무 이른 시기에 데이터에 관해 결정하도록 강요한다.
2. 데이터 중심의 설계에서는 협력이라는 문맥을 고려하지 않고 객체를 고립시킨 채 오퍼레이션을 결정한다.

</aside>

**데이터 중심 설계는 객체의 행동보다는 상태에 초점을 맞춘다.**

데이터중심 설계는 절차적 프로그래밍을 따르기 때문에 객체지향에 반한다.

데이터중심 설계에서 객체는 단순 데이터의 집합체일 뿐이다. 이로인해서 getter, setter를 과도하게 추가하게 되고 이는 public 속성과 큰 차이가 없기때문에 캡슐화는 무너진다.

작업과 데이터를 같은 객체에 두더라도 데이터에 초점이 맞추어져 있다면 데이터에 관한 지식들이 인터페이스에 고스란히 나타나고 코드 변경에 취약해져 캡슐화는 실패한다.

결론적으로 데이터를 너무 이른 시기에 고민하게 되고 내부 변경에 취약한 코드를 낳게 된다.

**데이터 중심 설계는 객체를 고립시킨 채 오퍼레이션을 정의하도록 만든다.**

객체지향 어플리케이션을 구현한다는 것은 협력하는 객체들의 공동체를 구축한다는 것이다.

올바른 객체지향 설계의 무게중심은 항상 객체의 내부가 아니라 외부에 맞춰져 있어야 한다. 하지만 데이터 중심 설계에서 초점은 객체의 외부가 아니라 내부로 향한다.

객체의 구현이 이미 결정된 상태에서 다른 객체와의 협력방법을 고민하기 때문에 협력이 힘들다.