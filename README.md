# item-service-spring-db2-

# 0. 데이터 접근 기술 - 시작

# 1. JdbcTemplate 소개와 설정

### **장점**

- 설정의 편리함
    - JdbcTemplate은 `spring-jdbc` 라이브러리에 포함되어 있는데, 이 라이브러리는 스프링으로 JDBC를 사용할 때 기본으로 사용되는 라이브러리입니다. 그리고 별도의 복잡한 설정 없이 바로 사용이 가능합니다.

- 반복 문제 해결
    - JdbcTemplate은 템플릿 콜백 패턴을 사용해서, JDBC를 직접 사용할 때 발생하는 대부분의 반복 작업을 대신 처리해줍니다.
    - 개발자는 SQL을 작성하고, 전달할 파리미터를 정의하고, 응답 값을 매핑하기만 하면 됩니다.
    - 우리가 생각할 수 있는 대부분의 반복 작업을 대신 처리해줍니다.
        - 커넥션 획득
        - `statement`를 준비하고 실행
        - 결과를 반복하도록 루프를 실행
        - 커넥션 종료, `statement`, `resultset` 종료
        - 트랜잭션 다루기 위한 커넥션 동기화
        - 예외 발생시 스프링 예외 변환기 실행

### **단점**

- 동적 SQL을 해결하기 어려움

### **JdbcTemplate 설정**

`build.gradle`

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    //JdbcTemplate 추가
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'

    //H2 데이터베이스 추가
    runtimeOnly 'com.h2database:h2'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    
    //테스트에서 lombok 사용
    testCompileOnly 'org.projectlombok:lombok'
    testAnnotationProcessor 'org.projectlombok:lombok'
}
```

`org.springframework.boot:spring-boot-starter-jdbc`를 추가하면 JdbcTemplate이 들어있는 `spring-jdbc`가 라이브러리에 포함됩니다.

여기서는 H2 데이터베이스에 접속해야 하기 때문에 H2 데이터베이스의 클라이언트 라이브러리(Jdbc Driver)도 추가해주었습니다.

# 2. JdbcTemplate 적용 1 - 기본

`JdbcTemplateItemRepositoryV1`

```java
import hello.itemservice.domain.Item;
import hello.itemservice.repository.ItemRepository;
import hello.itemservice.repository.ItemSearchCond;
import hello.itemservice.repository.ItemUpdateDto;
import lombok.extern.slf4j.Slf4j;
import org.springframework.dao.EmptyResultDataAccessException;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.jdbc.support.GeneratedKeyHolder;
import org.springframework.jdbc.support.KeyHolder;
import org.springframework.stereotype.Repository;
import org.springframework.util.StringUtils;

import javax.sql.DataSource;
import java.sql.PreparedStatement;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;

/**
* JdbcTemplate
*/
@Slf4j
@Repository
public class JdbcTemplateItemRepositoryV1 implements ItemRepository {
  
    private final JdbcTemplate template;
  
    public JdbcTemplateItemRepositoryV1(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }
  
