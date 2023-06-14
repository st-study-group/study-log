# Chapter 10 상속과 코드 재사용

## 1. 상속과 중복 코드

### DRY 원칙

중복코드를 제거해야하는 이유: 변경을 방해한다.

중복코드가 변경에 미치는 영향

1. 모든 중복코드를 찾아내야한다.
2. 모든 중복 코드를 변경해야한다.
3. 변경된 모든 중복 코드를 테스트해야한다.

앤드류 헌트와 데이비드 토마스의 말을 인용하면

모든 프로그래머는 DRY 원칙 을 따라야 한다.

DRY : Dont Repeat Yourself 반복하지 마라

(Once and Only Once 원칙 
Single-Point Control 원칙 으로도 불린다.)

### 중복과 변경

핸드폰 요금 계산 클래스

```java
public class Call {
    private LocalDateTime from;
    private LocalDateTime to;

    public Call(LocalDateTime from, LocalDateTime to) {
        this.from = from;
        this.to = to;
    }

    public Duration getDuration() {
        return Duration.between(from, to);
    }

    public LocalDateTime getFrom() {
        return from;
    }
}
```

```java
public class Phone {
    private Money amount;
    private Duration seconds;
    private List<Call> calls = new ArrayList<>();

    public Phone(Money amount, Duration seconds) {
        this.amount = amount;
        this.seconds = seconds;
    }

    public void call(Call call) {
        calls.add(call);
    }

    public List<Call> getCalls() {
        return calls;
    }

    public Money getAmount() {
        return amount;
    }

    public Duration getSeconds() {
        return seconds;
    }

    public Money calculateFee() {
        Money result = Money.ZERO;

        for(Call call : calls) {
            result = result.plus(amount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
        }

        return result;
    }
}
```

사용코드

```java
Phone phone = new Phone(Money.wons(5), Duration.ofSeconds(10));
phone.call(new Call(
						LocalDateTime.of(2018, 1,1,12,10,0), 
						LocalDateTime.of(2018, 1,1,12,11,0)
					)
phone.calculateFee();
```

위 클래스를 복붙해서 야간 요금을 따로 계산하는 클래스를 만들 수 있다.

```java
public class NightlyDiscountPhone {
    private static final int LATE_NIGHT_HOUR = 22;

    private Money nightlyAmount;
    private Money regularAmount;
    private Duration seconds;
    private List<Call> calls = new ArrayList<>();

    public NightlyDiscountPhone(Money nightlyAmount, Money regularAmount, Duration seconds) {
        this.nightlyAmount = nightlyAmount;
        this.regularAmount = regularAmount;
        this.seconds = seconds;
    }

    public Money calculateFee() {
        Money result = Money.ZERO;

        for(Call call : calls) {
            if (call.getFrom().getHour() >= LATE_NIGHT_HOUR) {
                result = result.plus(nightlyAmount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
            } else {
                result = result.plus(regularAmount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
            }
        }

        return result;
    }
}
```

⇒ 복붙하면 편하다.

### 중복코드 수정하기

통화 요금에 세금을 부과하는 요구사항이 추가되었다.

```java
public class Phone {
    private Money amount;
    private Duration seconds;
    private List<Call> calls = new ArrayList<>();
    private double taxRate;

    public Phone(Money amount, Duration seconds, double taxRate) {
        this.amount = amount;
        this.seconds = seconds;
        this.taxRate = taxRate;
    }

    public void call(Call call) {
        calls.add(call);
    }

    public List<Call> getCalls() {
        return calls;
    }

    public Money getAmount() {
        return amount;
    }

    public Duration getSeconds() {
        return seconds;
    }

    public Money calculateFee() {
        Money result = Money.ZERO;

        for(Call call : calls) {
            result = result.plus(amount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
        }

        return result.plus(result.times(taxRate));
    }
}

public class NightlyDiscountPhone {
    private static final int LATE_NIGHT_HOUR = 22;

    private Money nightlyAmount;
    private Money regularAmount;
    private Duration seconds;
    private List<Call> calls = new ArrayList<>();
    private double taxRate;

    public NightlyDiscountPhone(Money nightlyAmount, Money regularAmount, Duration seconds, double taxRate) {
        this.nightlyAmount = nightlyAmount;
        this.regularAmount = regularAmount;
        this.seconds = seconds;
        this.taxRate = taxRate;
    }

    public Money calculateFee() {
        Money result = Money.ZERO;

        for(Call call : calls) {
            if (call.getFrom().getHour() >= LATE_NIGHT_HOUR) {
                result = result.plus(nightlyAmount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
            } else {
                result = result.plus(regularAmount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
            }
        }

        return result.minus(result.times(taxRate));
    }
}
```

