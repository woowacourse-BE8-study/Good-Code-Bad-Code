## 방탈출 미션에서 경험한 강한 계약과 빠른 실패 케이스

```java
public class Reservation {
    private static final int MAX_NAME_LENGTH = 30;

    private final Long id;
    private final String name;
    private final LocalDate date; //강한 계약
    private final ReservationTime time; //강한 계약
    private final Theme theme; //강한 계약

    public Reservation(Long id, String name, LocalDate date, ReservationTime time, Theme theme) {
        validate(name, date, time, theme);
        this.id = id;
        this.name = name;
        this.date = date;
        this.time = time;
        this.theme = theme;
    }

    private static void validate(String name, LocalDate date, ReservationTime time, Theme theme) {
        validateName(name);
        validateDate(date);
        validateTime(time);
        validateTheme(theme);
    }
```

```java
public class ReservationTime {
    private final Long id;
    private final LocalTime startAt;

    public ReservationTime(Long id, LocalTime startAt) {
        validate(startAt);
        this.id = id;
        this.startAt = startAt;
    }

    private static void validate(LocalTime startAt) {
        validateStartAt(startAt);
    }

    private static void validateStartAt(LocalTime startAt) {
        if (startAt == null) {
            throw new IllegalArgumentException("예약 시간은 비어 있을 수 없습니다.");
        }
    }
```

- **강한 타입의 계약 :**`Reservation`객체를 만들 때,`Long timeId`나`Long themeId`같은 원시 타입을 받지 않고,`ReservationTime`과`Theme`이라는 명확한 객체
  타입을 요구합니다. 개발자가 실수로 파라미터 순서를 바꾸거나 이상한 ID를 넣는 것을**컴파일 타임**에 막아줍니다
- **빠른 실패 :** 각 도메인은 생성자에서`validate()`메서드를 호출합니다. 이름이 너무 길거나 null인 경우 나중에 DB 에러가 날 때까지 기다리지 않고, 객체 생성 시점에 즉시
  `IllegalArgumentException`을 던집니다. 이는 잘못된 상태의 객체가 돌아다니는 것을 막아 1장에서 말하는 오용을 방지하는 설계
    - **코드 품질의 6가지 핵심**
        1. **가독성이 좋아야 한다.**
        2. **놀람을 최소화해야 한다 (예측 가능해야 함).**
        3. **오용하기 어렵게 만들어야 한다.**
        4. **모듈화가 잘 되어 있어야 한다.**
        5. **재사용 가능해야 한다.**
        6. **테스트하기 쉬워야 한다.**
