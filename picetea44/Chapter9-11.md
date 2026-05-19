## 주제 1. 테스트는 함수 목록이 아니라 '실행 가능한 명세서'다 (11장)

**📖 책의 조언:** "함수당 테스트 하나가 아니라 동작마다 테스트를 만들고, private은 public API를 통해 검증하라."

> **나의 주장**
"동작 단위로 쪼개는 건 테스트를 '늘리는 비용'이 아니라 '명세를 적는 일'입니다. 그리고 `validateNotPast` 같은 private 메서드를 따로 테스트하지 않은 건, 테스트를 빠뜨린 게 아니라 의도된
> 설계입니다."
>

### 근거 1. `@DisplayName`을 위에서 아래로 읽으면 그것이 곧 요구사항 문서다.

`UserReservationServiceTest`의 `create` 관련 테스트 3개를 이어 읽어보면 명세가 명확해집니다.

- "미래 시점에 예약하면 정상적으로 생성된다" (Line 65)
- "과거 날짜로 예약하면 PastReservationException이 발생한다" (Line 79)
- "존재하지 않는 timeId로 예약하면 ReservationTimeNotFoundException이 발생한다" (Line 96)

함수 하나에 테스트 하나만 만들었다면 정상 케이스만 남고, 오류 시나리오는 코드 어디에도 문서화되지 않았을 것입니다. 즉, "테스트 3개 = 비용 3배"가 아니라 "명세 항목 3개"를 의미합니다.

### 근거 2. private을 직접 테스트하고 싶어지는 순간은 '클래스가 너무 크다'는 신호다.

`validateNotPast`, `validateOwner`, `findReservation`은 전부 private이며 직접적인 테스트가 없습니다. 대신 '과거_날짜_예약시_예외' 테스트가 public인
`create()`를 통해 내부의 `validateNotPast`를 자연스럽게 검증합니다.

저는 이것을 일부러 이렇게 두었습니다. 현재는 public 메서드 4~5개 만으로 내부의 모든 분기에 도달할 수 있기 때문입니다. 만약 public API만으로 특정 분기에 도달할 수 없다면, 그때는 private을
public으로 열거나 테스트할 것이 아니라 **별도의 클래스로 분리해야 한다는 신호**입니다.

**결론 :** "테스트 개수는 함수 개수가 아니라 명세 항목 개수를 따라가야 합니다. 그리고 private을 테스트하고 싶어지면 테스트 코드를 억지로 바꿀 것이 아니라 프로덕션 코드를 쪼개야 합니다. 우리 서비스는
아직 그 시점이 아니기 때문에 private 테스트를 두지 않았습니다."

## 주제 2. 매개변수화 테스트의 묶음 기준은 '값'이 아니라 '동작'이다 (11장)

**책의 조언:** "같은 동작을 입력만 바꿔 반복하면 매개변수화 테스트를 써라."

> **나의 주장**
"`@ValueSource`에 값을 넣기 전에 항상 '이 값들이 깨지면 같은 이유로 같은 메시지가 나오는가?'를 묻습니다. 매개변수화는 단순한 중복 제거 도구가 아니라, '하나의 동작'을 표현하는 도구입니다."
>

### ❌ Before: 글자 단위로 똑같은 복사본

값만 `0L`, `-1L`로 다르고 본문이 완전히 동일한 테스트가 두 개 존재합니다. (사실상 같은 동작)

```java

@Test
@DisplayName("시간 ID가 0이면 위반이 발생한다")
void 시간_ID가_0이면_위반이_발생한다() {
    ReservationRequest request = new ReservationRequest(VALID_NAME, VALID_DATE, 0L, VALID_THEME_ID);
    Set<ConstraintViolation<ReservationRequest>> violations = validator.validate(request);
    assertThat(violations).extracting(ConstraintViolation::getMessage).contains("시간 ID는 1 이상이여야 합니다");
}

@Test
@DisplayName("시간 ID가 음수이면 위반이 발생한다")
void 시간_ID가_음수이면_위반이_발생한다() {
    ReservationRequest request = new ReservationRequest(VALID_NAME, VALID_DATE, -1L, VALID_THEME_ID);

    Set<ConstraintViolation<ReservationRequest>> violations = validator.validate(request);

    assertThat(violations).extracting(ConstraintViolation::getMessage)
            .contains("시간 ID는 1 이상이여야 합니다");
}
```

### ✅ After: 동작 단위로 묶기

```java

@ParameterizedTest
@ValueSource(longs = {0L, -1L})
@DisplayName("시간 ID가 1 미만이면 위반이 발생한다")
void 시간_ID가_1_미만이면_위반이_발생한다(long invalidTimeId) {
    ReservationRequest request = new ReservationRequest(VALID_NAME, VALID_DATE, invalidTimeId, VALID_THEME_ID);
    Set<ConstraintViolation<ReservationRequest>> violations = validator.validate(request);
    assertThat(violations).extracting(ConstraintViolation::getMessage).contains("시간 ID는 1 이상이여야 합니다");
}
```

### 핵심: 왜 `null`은 여기에 묶지 않았을까?

`timeId`가 잘못된 값에는 `0`, `-1` 말고 `null`도 있습니다. "잘못된 timeId"라는 이유로 이 셋을 하나로 묶고 싶은 충동이 들 수 있지만, 저는 단호히 분리합니다.