⇒ 중복 코드를 모두 확인하고 고쳐야한다.

만약 둘중 하나만 수정하는 경우 오류가 발생한다.

또한, 동일하게 수정해야하는데 실수하기 쉽다. → 두 코드에서 잘못된 곳을 찾아보자

중복코드를 제거하지 않고 코드를 수정하는 유일한 방법은 중복코드를 추가하는것이다.

따라서, 최대한 중복코드를 제거하라.

### 타입 코드 사용하기

중복코드를 제거하는 방법중 하나로 클래스를 합치고 타입 코드를 추가하는 방법이 있다.

```java
public class Phone {
    private static final int LATE_NIGHT_HOUR = 22;
    enum PhoneType { REGULAR, NIGHTLY }

    private PhoneType type;

    private Money amount;
    private Money regularAmount;
    private Money nightlyAmount;
    private Duration seconds;
    private List<Call> calls = new ArrayList<>();

    public Phone(Money amount, Duration seconds) {
        this(PhoneType.REGULAR, amount, Money.ZERO, Money.ZERO, seconds);
    }

    public Phone(Money nightlyAmount, Money regularAmount,
                 Duration seconds) {
        this(PhoneType.NIGHTLY, Money.ZERO, nightlyAmount, regularAmount,
                seconds);
    }

    public Phone(PhoneType type, Money amount, Money nightlyAmount,
                 Money regularAmount, Duration seconds) {
        this.type = type;
        this.amount = amount;
        this.regularAmount = regularAmount;
        this.nightlyAmount = nightlyAmount;
        this.seconds = seconds;
    }

    public Money calculateFee() {
        Money result = Money.ZERO;

        for(Call call : calls) {
            if (type == PhoneType.REGULAR) {
                result = result.plus(amount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
            } else {
                if (call.getFrom().getHour() >= LATE_NIGHT_HOUR) {
                    result = result.plus(nightlyAmount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
                } else {
                    result = result.plus(regularAmount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
                }
            }
        }

        return result;
    }
}
```

⇒ 낮은 응집도와 높은 결합도를 가지게 된다. 

이렇게 하지말자

### 상속을 이용해서 중복 코드 제거하기

상속으로 코드를 재사용하자

```java
public class NightlyDiscountPhone extends Phone {
    private static final int LATE_NIGHT_HOUR = 22;

    private Money nightlyAmount;

    public NightlyDiscountPhone(Money nightlyAmount, Money regularAmount, Duration seconds) {
        super(regularAmount, seconds);
        this.nightlyAmount = nightlyAmount;
    }

    @Override
    public Money calculateFee() {
        // 부모클래스의 calculateFee 호출
        Money result = super.calculateFee();

        Money nightlyFee = Money.ZERO;
        for(Call call : getCalls()) {
            if (call.getFrom().getHour() >= LATE_NIGHT_HOUR) {
                nightlyFee = nightlyFee.plus(
                        getAmount().minus(nightlyAmount).times(
                                call.getDuration().getSeconds() / getSeconds().getSeconds()));
            }
        }

        return result.minus(nightlyFee);
    }
}
```

⇒ 코드 중복없이 Phone의 대부분의 코드를 재사용했다.

다만 요금을 계산하는 부분이 개발자의 생각을 이해하지 못하고 코드만 봐서는 해석하기 어렵다.

이건 상속을 염두에 두고 설계되지 않은 클래스를 상속해서 발생하는 문제다.

