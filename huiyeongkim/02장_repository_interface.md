# Repository 인터페이스 추상화
```plaintext
// p.48
하나의 추상화 계층에 대해 두 가지 이상의 다른 방식으로 구현을 하거나 
향후 다르게 구현할 것으로 예상되는 경우 인터페이스를 정의하는 것이 좋다.
```

[roomescape-admin](https://github.com/Huiyeongkim/spring-roomescape-admin) 코드에서 Repository 계층에 인터페이스 추상화를 도입했다.


## Before

기존에는 DAO를 구현 클래스 하나로만 정의했다.

```java
@Repository
public class ReservationDao {

    private final NamedParameterJdbcTemplate template;

    public ReservationDaoImpl(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
    }

    public Reservation save(Reservation reservation) {
        String sql = "INSERT INTO reservation (name, date, time_id)" +
                " VALUES (:name, :date, :time_id)";
        SqlParameterSource param = new MapSqlParameterSource()
                .addValue("name", reservation.getName())
                .addValue("date", reservation.getDate())
                .addValue("time_id", reservation.getTime().getId());

        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(sql, param, keyHolder);

        Long id = keyHolder.getKey().longValue();
        return Reservation.withId(id, reservation.getName(), reservation.getDate(), reservation.getTime());
    }
}
```

이 구조에서는 서비스가 구체 구현 클래스에 직접 의존하게 된다.
현재는 JdbcTemplate 기반 구현만 존재하지만, 이후 데이터 접근 방식이 변경될 경우 서비스 코드도 함께 영향을 받을 수 있다.

- JdbcTemplate에서 JPA로 변경
- 테스트용 InMemory Repository 추가
- DB 종류나 쿼리 구현 방식 변경
- 캐싱 Repository, 로그를 남기는 Repository 등 부가 기능 추가

이처럼 데이터 접근 계층은 구현 방식이 달라질 가능성이 있는 영역이라고 판단했다.

## After

그래서 ReservationDao를 인터페이스로 분리하고, 실제 구현체는 ReservationDaoImpl로 분리했다.

```java
public interface ReservationDao {
    Reservation save(Reservation reservation);

    List<Reservation> findAll();

    Optional<Reservation> findById(Long id);

    void deleteById(Long id);

    boolean existsByTimeId(Long timeId);
}
@Repository
public class ReservationDaoImpl implements ReservationDao {

    private final NamedParameterJdbcTemplate template;

    public ReservationDaoImpl(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
    }

    public Reservation save(Reservation reservation) {
        String sql = "INSERT INTO reservation (name, date, time_id)" +
                " VALUES (:name, :date, :time_id)";
        SqlParameterSource param = new MapSqlParameterSource()
                .addValue("name", reservation.getName())
                .addValue("date", reservation.getDate())
                .addValue("time_id", reservation.getTime().getId());

        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(sql, param, keyHolder);

        Long id = keyHolder.getKey().longValue();
        return Reservation.withId(id, reservation.getName(), reservation.getDate(), reservation.getTime());
    }
}
```

이제 서비스는 구체 구현체가 아니라 ReservationDao 인터페이스에 의존한다.

```
public class ReservationService {

    private final ReservationDao reservationDao;

    public ReservationService(ReservationDao reservationDao) {
        this.reservationDao = reservationDao;
    }
}
```

이를 통해 서비스는 데이터 접근 방식이 JdbcTemplate인지, JPA인지, InMemory 구현인지 알 필요가 없어진다.
즉, 서비스는 “예약을 저장한다”, “예약을 조회한다”는 역할에만 의존하고, 실제로 어떻게 저장하고 조회하는지는 구현체가 담당하게 된다.

## 왜 Repository에만 추상화를 적용했을까?

이번 코드에서는 모든 계층에 무조건 인터페이스를 만들기보다, 구현 방식이 달라질 가능성이 높은 계층에만 추상화를 적용했다.

### Repository

Repository는 DB나 데이터 접근 기술의 영향을 가장 많이 받는 계층이다.

예를 들어 JdbcTemplate, JPA, InMemory 저장소처럼 구현 방식이 달라질 수 있다.
또한 SQL 작성 방식, DB 종류, 테스트 방식에 따라 구현체가 변경될 가능성도 있다.

따라서 Repository는 인터페이스로 추상화했을 때 얻는 이점이 크다고 판단했다.

#### Service

Service는 비즈니스 로직을 담당한다.

현재 미션에서는 하나의 비즈니스 흐름에 대해 여러 구현 방식이 필요하지 않다고 판단했다.
예를 들어 ReservationService를 JdbcReservationService, JpaReservationService, InMemoryReservationService처럼 나눌 필요는 없다.

서비스는 데이터 저장 방식이 아니라 예약 생성, 조회, 삭제와 같은 비즈니스 규칙을 표현하는 계층이기 때문에, 현재 단계에서는 별도의 인터페이스를 두지 않았다.

### Controller

Controller는 HTTP 요청을 받고 응답을 반환하는 역할을 담당한다.

현재 REST API 방식 하나만 사용하고 있기 때문에 컨트롤러도 여러 구현체로 나눌 필요가 없다고 판단했다.
즉, 같은 요청을 처리하는 컨트롤러를 여러 방식으로 구현해야 하는 상황이 아니므로 인터페이스 추상화 대상에서 제외했다.
