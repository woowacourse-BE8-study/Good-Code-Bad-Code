## 주제 1. 불변 객체와 쓰기 시 복사 (Copy-on-write)

> **연관 개념:** 7장(불변 객체 생성 및 변경 처리), 6장(입력 매개변수 수정 금지)
>

단순히 `final` 필드를 사용하는 것을 넘어, 상태 변경이 일어날 때 기존 객체를 어떻게 다루는지(수정)에 대해 다룹니다. 현재 제 프로젝트에는 Setter가 존재하지 않으며, '쓰기 시 복사' 패턴을 구현하고
있고 이 사례를 보여드리겠습니다.

### 1. 도메인이 전부 불변 객체

- 모든 필드가 `final`이며 Setter가 없어 생성 후 상태 변경이 불가능합니다.
- 생성자 내부에서 `validate(...)`를 호출하여 "객체가 존재한다 = 항상 유효하다"는 불변식을 보장합니다.

```java
// domain/Reservation.java
private final Long id;
private final String name;
private final LocalDate date;
private final ReservationTime time;
private final Theme theme;

public Reservation(Long id, String name, LocalDate date, ReservationTime time, Theme theme) {
    validate(name, date, time, theme);
    this.id = id;
    this.name = name;
    this.date = date;
    this.time = time;
    this.theme = theme;
}

리퀘스트(사용자가 원천 입력값) ->컨트롤러 ->서비스 ->

도메인 생성(예외) :400

에러 create()

DB(소스의 원천 내부) ->서비스 ->

도메인 생성(예외) :500

에러 load()
```

### 2. 변경 = 새 객체 반환 (Copy-on-write)

- 기존 `reservation` 객체는 건드리지 않고, 변경된 데이터만 교체한 새로운 객체를 생성합니다.
- 만약 `setDate()` 같은 방식을 썼다면, 이 객체를 참조하는 다른 코드들에 부수 효과(Side-effect)가 발생했을 것입니다.

```java
// service/UserReservationService.java
Reservation updated = new Reservation(
                reservation.getId(),     // 기존 값 유지
                reservation.getName(),   // 기존 값 유지
                command.date(),          // 바뀐 값
                newTime,                 // 바뀐 값
                reservation.getTheme()   // 기존 값 유지
        );      
return ReservationResult.

from(reservationRepository.update(updated));
```

### 3. 저장시에도 입력 매개변수를 보호

- DB가 자동 생성한 `id`를 입력 객체에 `setId()`로 주입하지 않고, 새로운 객체를 만들어 반환합니다.
- 호출자가 넘긴 원본 객체(`id=null`)는 그대로 유지됩니다.

```java
// repository/JdbcReservationRepository.java
Long id = keyHolder.getKey().longValue();
return new

Reservation(id, reservation.getName(),reservation.

getDate(),
        reservation.

getTime(),reservation.

getTheme());
```

---

## 주제 2. 구현 세부 정보 유출 막기 (계층별 예외 추상화)

> **연관 개념:** 8장(내부 구현 방식 노출 금지, 자체 예외 사용), 6장(오류는 매직값 대신 옵셔널이나 예외로 전달)
>

단일 예외가 3개의 계층(Repository → Service → Controller)을 통과하며 추상화 수준이 어떻게 변하는지 추적합니다. "왜 이렇게 예외 클래스를 여러 개 만들었을까?"에 대한 이유를
살펴봅니다.

### 1. Repository 계층: JDBC 세부사항 격리

- `EmptyResultDataAccessException`은 Spring JDBC만의 구체적인 기술 디테일입니다.
- 이 예외를 잡아 `Optional.empty()`로 변환하여 서비스 계층으로 넘깁니다. (JPA로 기술 스택이 바뀌어도 서비스 계층은 영향받지 않음)

```java
// repository/JdbcReservationRepository.java
try{
Reservation reservation = jdbcTemplate.queryForObject(sql, RESERVATION_ROW_MAPPER, id);
    return Optional.

ofNullable(reservation);
}catch(
EmptyResultDataAccessException e){
        return Optional.

empty();
}
```

### 2. Service 계층: 도메인 언어로 예외 변환

- `Optional`을 받아 도메인 관점의 `ReservationNotFoundException`으로 변환합니다.
- 서비스 로직은 자신이 웹 환경(HTTP)에서 호출되는지, 배치(Batch)에서 호출되는지 알 필요가 없습니다.

```java
// service/UserReservationService.java
private Reservation findReservation(Long id) {
    return reservationRepository.findById(id)
            .orElseThrow(() -> new ReservationNotFoundException(
                    "존재하지 않는 예약입니다: reservationId=" + id));
}
```

### 3. Controller 계층

- 하위 예외들(`ReservationNotFoundException`, `ThemeNotFoundException` 등)이 공통 부모인 `ResourceNotFoundException`을 상속받도록 설계되었습니다.
- Controller에서 부모 타입 하나만 잡아 핸들러 폭발을 막고, 프로젝트 전체에서 **이곳에서만 HTTP 상태 코드를 다루도록 격리**했습니다.

```java
// exception/GlobalExceptionHandler.java
@ExceptionHandler(ResourceNotFoundException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
public ErrorResponse handleNotFound(ResourceNotFoundException e) {
    return new ErrorResponse(e.getMessage());
}
```

---

## 주제 3. 진실의 원천

> **연관 개념:** 7장(파생 데이터와 로직은 한 곳에), 8장(디미터의 법칙, 관련 데이터의 캡슐화)
>

리팩토링 사례입니다. `Reservation` 객체가 `date`와 `time`을 분리해서 가지고 있다 보니, "이 예약이 과거인가?"를 묻는 파생 로직이 서비스 곳곳에 흩어져 있는 전형적인 빈약한 도메인 모델의
한계를 보여줍니다.

### Before: 흩어진 진실의 원천과 디미터 법칙 위반

`UserReservationService`와 `ReservationTimeService` 두 곳에서 날짜를 조립하고 과거인지 판정하는 로직이 중복되어 있습니다. 특히 `cancel` 로직은 예약 객체를 가지고
있으면서도 객체 내부를 깊숙이 파고듭니다.

```java
// UserReservationService.java(디미터 법칙 위반)
validateNotPast(
        reservation.getDate(),
        reservation.

getTime().

getStartAt(),   // 객체의 속살을 외부에서 꺼내서 조립함
        "과거 예약은 취소할 수 없습니다"
                );
```

### After: 도메인이 스스로 대답하게 만들기 (Rich Domain)

`Reservation` 도메인 내부에 행위를 추가하여 관련된 데이터를 캡슐화합니다.

```java
// domain/Reservation.java
public LocalDateTime startDateTime() {
    return LocalDateTime.of(date, time.getStartAt());
}

public boolean isPast(LocalDateTime now) {
    return startDateTime().isBefore(now);
}
```

서비스 로직은 더 이상 `getTime().getStartAt()`을 호출하지 않으며, "예약이 과거인가?"에 대한 진실의 원천은 `Reservation` 도메인 한 곳으로 모이게 됩니다.

```java
// service/UserReservationService.java (cancel 로직 개선)
public void cancel(Long id, String name) {
    Reservation reservation = findReservation(id);
    validateOwner(reservation, name);

    // 도메인에게 직접 물어본다 (디미터 법칙 준수)
    if (reservation.isPast(LocalDateTime.now())) {
        throw new PastReservationException("과거 예약은 취소할 수 없습니다");
    }
    reservationRepository.deleteById(id);
}
```