### 강하게 결합된 Phone 과 NightlyDiscountPhone

위 코드에서 세금이 부과되는 요구사항을 처리해보자

```java
public class Phone {
    private Money amount;
    private Duration seconds;
    private List<Call> calls = new ArrayList<>();
    private double taxRate;

    public Phone(Money amount, Duration seconds, double taxRate) {
        this.amount = amount;
        this.seconds = seconds;
        this.taxRate = taxRate;
    }

    public void call(Call call) {
        calls.add(call);
    }

    public List<Call> getCalls() {
        return calls;
    }

    public Money getAmount() {
        return amount;
    }

    public Duration getSeconds() {
        return seconds;
    }

    public Money calculateFee() {
        Money result = Money.ZERO;

        for(Call call : calls) {
            result = result.plus(amount.times(call.getDuration().getSeconds() / seconds.getSeconds()));
        }

        return result.plus(result.times(taxRate));
    }

    public double getTaxRate() {
        return taxRate;
    }
}

public class NightlyDiscountPhone extends Phone {
    private static final int LATE_NIGHT_HOUR = 22;

    private Money nightlyAmount;

    public NightlyDiscountPhone(Money nightlyAmount, Money regularAmount, Duration seconds, double taxRate) {
        super(regularAmount, seconds, taxRate);
        this.nightlyAmount = nightlyAmount;
    }

    @Override
    public Money calculateFee() {
        // 부모클래스의 calculateFee() 호출
        Money result = super.calculateFee();

        Money nightlyFee = Money.ZERO;
        for(Call call : getCalls()) {
            if (call.getFrom().getHour() >= LATE_NIGHT_HOUR) {
                nightlyFee = nightlyFee.plus(
                        getAmount().minus(nightlyAmount).times(
                                call.getDuration().getSeconds() / getSeconds().getSeconds()));
            }
        }

        return result.minus(nightlyFee.plus(nightlyFee.times(getTaxRate())));
    }
```

⇒ 세급을 계산하는 로직을 추가하기 위해 새로운 중복코드를 만들어야한다.

이것은 Phone과 NightlyDiscountPhone이 너무 강하게 결합되어 있어서 발생하는 문제다.

super 참조로 부모 클래스의 메서드를 호출하면 강하게 결합된다.

최대한 super 호출을 제거해야한다.

## 2. 취약한 기반 클래스 문제

부모클래스의 변경에 의해 자식 클래스가 영향을 받는 현상을 취약한 기반 클래스 문제 라고 부른다.

취약한 기반 클래스 문제는 캡슐화를 약화시키고 결합도를 높인다.

상속은 자식 클래스가 부모 클래스의 구현 세부사항에 의존하도록 만들어서 캡슐화를 약화시킨다.

### 불필요한 인터페이스 상속 문제

![Untitled](Chapter%2010%20%E1%84%89%E1%85%A1%E1%86%BC%E1%84%89%E1%85%A9%E1%86%A8%E1%84%80%E1%85%AA%20%E1%84%8F%E1%85%A9%E1%84%83%E1%85%B3%20%E1%84%8C%E1%85%A2%E1%84%89%E1%85%A1%E1%84%8B%E1%85%AD%E1%86%BC%206157687dce034c5bbe1a371ca1af0dbc/Untitled.png)

![Untitled](Chapter%2010%20%E1%84%89%E1%85%A1%E1%86%BC%E1%84%89%E1%85%A9%E1%86%A8%E1%84%80%E1%85%AA%20%E1%84%8F%E1%85%A9%E1%84%83%E1%85%B3%20%E1%84%8C%E1%85%A2%E1%84%89%E1%85%A1%E1%84%8B%E1%85%AD%E1%86%BC%206157687dce034c5bbe1a371ca1af0dbc/Untitled%201.png)

자바 초기버전에서는 위와같은 상속관계로 인해 문제가 발생한다.

Stack에는 add가 필요없는데, 인터페이스를 상속받아서 Stack의 규칙을 깨버린다.

Properties는 String만 들어가는데 인터페이스에는 Object로 넣을수있어서 의도하지 않은 값이 들어갈수있다.