    @Override
    public Item save(Item item) {
        String sql = "insert into item (item_name, price, quantity) values (?, ?, ?)";
        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(connection -> {
            //자동 증가 키
            PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});
            ps.setString(1, item.getItemName());
            ps.setInt(2, item.getPrice());
            ps.setInt(3, item.getQuantity());
            return ps;
        }, keyHolder);
      
        long key = keyHolder.getKey().longValue();
        item.setId(key);
        return item;
    }
  
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item set item_name=?, price=?, quantity=? where id=?";
        template.update(sql,
            updateParam.getItemName(),
            updateParam.getPrice(),
            updateParam.getQuantity(),
            itemId);
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name, price, quantity from item where id = ?";
        try {
            Item item = template.queryForObject(sql, itemRowMapper(), id);
            return Optional.of(item);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        String sql = "select id, item_name, price, quantity from item";
        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
      
        boolean andFlag = false;
        List<Object> param = new ArrayList<>();
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',?,'%')";
            param.add(itemName);
            andFlag = true;
        }
      
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
                sql += " price <= ?";
                param.add(maxPrice);
        }
      
        log.info("sql={}", sql);
        return template.query(sql, itemRowMapper(), param.toArray());
    }
  
    private RowMapper<Item> itemRowMapper() {
        return (rs, rowNum) -> {
            Item item = new Item();
            item.setId(rs.getLong("id"));
            item.setItemName(rs.getString("item_name"));
            item.setPrice(rs.getInt("price"));
            item.setQuantity(rs.getInt("quantity"));
            return item;
        };
    }
}
```

**기본**

`JdbcTemplateItemRepositoryV1`은 `ItemRepository` 인터페이스를 구현했습니다.

`this.template = new JdbcTemplate(dataSource)`

- `JdbcTemplate`은 데이터소스(`dataSource`)가 필요합니다.
- `JdbcTemplateItemRepositoryV1()` 생성자를 보면 `dataSource`를 의존 관계 주입 받고 생성자 내부에서 `JdbcTemplate`을 생성하고 있습니다. 스프링에서는 `JdbcTemplate`을 사용할 때 관례상 이 방법을 많이 사용합니다.
- 물론 `JdbcTemplate`을 스프링 빈으로 직접 등록하고 주입받아도 됩니다.

`save()`

데이터를 저장하는 기능을 합니다.

`template.update()` : 데이터를 변경할 때는 `update()`를 사용합니다.

`INSERT`, `UPDATE`, `DELETE` SQL에 사용합니다.

`template.update()`의 반환 값은 `int`인데, 영향 받은 row의 수를 반환합니다.

데이터를 저장할 때 PK 생성에 `identity` (auto increment) 방식을 사용하기 때문에, PK인 ID 값을 개발자가 직접 지정하는 것이 아니라 비워두고 저장해야 합니다.. 그러면 데이터베이스가 PK인 ID를 대신 생성

문제는 이렇게 데이터베이스가 대신 생성해주는 PK ID 값은 데이터베이스가 생성하기 때문에, 데이터베이스에 INSERT 가 완료 되어야 생성된 PK ID 값을 확인할 수 있습니다.

`KeyHolder`와 `connection.prepareStatement(sql, new String[]{"id"})` 를 사용해서 id 를 지정해주면 INSERT 쿼리 실행 이후에 데이터베이스에서 생성된 ID 값을 조회할 수 있습니다.

물론 데이터베이스에서 생성된 ID 값을 조회하는 것은 순수 JDBC로도 가능하지만, 코드가 훨씬 더 복잡합니다.

참고로 뒤에서 설명하겠지만 JdbcTemplate 이 제공하는 `SimpleJdbcInsert` 라는 훨씬 편리한 기능이 있으므로 대략 이렇게 사용한다 정도로만 알아둘 두면 됩니다.

`update()`

데이터를 업데이트하는 기능을 합니다.

`template.update()`: 데이터를 변경할 때는 `update()`를 사용합니다.

- `?`에 바인딩할 파라미터를 순서대로 전달
- 반환 값은 해당 쿼리의 영향을 받은 row의 수 입니다. 여기서는 `where id=?`를 지정했기 때문에 영향 받은 로우 수는 최대 1개입니다.

`findById()`

데이터를 하나 조회하는 기능을 합니다.

`template.queryForObject()`

- 결과 로우가 하나일 때 사용합니다.
- `RowMapper`는 데이터베이스의 반환 결과인 `ResultSet`을 객체로 변환합니다.
- 결과가 없으면 `EmptyResultDataAccessException` 예외가 발생합니다.
- 결과가 둘 이상이면 `IncorrectResultSizeDataAccessException` 예외가 발생합니다.

`ItemRepository.findById()` 인터페이스는 결과가 없을 때 `Optional`을 반환해야 합니다. 따라서 결과가 없으면 예외를 잡아서 `Optional.empty`를 대신 반환합니다.

`queryForObject()` 인터페이스 정의

```java
<T> T queryForObject(String sql, RowMapper<T> rowMapper, Object... args) 
			throws DataAccessException;
```

`findAll()`

데이터를 리스트로 조회하고 검색 조건으로 적절한 데이터를 검색하는 기능을 합니다.

`template.query()`

- 결과가 하나 이상일 때 사용
- `RowMapper`는 데이터베이스의 반환 결과인 `ResultSet`을 객체로 변환
- 결과가 없으면 빈 컬렉션을 반환

`query()` 인터페이스 정의

```java
<T> List<T> query(String sql, RowMapper<T> rowMapper, Object... args) 
			throws DataAccessException;
