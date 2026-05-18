## 추상화와 관심사 분리

### 추상화 계층의 예시 (컨트롤러 - 서비스 - 레포지토리 )

```java

@RestController
@RequestMapping("/user/themes")
public class UserThemeController {

    private final ThemeService themeService;
    private final ReservationService reservationService;

    public UserThemeController(ThemeService themeService, ReservationService reservationService) {
        this.themeService = themeService;
        this.reservationService = reservationService;
    }

    @GetMapping("/popular")
    public List<PopularThemeResponse> popular() {
        return themeService.findPopular();
    }
```

```java

@Service
public class ThemeService {

    private final ThemeRepository themeRepository;

    public ThemeService(ThemeRepository themeRepository) {
        this.themeRepository = themeRepository;
    }

    public List<PopularThemeResponse> findPopular() {
        LocalDate endDate = LocalDate.now();
        LocalDate startDate = endDate.minusDays(7);
        return themeRepository.findPopular(startDate, endDate, 10).stream()
                .map(PopularThemeResponse::from)
                .toList();
    }
```

책에서의 2장 “하위 세부 구현 숨기기”의 예 입니다.

컨트롤러에서 아는 것은 단지 서비스를 통해 인기있는 테마 목록을 가져와 반환한다는 것 입니다.

서비스에서는 이보다 구체화하여 어느 시기부터 언제 까지 몇 순위의 리스트를 반환할지 결정합니다.

이때 데이터베이스는 뭘 쓰는지 어떤 SQL을 작성하는지는 모릅니다. (레포지토리도 추상화)

---

### DTO 에서의 관심사 분리

```java
//엔티티
public class Theme {

    private final Long id;
    private final String name;
    private final String description;
    private final String thumbnailUrl;

    // popular 기능만을 위한 필드 -> 도메인 오염 
    private final Long reservationCount;

```

```java
// 서비스 <-> 레포 계층 전송 용도의 DTO
public class PopularThemeResult {

    private final Theme theme;
    private final long reservationCount;
    

---

    // 서비스 -> 컨트롤러(외부) 용도 DTO  
    public class PopularThemeResponse {

        private final Long id;
        private final String name;
        private final String description;
        private final String thumbnailUrl;
        private final long reservationCount;

---

        // 추후에 다른 집계나 필드가 생겨도 기존 Theme 도메인을 건드리지 않고
// PopularThemeResult에 추가하면 끝. 
        @GetMapping("/popular")
        public List<PopularThemeResponse> popular() {
            return themeService.findPopular();
        }

```

`Theme` 도메인 엔티티는 DB의 테이블과 맵핑되는 책임과 그에 해당하는 비즈니스 규칙을 담당합니다.
이 엔티티는 Theme 라는 개념에 필요한 최소한의 정보(본질적인 정보)만을 담아야 합니다.
하지만 `인기 테마의 목록과 예약된 건수를 조회한다.` 라는 기능이 추가되었을 때 이를 Theme로 전달하는게 맞을까요?
만약 예약된 건수(reservationCount)를 표현하기 위해 Theme 엔티티에 이를 추가하면, 기존에 정의한 '최소한의 정보만을 담는다' 라는 규칙을 위반하게 되어
이는 도메인 오염으로 이어집니다. 도메인이 오염될 경우 기존에 이를 사용하던 모든 코드에 이 변경 사항이 전파될 수 있고, 또는 이를 전달받는 사용자 입장에서 혼선이 생길 수 있습니다.
따라서 별도의 전달 객체 (PopularThemeResult)를 작성하여 '예약 건수 조회'라는 관심사를 담당하도록 구분할 수 있습니다. 