이 문제는 상속받은 부모 클래스의 메서드가 자식클래스의 내부 구조에 대한 규칙을 깨트릴 수 있다는 경고를 준다.

### 메서드 오버라이딩의 오작용 문제

```java
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    public InstrumentedHashSet() {
    }

    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

		// 이 케이스에서는 오류가 발생한다.
    @Override
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }

		// 이렇게 수정하면 문제가 없지만, 코드의 중복이 발생한다.
    @Override
    public boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c)
            if (add(e))
                modified = true;
        return modified;
    }

    public int getAddCount() {
        return addCount;
    }
}
```

⇒ 자식클래스가 부모 클래스의 메서드를 오버라이딩 할 경우, 부모 클래스가 자신의 메서드를 사용하는 방법에 자식 클래스가 결합 될 수 있다.

상속되기를 원한다면 상속을 위해 클래스를 설계하고 문서화 해야 한다.

→ 내부 구현을 문서화하면 what이 아닌 how를 설명하는 것이고 캡슐화를 위반한다.

→ 설계는 트레이드 오프이기 때문에, 상속은 코드의 재사용을 위해 캡슐화를 희생한다.

    완벽한 캡슐화를 원하면, 코드 재사용을 포기하거나 상속 의외의 방법을 사용해야 한다.

### 부모 클래스와 자식 클래스의 동시 수정 문제

```java
public class Playlist {
    private List<Song> tracks = new ArrayList<>();

    public void append(Song song) {
        getTracks().add(song);
    }

    public List<Song> getTracks() {
        return tracks;
    }
}

public class PersonalPlaylist extends Playlist {
    public void remove(Song song) {
        getTracks().remove(song);
    }
}
```

위 코드에 가수도 관리해야하는 요구사항이 추가되면, 부모클래스와 자식 클래스가 전부 변경되야한다.

```java
public class Playlist {
    private List<Song> tracks = new ArrayList<>();
    private Map<String, String> singers = new HashMap<>();

    public void append(Song song) {
        tracks.add(song);
        singers.put(song.getSinger(), song.getTitle());
    }

    public List<Song> getTracks() {
        return tracks;
    }

    public Map<String, String> getSingers() {
        return singers;
    }
}

public class PersonalPlaylist extends Playlist {
    public void remove(Song song) {
        getTracks().remove(song);
        getSingers().remove(song.getSinger());
    }
}
```

자식 클래스가 부모클래스의 구현에 강하게 결합되기 때문에 이 문제는 피하기 어렵다.

클래스를 상속하면 결합도로 인해 자식 클래스와 부모 클래스의 구현을 영원히 변경하지 않거나, 자식 클래스와 부모 클래스를 동시에 변경하거나 둘중 하나를 선택할 수 밖에 없다.

## 3. Phone 다시 살펴보기

### 추상화에 의존하자

NightDiscountPhone의 가장 큰 문제점은 Phone에 강하게 결합이 되어있기 때문에 Phone이 수정될 경우 같이 수정 될 가능성이 높다는 것이다. 가장 일반적인 해결 방법은 **부모 클래스와 자식클래스 모두 추상화에 의존**하도록 수정해야 한다.

<aside>
🔖 개인적인 코드 중복 제거를 위해 상속을 도입할 때 따르는 두가지 원칙이 있다.

**1. 두 메서드가 유사하게 보인다면 차이점을 메서드로 추출하라.** 
메서드 추출을 통해 두 메서드를 동일한 형태로 보이도록 만들수 있다.

**2. 부모 클래스의 코드를 하위로 내리지 말고 자식 클래스의 코드를 상위로 올려라.**
부모 클래스의 구체적인 메서드를 자식 클래스로 내리는 것보다 자식 클래스의 추상적인 메서드를 부모 클래스로 올리는 것이 재사용성과 응집도 측면에서 더 뛰어난 결과를 얻을 수 있다.

</aside>

1. 이전코드