```

`itemRowMapper()`

- 데이터베이스의 조회 결과를 객체로 변환할 때 사용합니다.
- JDBC 를 직접 사용할 때 `ResultSet`를 사용했던 부분과 유사합니다.
- 차이가 있다면 다음과 같이 JdbcTemplate이 다음과 같은 루프를 돌려주고, 개발자는 `RowMapper`를 구현해서 그 내부 코드만 채우면 된다는 것입니다.

```java
while(resultSet 이 끝날 때 까지) {
    rowMapper(rs, rowNum)
}
```

# 3. JdbcTemplate 적용 2 - 동적 쿼리 문제

앞서 말했듯 JdbcTemplate 기술에서는 동적 쿼리를 적용하기 어렵다는 문제가 있습니다.

결과를 검색하는 `findAll()`에서 어려운 부분은 사용자가 검색하는 값에 따라서 실행하는 SQL이 동적으로 달려져야 합니다. 

검색 상황에 따라 아래처럼 4가지 상황이 있습니다.

- 검색 조건이 없음

```sql
select id, item_name, price, quantity from item
```

- 상품명( itemName )으로 검색

```sql
select id, item_name, price, quantity from item
where item_name like concat('%',?,'%')
```

- 최대 가격( maxPrice )으로 검색

```sql
select id, item_name, price, quantity from item
where price <= ?
```

- 상품명( itemName ), 최대 가격( maxPrice ) 둘다 검색

```sql
select id, item_name, price, quantity from item
where item_name like concat('%',?,'%')
and price <= ?
```

결과적으로 4가지 상황에 따른 SQL을 동적으로 생성해야 한다. 동적 쿼리가 쉬워 보이지만, 막상 개발해보면 생각보다 다양한 상황을 고민해야 합니다. 

예를 들어서 어떤 경우에는 `where`를 앞에 넣고 어떤 경우에는 `and`를 넣어야 하는지 등을 모두 계산해야 합니다.

그리고 각 상황에 맞추어 파라미터도 생성해야 합니다. 물론 실무에서는 이보다 훨씬 더 복잡한 동적 쿼리들이 사용됩니다.


# 5. JdbcTemplate - 이름 지정 파라미터 1

### **순서대로 바인딩**

JdbcTemplate을 기본으로 사용하면 파라미터를 순서대로 바인딩합니다.

```java
String sql = "update item set item_name=?, price=?, quantity=? where id=?";
template.update(sql,
                itemName,
                price,
                quantity,
                itemId);
```

여기서는 `itemName`, `price`, `quantity`가 SQL에 있는 `?`에 순서대로 바인딩하고 있습니다. 

따라서 순서만 잘 지키면 문제가 될 것은 없으나 문제는 변경시점에 발생 누군가 다음과 같이 SQL 코드의 순서를 변경했다고 가정합시다. (`price`와 `quantity`의 순서를 변경)

```java
String sql = "update item set item_name=?, quantity=?, price=? where id=?";
template.update(sql,
                itemName,
                price,
                quantity,
                itemId);
```

이렇게 되면 다음과 같은 순서로 데이터가 바인딩됩니다.

```java
item_name=itemName, quantity=price, price=quantity
```

결과적으로 `price`와 `quantity`가 바뀌는 매우 심각한 문제가 발생합니다. 이럴 일이 없을 것 같지만, 실무에서는 파라미터가 10~20개가 넘어가는 일도 아주 많습니다. 그래서 미래에 필드를 추가하거나 수정하면서 이런 문제가 충분히 발생할 수 있습니다.

버그 중에서 가장 고치기 힘든 버그는 데이터베이스에 데이터가 잘못 들어가는 버그입니다. 이것은 코드만 고치는 수준이 아니라 데이터베이스의 데이터를 복구해야 하기 때문에 버그를 해결하는데 들어가는 리소스가 굉장히 많습니다.

**개발을 할 때는 코드를 몇줄 줄이는 편리함도 중요하지만, 모호함을 제거해서 코드를 명확하게 만드는 것이 유지보수 관점에서 매우 중요합니다.**

이처럼 파라미터를 순서대로 바인딩 하는 것은 편리하기는 하지만, 순서가 맞지 않아서 버그가 발생할 수도 있으므로 주의해서 사용해야 합니다.