- `0`, `1` 입력 시: `@Positive` 위반 → "시간 ID는 1 이상이여야 합니다"
- `null` 입력 시: `@NotNull` 위반 → "시간 ID는 필수입니다"

이 둘은 명백히 **서로 다른 동작**입니다. `{null, 0, -1}`을 하나의 `@ValueSource`에 넣으면 코드 줄 수는 줄어들겠지만, "한 번에 하나의 동작"을 테스트한다는 원칙이 깨집니다.

**결론 :** "매개변수화를 단순히 '비슷한 테스트 합치기' 용도로 쓰면 안 됩니다. 묶음 하나가 곧 동작 하나여야 합니다. 묶었을 때 기대 결과나 에러 메시지가 달라진다면, 그건 묶지 말라는 강력한 신호입니다."

## 주제 3. [핵심] verify는 두 종류다. 하나는 동작 검증, 하나는 구현 박제다 (10장)

**책의 조언:** "Mock·Stub 과사용은 구현 세부사항과 얽혀 리팩터링을 막는다 — Fake나 실제 의존성을 선호하라."

> **나의 주장 (발표 결론)**
"저는 `verify`를 두 갈래로 나눕니다. ① '저장됐다 / 삭제 안 됐다' 같은 **부수효과 검증**은 남깁니다. ② 'A가 B를 호출했다' 같은 협력 검증과 `verifyNoMoreInteractions`는
> 구현 선택을 박제하는 것이므로 버립니다. 그리고 정상 케이스엔 반드시 결과 검증(`assertThat`)을 넣어야 합니다."
>

### 근거 1. 남겨야 하는 verify (부수효과·부정 검증)

"과거 날짜면 저장이 일어나지 않는다"는 사용자가 관측해야 하는 결과이자 동작입니다. 구현 방식을 어떻게 바꾸든 이 사실은 참이어야 하므로 유지하는 것이 맞습니다.

```java
// UserReservationServiceTest.java:90
verify(reservationService, never()).

create(any());
```

### 근거 2. 버려야 하는 verify (협력 박제)

아래 테스트는 명백한 결함을 가지고 있습니다.

```java
void 미래_시점_예약은_정상_생성된다() {
    given(reservationTimeRepository.findById(1L)).willReturn(Optional.of(VALID_TIME));

    assertDoesNotThrow(() -> userReservationService.create(command));   // 결과를 안 본다

    verify(reservationService, times(1)).create(command);               // 🚨 위임을 박제
    verifyNoMoreInteractions(reservationTimeRepository, reservationService); // 🚨 구조를 박제
}
```

1. **동작이 아닌 구현의 박제:** `verify(reservationService).create(command)`는 "Service가 AdminService에 위임한다"는 제 설계 선택일 뿐입니다. 리팩터링을
   통해 위임을 걷어내고 로직을 합치면, 외부 동작은 똑같은데 이 테스트는 깨집니다. 책이 경고한 "리팩터링에 깨지는 테스트"의 전형입니다.
2. **구조의 박제:** `verifyNoMoreInteractions`는 호출 목록 전체를 얼려버립니다. 로깅이나 검증용 메서드 호출 한 줄만 추가해도 테스트가 실패합니다.
3. **결과 검증의 부재:** Mock에 대한 Stub이 없어 `create`가 `null`을 반환해도 이 테스트는 초록불을 띄웁니다.

### 근거 3. 우리 프로젝트 안에 이미 존재하는 '맞는 예'

같은 프로젝트 안에서도 `update` 테스트는 `verify` 대신 결과 상태를 명확히 검증하고 있습니다. 정상 케이스는 모두 이런 방식으로 통일해야 합니다.

```java
// 본인_예약을_정상적으로_변경한다(:202-205)
ReservationResult result = userReservationService.update(command);

assertThat(result.date()).

isEqualTo(ANOTHER_FUTURE_DATE);

assertThat(result.time().

id()).

isEqualTo(2L);
```

### 근거 4. 더 나아간 방향성: Mock에서 Fake로

단위 테스트에서도 `FakeReservationRepository`(인메모리 Map 구현체 등)를 사용하면, 무엇이 호출됐는지가 아니라 **실제로 어떤 결과가 일어났는지** 검증할 수 있습니다.

```java
// ❌ Before: 무엇이 호출되었는가? (구현 검증)
verify(reservationRepository, times(1)).

deleteById(1L);

verifyNoMoreInteractions(reservationRepository);

// ✅ After: 실제로 무엇이 일어났는가? (상태/동작 검증)
assertThat(fakeRepository.findById(reservation.getId())).

isEmpty();
```

**결론 :** "모든 `verify`를 빼자는 것이 아닙니다. '저장되지 않음'과 같은 부수효과 검증은 동작이므로 남겨야 합니다. 하지만 '누가 누구를 호출했나'를 묻는 검증과
`verifyNoMoreInteractions`는 우리가 만든 내부 구조를 옴짝달싹 못 하게 박제합니다. 그 자리에 결과 상태에 대한 `assertThat`을 넣고, 장기적으로는 Mock을 Fake로 교체해 나가는
것이 책이 말하는 유연한 테스트의 방향이라고 생각합니다."