```java
public class Phone {
	private Money amount;
	private Duration seconds;
	private List<Call> calls = new ArrayList<>();

	public Phone(Money amount, Duration seconds){
		this.amount = amount;
		this.seconds = seconds;
	}

	public Money calculateFee(){
		Money result = Money.ZERO;

		for(Call call: calls){
			result = result.plus(
				amount.times(
					call.getDuration().getSeconds() / seconds.getSeconds()
				)
			);
		}
	}
}
```

```java
public class NightDiscountPhone {
	private stataic final int LATE_NIGHT_HOUR = 22;

	private Money nightlyAmount;
	private Money regularAmount;
	private Duration seconds;
	private List<Call> calls = new ArrayList<>();

	public NightDiscountPhone(Money nightlyAmount, Money regularAmount, Duration seconds){
		this.nightlyAmount = nightlyAmount;
		this.regularAmount = regularAmount;
		this.seconds = seconds;
	}

	public Money calculateFee(){
		Money result = Money.ZERO;

		for(Call call : calls){
			if(call.getFrom().getHour() >= LATE_NIGHT_HOUR){
				result = result.plus(
					nightlyAmount.time(call.getDuratioon().getSeconds() / seconds.getSecond()));
				)
			}else{
				result = result.plus(
					regularAmount.time(call.getDuratioon().getSeconds() / seconds.getSecond()));
				}
			}
		}

		return reulst;
	}
}
```

위 수정 전 코드를 수정할때 먼저 할 일은 두 클래스의 메서드에서 다른 부분을 별도의 메서드로 추출하는 것이다.

이 경우에는 calculateFee의 for문 안에 구현된 요금 계산 로직이 서로 다르다는 것을 알 수 있다. 이 부분을 동일한 이름을 가진 메서드로 추출하자.

1. 추출 코드

```java
public class Phone {
	...

	public Money calcurateFee(){
		Money result = Money.ZERO;

		for(Call call : calls){
			result = result.plus(calculateCallFee(call)); // 추출한 메서드
		}
	
		return result;
	}

	// 추출한 메서드
	private Money calculateCallFee(Call call){
		return amount.times(call.getDuration().getSeconds() / seconds.getSeconds());
	}
}
```

```java
public class NightlyDiscountPhone {
	...

	public Money calculateFee(){
		Money result = Money.ZERO;

		for(Call call : calls){
			result = result.plus(calculateCallFee(call));
		}

		return result;
	}

	private Money calculateCallFee(Call call){
		if(call.getForm).getHour() >= LATE_NIGHT_HOUR) {
			return nightlyAmount.times(call.getDuratioon().getSeconds() / seconds().getSeconds());
		} else {
			return regularAmount.times(call.getDuratioon().getSeconds() / seconds().getSeconds());
		}
	}
}
```

1. 중복 코드를 부모클래스로 올리기

부모 클래스와 자식클래스가 추상화에 의존하도록 하는것이 목표이기 때문에 추상클래스로 구현하는것이 적합 할것이다.

```java
public abstract class AbstractPhone{}
public class Phone extends AbstractPhone{ ... }
public class NightlyDiscountPhone extends AbstractPhone { ... }
```

```java
public abstract class AbstractPhone {

	// calculateFee를 옮기면 calls가 없기 때문에 오류가 나 추가를 해준다.
	private List<Call> calls = new ArrayList<>();

	public Money calculateFee(){
		Money result = Money.ZERO;

		for(Call call : calls){
			result = result.plus(calculateCallFee(call));
		}

		return result;
	}

	/**
   * calculateCallFee 의 내부 구현 내용이 다르기 때문에 메서드 구현은 그대로 기존 클래스에 두고
	 * 시그니처만 부모 클래스로 이동시켜야 한다.
	 * 자식클래스에서 오버라이딩을 할 수 있도록 protected로 선언을 한다.
   * protected란 - 타 클래스에서는 직접적인 호출은 막지만 자식 클래스에서는 public처럼 사용 가능
	**/
	abstract protected Money calculateCallFee(Call call);
}
```

```java
public class Phone extends AbstractPhone{
	private Money amount;
	private Duration seconds;

	public Phone(Money amount, Duration seconds){
		this.amount = amount;
		this.seconds = seconds;
	}

	@override
	protected Money calculateCallFee(Call call){
		return amount.times(call.getDuration().getSeconds() / seconds.getSeconds());
	}
}
```

