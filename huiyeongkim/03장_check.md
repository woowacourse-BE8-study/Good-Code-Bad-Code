# 체크

```plaintext
// p.73
체크
코드 계약 조건을 확인하기 위한 일반적인 방법은 체크(check)를 사용하는 것이다.
이것은 코드 계약이 준수되었는지 확인하기 위한 추가적인 로직이며,
준수되지 않을 경우 오류를 발생시켜 실패를 명백하게 드러낸다.
```

- 체크: 코드가 기대하는 조건이 지켜졌는지 확인하는 로직
- 신속한 실패: 문제가 생긴 지점을 최대한 빨리 발견하고 즉시 멈추는 설계 원칙

Reservation 객체를 생성할 때 반드시 필요한 값이 누락되어도 객체가 만들어질 수 있는 문제가 있었다.
예약자명뿐만 아니라 예약 시간도 예약 객체가 성립하기 위해 필요한 값이므로, 객체 생성 시점에 함께 검증하도록 수정했다.


## Before

기존 코드에서는 예약자명만 검증하고 있었다.

```java
public class Reservation {

    private Reservation(Long id, String name, LocalDate date, ReservationTime time) {
        this.id = id;
        this.name = name;
        this.date = date;
        this.time = time;
    }

    public static Reservation create(String name, LocalDate date, ReservationTime time) {
        validate(name);
        return new Reservation(null, name, date, time);
    }

    public static Reservation withId(Long id, String name, LocalDate date, ReservationTime time) {
        validate(name);
        return new Reservation(id, name, date, time);
    }
}
```

이 구조에서는 time이 null이어도 Reservation 객체가 생성될 수 있다.
하지만 예약 시간은 예약이 성립하기 위한 필수 값이다.

따라서 time이 없는 예약 객체가 만들어지면, 객체 생성 시점이 아니라 이후 로직에서 문제가 드러날 수 있다.

예를 들어 예약을 저장하거나 조회 응답을 만들 때 time.getId() 또는 time.getStartAt()을 호출하면 NullPointerException이 발생할 수 있다.
이 경우 문제의 원인이 객체 생성 시점에 있었음에도, 실제 오류는 전혀 다른 위치에서 발생하게 된다.


## After

예약 객체 생성 시점에 예약자명과 예약 시간을 모두 검증하도록 수정했다.

```java
public class Reservation {

    public static Reservation create(String name, LocalDate date, ReservationTime time) {
        validateName(name);
        validateTime(time);
        return new Reservation(null, name, date, time);
    }

    public static Reservation withId(Long id, String name, LocalDate date, ReservationTime time) {
        validateName(name);
        validateTime(time);
        return new Reservation(id, name, date, time);
    }

    private static void validateName(String name) {
        if (name == null || name.isEmpty()) {
            throw new IllegalArgumentException("예약자명이 없습니다.");
        }
    }

    private static void validateTime(ReservationTime time) {
        if (time == null) {
            throw new IllegalArgumentException("예약시간이 없습니다.");
        }
    }
}
```

수정 후에는 Reservation 객체가 생성되기 전에 필수 값이 모두 존재하는지 확인한다.
예약자명이 없거나 예약 시간이 없으면 즉시 IllegalArgumentException을 발생시킨다.

이를 통해 잘못된 상태의 객체가 만들어지는 것을 막을 수 있다.
또한 문제가 발생한 위치가 객체 생성 시점임을 명확히 알 수 있어 디버깅도 쉬워진다.


## 왜 체크를 추가했을까?

Reservation은 예약이라는 도메인 객체이다.
예약은 최소한 예약자명, 예약 날짜, 예약 시간을 가져야 의미 있는 객체가 된다.

그런데 예약 시간이 null인 상태로 객체가 생성되면, 도메인 규칙을 만족하지 않는 객체가 애플리케이션 내부에 존재하게 된다.
이런 객체는 당장 문제가 없어 보이더라도 이후 로직에서 예상치 못한 오류를 만들 수 있다.

따라서 객체를 생성하는 시점에 필수 조건을 확인하는 것이 더 안전하다.

이것이 체크의 목적과 연결된다.
체크는 코드가 기대하는 계약 조건을 확인하고, 조건이 깨졌을 때 즉시 실패하게 만든다.