```java
public class NightlyDiscountPhone extends AbstractPhone{
	private static final int LATE_NIGHT_HOUR = 22;
	
	private Money nightlyAmount;
	private Money regularAmount;
	private Duration seconds;

	public NightlyDiscountPhone(Money nightlyAmount, Money regularAmount, Duration seconds){
		this.nightlyAmount = nightlyAmount;
		this.regularAmount = regularAmount;
		this.seconds = seconds;
	}

	@override
	protected Money calculateCallFee(Call call){
		if(call.getFrom().getHour() >= LATE_NIGHT_HOUR){
			return nightlyAmount.times(call.getDuration().getSeconds() / seconds.getSeconds());
		}
		return regularAmount.times(call.getDuration().getSeconds() / seconds.getSeconds());
	}
}
```

**추상화가 핵심이다 - (추상화에 의존하면 얻는 장점)**

공통코드를 이동시킨 후 각 클래스는 서로 다른 변경의 이유를 가진다.

자식클래스인 Phone과 NightlyDiscountPhone은 부모클래스의 구체적인 구현에 의존하지 않는다.

그래서 시그니처가 변경되지 않는 한 부모클래스의 내부 구현이 변경 되더라도 자식클래스는 영향을 받지 않는다. → 낮은 결합도 유지

**의도를 드러내는 이름 선택하기**

위에 이름들은 명시적으로 전달하지 못한다.

<aside>
🔖 AbstractPhone → Phone
Phone → RegularPhone

</aside>

### 세금 추가하기

정말 수정이 쉬울까? 세금을 추가해본다.

```java
public abstract class Phone {
	private double taxRate;
	private List<Call> calls = new ArrayList<>();

	public Phone(double taxRate){
		this.taxRate = taxRate;
	}

	public Money calculateFee(){
		Money result = Money.ZERO;

		for(Call call : calls){
			result = result.plus(calculateCallFee(call));
		}

		return result.plus(result.times(taxRate));
	}

	protected abstract Money calculateCallFee(Call call)
}
```

```java
public class RegularPhone extends Phone {
	...

	public RegularPhone(Money amount, Duration secdons. double taxRate){
		super(taxRate);
		this.amount = amount;
		this.seconds = seconds;
	}
	...
}
```

```java
public class NightlyDiscountPhone extends Phone {
	...

	public NightlyDiscountPhone(Money nightlyAmount, Money regularAmoumt,
			Duration seconds, double taxRate){
		super(taxRate);
		this.nightlyAmount = nightlyAmount;
		this.regularAmoumt = regularAmoumt;
		this.seconds = seconds;
	}
}
```

이렇게 상속으로 인한 클래스 사이의 결합은 피할 수 없다. 상속은 어떠한 방식으로든 부모클래스와 자식클래스를 결합시킨다.

**우리가 원하는 것은 행동을 변경하기 위해 인스턴스 변수를 추가하더라도 상속 계층 전체에 걸쳐 부작용이 퍼지지 않게 막는것이다.**

### 차이에 의한 프로그래밍

위처럼 기존 코드와 다른 부분만을 추가함으로써 애플리케이션의 기능을 확장하는 방법을 **차이에 의한 프로그래밍**이라 부른다. 상속을 이용하면 이미 존재하는 클래스의 코드를 쉽게 재사용할 수 있기 때문에 애플리케이션의 **점진적인 정의**가 가능해진다.

중복코드는 악의 근원이다. 중복코드를 제거하기 위해 최대한 코드를 재사용 해야한다.

코드의 재사용 방법 중 가장 유명한 방법이 상속이다.

하지만 상속을 잘못 사용할 경우 오는 피해도 크다. 오용과 남용은 애플리케이션의 이해와 확장을 어렵게 하기 때문에 정말 필요한 경우에만 상속을 사용하라

더 좋은 방법이 있다. 바로 **합성**이다. 다음 챕터에서 다룰 예정이다.