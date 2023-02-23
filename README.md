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

# 6. JdbcTemplate - 이름 지정 파라미터 2

### **이름 지정 바인딩**

JdbcTemplate 은 이런 문제를 보완하기 위해 `NamedParameterJdbcTemplate`라는 이름을 지정해서 파라미터를 바인딩 하는 기능을 제공합니다.

`JdbcTemplateItemRepositoryV2`

```java
/**
* NamedParameterJdbcTemplate
* SqlParameterSource
* - BeanPropertySqlParameterSource
* - MapSqlParameterSource
* Map
*
* BeanPropertyRowMapper
*
*/
@Slf4j
@Repository
public class JdbcTemplateItemRepositoryV2 implements ItemRepository {
  
    private final NamedParameterJdbcTemplate template;
  
    public JdbcTemplateItemRepositoryV2(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
    }
  
    @Override
    public Item save(Item item) {
        String sql = "insert into item (item_name, price, quantity) " +
        "values (:itemName, :price, :quantity)";
      
        SqlParameterSource param = new BeanPropertySqlParameterSource(item);
        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(sql, param, keyHolder);
      
        Long key = keyHolder.getKey().longValue();
        item.setId(key);
        return item;
    }
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item " +
        "set item_name=:itemName, price=:price, quantity=:quantity " +
        "where id=:id";
      
        SqlParameterSource param = new MapSqlParameterSource()
            .addValue("itemName", updateParam.getItemName())
            .addValue("price", updateParam.getPrice())
            .addValue("quantity", updateParam.getQuantity())
            .addValue("id", itemId); //이 부분이 별도로 필요하다.
        template.update(sql, param);
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name, price, quantity from item where id = :id";
      
        try {
            Map<String, Object> param = Map.of("id", id);
            Item item = template.queryForObject(sql, param, itemRowMapper());
            return Optional.of(item);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
      
        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();
      
        SqlParameterSource param = new BeanPropertySqlParameterSource(cond);
      
        String sql = "select id, item_name, price, quantity from item";
        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
        boolean andFlag = false;
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',:itemName,'%')";
            andFlag = true;
        }
      
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= :maxPrice";
        }
      
        log.info("sql={}", sql);
        return template.query(sql, param, itemRowMapper());
    }
  
    private RowMapper<Item> itemRowMapper() {
        return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원
    }
}
```

**기본**

`JdbcTemplateItemRepositoryV2`는 `ItemRepository` 인터페이스를 구현했습니다.

`this.template = new NamedParameterJdbcTemplate(dataSource)`

- `NamedParameterJdbcTemplate`도 내부에 `dataSource`가 필요합니다.
- `JdbcTemplateItemRepositoryV2` 생성자를 보면 의존관계 주입은 `dataSource`를 받고 내부에서 `NamedParameterJdbcTemplate`을 생성해서 보유합니다. 스프링에서는 `JdbcTemplate` 관련 기능을 사용할 때 관례상 이 방법을 많이 사용합니다.
- 물론 `NamedParameterJdbcTemplate`을 스프링 빈으로 직접 등록하고 주입받아도 됩니다.

`save()` 

SQL에서 다음과 같이 `?` 대신에 `:파라미터이름`을 받는 것을 확인할 수 있습니다.

```java
insert into item (item_name, price, quantity) " +
"values (:itemName, :price, :quantity)"
```

추가로 `NamedParameterJdbcTemplate`은 데이터베이스가 생성해주는 키를 매우 쉽게 조회하는 기능도 제공합니다.

# 6. JdbcTemplate - 이름 지정 파라미터 2

### **이름 지정 바인딩**

JdbcTemplate 은 이런 문제를 보완하기 위해 `NamedParameterJdbcTemplate`라는 이름을 지정해서 파라미터를 바인딩 하는 기능을 제공합니다.

`JdbcTemplateItemRepositoryV2`

```java
/**
* NamedParameterJdbcTemplate
* SqlParameterSource
* - BeanPropertySqlParameterSource
* - MapSqlParameterSource
* Map
*
* BeanPropertyRowMapper
*
*/
@Slf4j
@Repository
public class JdbcTemplateItemRepositoryV2 implements ItemRepository {
  
    private final NamedParameterJdbcTemplate template;
  
    public JdbcTemplateItemRepositoryV2(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
    }
  
    @Override
    public Item save(Item item) {
        String sql = "insert into item (item_name, price, quantity) " +
        "values (:itemName, :price, :quantity)";
      
        SqlParameterSource param = new BeanPropertySqlParameterSource(item);
        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(sql, param, keyHolder);
      
        Long key = keyHolder.getKey().longValue();
        item.setId(key);
        return item;
    }
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item " +
        "set item_name=:itemName, price=:price, quantity=:quantity " +
        "where id=:id";
      
        SqlParameterSource param = new MapSqlParameterSource()
            .addValue("itemName", updateParam.getItemName())
            .addValue("price", updateParam.getPrice())
            .addValue("quantity", updateParam.getQuantity())
            .addValue("id", itemId); //이 부분이 별도로 필요하다.
        template.update(sql, param);
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name, price, quantity from item where id = :id";
      
        try {
            Map<String, Object> param = Map.of("id", id);
            Item item = template.queryForObject(sql, param, itemRowMapper());
            return Optional.of(item);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
      
        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();
      
        SqlParameterSource param = new BeanPropertySqlParameterSource(cond);
      
        String sql = "select id, item_name, price, quantity from item";
        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
        boolean andFlag = false;
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',:itemName,'%')";
            andFlag = true;
        }
      
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= :maxPrice";
        }
      
        log.info("sql={}", sql);
        return template.query(sql, param, itemRowMapper());
    }
  
    private RowMapper<Item> itemRowMapper() {
        return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원
    }
}
```

**기본**

`JdbcTemplateItemRepositoryV2`는 `ItemRepository` 인터페이스를 구현했습니다.

`this.template = new NamedParameterJdbcTemplate(dataSource)`

- `NamedParameterJdbcTemplate`도 내부에 `dataSource`가 필요합니다.
- `JdbcTemplateItemRepositoryV2` 생성자를 보면 의존관계 주입은 `dataSource`를 받고 내부에서 `NamedParameterJdbcTemplate`을 생성해서 보유합니다. 스프링에서는 `JdbcTemplate` 관련 기능을 사용할 때 관례상 이 방법을 많이 사용합니다.
- 물론 `NamedParameterJdbcTemplate`을 스프링 빈으로 직접 등록하고 주입받아도 됩니다.

`save()` 

SQL에서 다음과 같이 `?` 대신에 `:파라미터이름`을 받는 것을 확인할 수 있습니다.

```java
insert into item (item_name, price, quantity) " +
"values (:itemName, :price, :quantity)"
```

추가로 `NamedParameterJdbcTemplate`은 데이터베이스가 생성해주는 키를 매우 쉽게 조회하는 기능도 제공합니다.

### **이름 지정 파라미터**

파라미터를 전달 시 `Map`처럼 `key`, `value` 데이터 구조를 만들어서 전달합니다. 

여기서 `key`는 `:파리이터이름`으로 지정한 파라미터의 이름이고 , `value`는 해당 파라미터의 값

이름을 지정하는 것입니다. 바인딩에서 자주 사용하는 파라미터의 종류는 크게 3가지가 있습니다.

- `Map`
- `SqlParameterSource`
    - `MapSqlParameterSource`
    - `BeanPropertySqlParameterSource`

### Map

단순히 `Map`을 사용합니다.

```java
Map<String, Object> param = Map.of("id", id);
Item item = template.queryForObject(sql, param, itemRowMapper());
```

### MapSqlParameterSource

`Map`과 유사한데 SQL 타입을 지정할 수 있는 등 SQL에 좀 더 특화된 기능을 제공합니다. 

`SqlParameterSource` 인터페이스의 구현체입니다.

`MapSqlParameterSource`는 메서드 체인을 통해 편리한 사용법도 제공합니다.

```java
SqlParameterSource param = new MapSqlParameterSource()
  .addValue("itemName", updateParam.getItemName())
  .addValue("price", updateParam.getPrice())
  .addValue("quantity", updateParam.getQuantity())
  .addValue("id", itemId); //이 부분이 별도로 필요하다.
template.update(sql, param);
```

### BeanPropertySqlParameterSource

자바빈 프로퍼티 규약을 통해서 자동으로 파라미터 객체를 생성합니다. 

예) ( `getXxx() -> xxx, getItemName() -> itemName` ) 

예를 들어서 `getItemName()`, `getPrice()`가 있으면 다음과 같은 데이터를 자동으로 생성합니다.

- `key=itemName, value=상품명 값`
- `key=price, value=가격 값`

### `SqlParameterSource` 인터페이스의 구현체

```java
SqlParameterSource param = new BeanPropertySqlParameterSource(item);
KeyHolder keyHolder = new GeneratedKeyHolder();
template.update(sql, param, keyHolder);
```

여기서 보면 `BeanPropertySqlParameterSource`가 많은 것을 자동화 해주기 때문에 가장 좋아보이지만, `BeanPropertySqlParameterSource`를 항상 사용할 수 있는 것은 아닙니다.

예를 들어서 `update()`에서는 SQL에 `:id`를 바인딩 해야 하는데, `update()`에서 사용하는 `ItemUpdateDto`에는 `itemId`가 없습니다.

따라서 `BeanPropertySqlParameterSource`를 사용할 수 없고, 대신에 `MapSqlParameterSource`를 사용합니다.

### **BeanPropertyRowMapper**

**BeanPropertyRowMapper 미사용**

```java
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
```

**BeanPropertyRowMapper 사용**

```java
private RowMapper<Item> itemRowMapper() {
    return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원
}
```

`BeanPropertyRowMapper`는 ResultSet 의 결과를 받아서 자바빈 규약에 맞추어 데이터를 변환합니다. 

예를 들어서 데이터베이스에서 조회한 결과가 `select id, price` 라고 하면 아래과 같은 코드를 작성합니다. (실제로는 완전히 같은 코드는 아니고 리플렉션 같은 기능을 사용함)

```java
Item item = new Item();
item.setId(rs.getLong("id"));
item.setPrice(rs.getInt("price"));
```

데이터베이스에서 조회한 결과 이름을 기반으로 `setId()`, `setPrice()`처럼 자바빈 프로퍼티 규약에 맞춘 메서드를 호출합니다.

### **별칭**

그런데 `select item_name`의 경우 `setItem_name()`이라는 메서드가 없습니다. 이런 경우 개발자가 조회 SQL을 아래처럼 고치면 됩니다. 

```sql
select item_name as itemName
```

별칭 `as` 를 사용해서 SQL 조회 결과의 이름을 변경할 수 있습니다.. 실제로 이 방법은 자주 사용됩니다. 

특히 데이터베이스 컬럼 이름과 객체 이름이 완전히 다를 때 문제를 해결할 수 있습니다. 예를 들어서 데이터베이스에는 `member_name`이라고 되어 있는데 객체에 `username` 이라고 되어 있다면 다음과 같이 해결할 수 있습니다. 

```sql
select member_name as username
```

이렇게 데이터베이스 컬럼 이름과 객체의 이름이 다를 때 별칭(`as`)을 사용해서 문제를 많이 해결합니다.  `JdbcTemplate`은 물론이고, `MyBatis` 같은 기술에서도 자주 사용됩니다.

### **관례의 불일치**

자바 객체는 카멜(`camelCase`) 표기법을 사용합니다. 반면에 관계형 데이터베이스에서는 주로 언더스코어를 사용하는 `snake_case` 표기법을 사용합니다.

이 부분을 관례로 많이 사용하다 보니 `BeanPropertyRowMapper`는 언더스코어 표기법을 카멜로 자동 변환합니다. 따라서 `select item_name`으로 조회해도 `setItemName()`에 문제 없이 값이 들어갑니다.

정리하면 `snake_case`는 자동으로 해결되니 그냥 두면 되고, 컬럼 이름과 객체 이름이 완전히 다른 경우에는 조회 SQL에서 별칭을 사용하면 됩니다!

작성한 코드를 설정해 봅시다.

`JdbcTemplateV2Config`

```java
import hello.itemservice.repository.ItemRepository;
import hello.itemservice.repository.jdbctemplate.JdbcTemplateItemRepositoryV2;
import hello.itemservice.service.ItemService;
import hello.itemservice.service.ItemServiceV1;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.sql.DataSource;

@Configuration
@RequiredArgsConstructor
public class JdbcTemplateV2Config {
  
    private final DataSource dataSource;
  
    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
  
    @Bean
    public ItemRepository itemRepository() {
        return new JdbcTemplateItemRepositoryV2(dataSource);
    }
}
```

앞서 개발한 `JdbcTemplateItemRepositoryV2`를 사용하도록 스프링 빈에 등록했습니다.

`ItemServiceApplication` - 변경

```java
import hello.itemservice.config.*;
import hello.itemservice.repository.ItemRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Profile;

@Slf4j
//@Import(MemoryConfig.class)
//@Import(JdbcTemplateV1Config.class)
@Import(JdbcTemplateV2Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {
  
    public static void main(String[] args) {
        SpringApplication.run(ItemServiceApplication.class, args);
    }
  
    @Bean
    @Profile("local")
    public TestDataInit testDataInit(ItemRepository itemRepository) {
        return new TestDataInit(itemRepository);
    }
}
```

`JdbcTemplateV2Config.class`를 설정으로 사용하도록 변경합니다.

- `@Import(JdbcTemplateV1Config.class)` → `@Import(JdbcTemplateV2Config.class)`

이후에 실행해보면 모두 정상적으로 실행되는 것을 확인할 수 있습니다.

# 7. JdbcTemplate - SimpleJdbcInsert

JdbcTemplate은 INSERT SQL를 직접 작성하지 않아도 되도록 `SimpleJdbcInsert`라는 편리한 기능을 제공합니다.

`JdbcTemplateItemRepositoryV3`

```java
/**
* SimpleJdbcInsert
*/
@Slf4j
@Repository
public class JdbcTemplateItemRepositoryV3 implements ItemRepository {
  
    private final NamedParameterJdbcTemplate template;
    private final SimpleJdbcInsert jdbcInsert;
  
    public JdbcTemplateItemRepositoryV3(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
        this.jdbcInsert = new SimpleJdbcInsert(dataSource)
          .withTableName("item")
          .usingGeneratedKeyColumns("id");
      // .usingColumns("item_name", "price", "quantity"); //생략 가능
    }
  
    @Override
    public Item save(Item item) {
        SqlParameterSource param = new BeanPropertySqlParameterSource(item);
        Number key = jdbcInsert.executeAndReturnKey(param);
        item.setId(key.longValue());
        return item;
    }
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item " +
        "set item_name=:itemName, price=:price, quantity=:quantity " +
        "where id=:id";
      
        SqlParameterSource param = new MapSqlParameterSource()
          .addValue("itemName", updateParam.getItemName())
          .addValue("price", updateParam.getPrice())
          .addValue("quantity", updateParam.getQuantity())
          .addValue("id", itemId);
        template.update(sql, param);
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name, price, quantity from item where id = :id";
        try {
            Map<String, Object> param = Map.of("id", id);
            Item item = template.queryForObject(sql, param, itemRowMapper());
            return Optional.of(item);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
      
        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();
      
        SqlParameterSource param = new BeanPropertySqlParameterSource(cond);
        String sql = "select id, item_name, price, quantity from item";
        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }
      
        boolean andFlag = false;
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',:itemName,'%')";
            andFlag = true;
        }
      
        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= :maxPrice";
        }
        log.info("sql={}", sql);
        return template.query(sql, param, itemRowMapper());
    }
  
    private RowMapper<Item> itemRowMapper() {
        return BeanPropertyRowMapper.newInstance(Item.class);
    }
}
```

### **기본**

`JdbcTemplateItemRepositoryV3`은 `ItemRepository` 인터페이스를 구현합니다.

`this.jdbcInsert = new SimpleJdbcInsert(dataSource)`

- 생성자를 보면 의존관계 주입은 `dataSource`를 받고 내부에서 `SimpleJdbcInsert`을 생성해서 가지고 있습니다. 스프링에서는 `JdbcTemplate` 관련 기능을 사용할 때 관례상 이 방법을 많이 사용합니다.
- 물론 `SimpleJdbcInsert` 을 스프링 빈으로 직접 등록하고 주입받아도 됩니다.

`SimpleJdbcInsert`

```java
this.jdbcInsert = new SimpleJdbcInsert(dataSource)
  .withTableName("item")
  .usingGeneratedKeyColumns("id");
// .usingColumns("item_name", "price", "quantity"); //생략 가능
```

`withTableName`: 데이터를 저장할 테이블 명을 지정.

`usingGeneratedKeyColumns`: key 를 생성하는 PK 컬럼 명을 지정.

`usingColumns`: INSERT SQL에 사용할 컬럼을 지정. 특정 값만 저장하고 싶을 때 사용하며 생략 가능함.

`SimpleJdbcInsert`는 생성 시점에 데이터베이스 테이블의 메타 데이터를 조회하므로 어떤 컬럼이 있는지 확인 할 수 있습니다. 그래서 `usingColumns`을 생략할 수 있습니다. 

만약 특정 컬럼만 지정해서 저장하고 싶다면 `usingColumns`를 사용하면 됩니다.

애플리케이션을 실행해보면 `SimpleJdbcInsert`이 어떤 INSERT SQL을 만들어서 사용하는지 로그로 확인할 수 있음

```java
DEBUG 39424 --- [ main] o.s.jdbc.core.simple.SimpleJdbcInsert :
Compiled insert object: insert string is [INSERT INTO item (ITEM_NAME, PRICE, QUANTITY) VALUES(?, ?, ?)]
```

`save()` 

`jdbcInsert.executeAndReturnKey(param)`을 사용해서 INSERT SQL을 실행하고, 생성된 키 값도 매우 편리하게 조회할 수 있습니다.

```java
public Item save(Item item) {
    SqlParameterSource param = new BeanPropertySqlParameterSource(item);
    Number key = jdbcInsert.executeAndReturnKey(param);
    item.setId(key.longValue());
    return item;
}
```

`JdbcTemplateV3Config`

```java
@Configuration
@RequiredArgsConstructor
public class JdbcTemplateV3Config {
  
    private final DataSource dataSource;
  
    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
  
    @Bean
    public ItemRepository itemRepository() {
        return new JdbcTemplateItemRepositoryV3(dataSource);
    }
}
```

`JdbcTemplateItemRepositoryV3`를 사용하도록 스프링 빈에 등록합니다.

`ItemServiceApplication` - 변경

```java
@Slf4j
//@Import(MemoryConfig.class)
//@Import(JdbcTemplateV1Config.class)
//@Import(JdbcTemplateV2Config.class)
@Import(JdbcTemplateV3Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {}
```

`JdbcTemplateV3Config.class` 를 설정으로 사용하도록 변경

- `@Import(JdbcTemplateV2Config.class)` → `@Import(JdbcTemplateV3Config.class)`

# 8. JdbcTemplate 기능 정리

### **주요 기능**

`JdbcTemplate`

- 순서 기반 파라미터 바인딩을 지원

`NamedParameterJdbcTemplate`

- 이름 기반 파라미터 바인딩을 지원 (권장)

`SimpleJdbcInsert`

- INSERT SQL을 편리하게 사용 가능

`SimpleJdbcCall`

- 스토어드 프로시저를 편리하게 호출 가능

> 참고 - 스토어드 프로시저를 사용하기 위한 SimpleJdbcCall 에 대한 자세한 내용은 다음 스프링 공식 메뉴얼을 참고합시다. 
https://docs.spring.io/spring-framework/docs/current/reference/html/dataaccess.html#jdbc-simple-jdbc-call-1
> 

### **JdbcTemplate 사용법 정리**

> 참고 - 스프링 JdbcTemplate 사용 방법 공식 메뉴얼 
https://docs.spring.io/spring-framework/docs/current/reference/html/dataaccess.html#jdbc-JdbcTemplate
> 

### **조회**

**단건 조회 - 숫자 조회**

```java
int rowCount = jdbcTemplate.queryForObject("select count(*) from t_actor", Integer.class);
```

하나의 row 를 조회할 때는 `queryForObject()`를 사용합니다. 

지금처럼 조회 대상이 객체가 아니라 단순 데이터 하나라면 타입을 `Integer.class`, `String.class`와 같이 지정합니다.

**단건 조회 - 숫자 조회, 파라미터 바인딩**

```java
int countOfActorsNamedJoe = 
		jdbcTemplate.queryForObject(
				"select count(*) from t_actor where first_name = ?", 
				Integer.class, "Joe");
```

**단건 조회 - 문자 조회**

```java
String lastName = jdbcTemplate.queryForObject(
				"select last_name from t_actor where id = ?", 
				String.class, 1212L);
```

**단건 조회 - 객체 조회**

```java
Actor actor = jdbcTemplate.queryForObject(
    "select first_name, last_name from t_actor where id = ?",
    (resultSet, rowNum) -> {
        Actor newActor = new Actor();
        newActor.setFirstName(resultSet.getString("first_name"));
        newActor.setLastName(resultSet.getString("last_name"));
        return newActor;
    },
  1212L);
```

객체 하나를 조회합니다. 결과를 객체로 매핑해야 하므로 `RowMapper`를 사용합니다.

**목록 조회 - 객체**

```java
List<Actor> actors = jdbcTemplate.query(
		"select first_name, last_name from t_actor",
		(resultSet, rowNum) -> {
				Actor actor = new Actor();
				actor.setFirstName(resultSet.getString("first_name"));
				actor.setLastName(resultSet.getString("last_name"));
		return actor;
});
```

여러 row 를 조회할 때는 `query()`를 사용합니다. 결과를 리스트로 반환 결과를 객체로 매핑해야 하므로 `RowMapper`를 사용합니다.

**목록 조회 - 객체**

```java
private final RowMapper<Actor> actorRowMapper = (resultSet, rowNum) -> {
    Actor actor = new Actor();
    actor.setFirstName(resultSet.getString("first_name"));
    actor.setLastName(resultSet.getString("last_name"));
    return actor;
};

public List<Actor> findAllActors() {
    return this.jdbcTemplate.query("select first_name, last_name from t_actor", actorRowMapper);
}
```

`RowMapper`를 분리하면 여러 곳에서 재사용이 가능합니다.

### **변경(INSERT, UPDATE, DELETE)**

데이터를 변경할 때는 `jdbcTemplate.update()`를 사용합니다. 참고로 `int` 반환값을 반환하는데, SQL 실행 결과에 영향받은 row 수를 반환합니다.

**등록**

```java
jdbcTemplate.update(
  "insert into t_actor (first_name, last_name) values (?, ?)",
  "Leonor", "Watling");
```

**수정**

```java
jdbcTemplate.update(
  "update t_actor set last_name = ? where id = ?",
  "Banjo", 5276L);
```

**삭제**

```java
jdbcTemplate.update(
  "delete from t_actor where id = ?",
  Long.valueOf(actorId));
```

### **기타 기능**

임의의 SQL을 실행할 때는 `execute()`를 사용합니다. 테이블을 생성하는 DDL에 사용이 가능합니다.

**DDL**

```java
jdbcTemplate.execute("create table mytable (id integer, name varchar(100))");
```

**스토어드 프로시저 호출**

```java
jdbcTemplate.update(
  "call SUPPORT.REFRESH_ACTORS_SUMMARY(?)",
  Long.valueOf(unionId));
```

### **정리**

실무에서 가장 간단하고 실용적인 방법으로 SQL을 사용하려면 JdbcTemplate을 사용하면 됩니다. 

JPA와 같은 ORM 기술을 사용하면서 동시에 SQL을 직접 작성해야 할 때가 있는데, 그때도 JdbcTemplate을 함께 사용하면 됩니다. 

그런데 JdbcTemplate의 최대 단점이 있는데, 바로 동적 쿼리 문제를 해결하지 못한다는 점입니다. 그리고 SQL을 자바 코드로 작성하기 때문에 SQL 라인이 코드를 넘어갈 때 마다 문자 더하기를 해주어야 하는 단점도 있습니다.

# ==== 2. 데이터 접근 기술 - 테스트 ====
# 1. 테스트 - 데이터베이스 연동

데이터 접근 기술을 개발할 때에는 실제 데이터베이스에 접근해서 데이터를 잘 저장하고 조회할 수 있는지 확인하는 것이 필요합니다.

테스트를 실행할 때 실제 데이터베이스를 연동해서 진행 앞서 개발한 `ItemRepositoryTest`를 통해서 테스트를 진행하겠습니다.

### **main - application.properties**

`src/main/resources/application.properties`

```java
spring.profiles.active=local
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa

logging.level.org.springframework.jdbc=debug
```

### **test - application.properties**

`src/test/resources/application.properties`

```java
spring.profiles.active=test
```

테스트 케이스는 `src/test`에 있기 때문에, 실행하면 `src/test`에 있는 `application.properties` 파일이 우선순위를 가지고 실행됩니다. 그런데 테스트용 설정에는 `spring.datasource.url` 과 같은 데이터베이스 연결 설정이 없습니다.

테스트 케이스에서도 데이터베이스에 접속할 수 있게 test의 `aplication.properties` 를 다음과 같이 수정합니다.

**test - application.properties 수정** 

`src/test/resources/application.properties`

```java
spring.profiles.active=test
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa

logging.level.org.springframework.jdbc=debug
```

### **테스트 실행 - 로컬DB**

**@SpringBootTest**

```java
@SpringBootTest
class ItemRepositoryTest { ... }
```

`ItemRepositoryTest`는 `@SpringBootTest`를 사용합니다. 

`@SpringBootTest` 는 `@SpringBootApplication`를 찾아서 설정으로 사용합니다.

**@SpringBootApplication**

```java
@Slf4j
//@Import(MemoryConfig.class)
//@Import(JdbcTemplateV1Config.class)
//@Import(JdbcTemplateV2Config.class)
@Import(JdbcTemplateV3Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {}
```

`@SpringBootApplication` 설정이 과거에는 `MemoryConfig.class`를 사용하다가 이제는 `JdbcTemplateV3Config.class`를 사용하도록 변경되었습니다. 따라서 테스트도 `JdbcTemplate`을 통해 실제 데이터베이스를 호출합니다.

- `MemoryItemRepository` → `JdbcTemplateItemRepositoryV3`

### **실행 결과**

`updateitem()`: 성공

`save()`: 성공

`findItems()`: 실패

`findItems()`는 다음과 같은 오류를 내면서 실패

```java
java.lang.AssertionError:
Expecting actual:
  [Item(id=7, itemName=ItemTest, price=10000, quantity=10),
    Item(id=8, itemName=itemA, price=10000, quantity=10),
    Item(id=9, itemName=itemB, price=20000, quantity=20),
    Item(id=10, itemName=itemA, price=10000, quantity=10),
...
```

`findItems()` 코드를 확인해보면 상품을 3개 저장하고, 조회하는 동작입니다.

`ItemRepositoryTest.findItems()`

```java
@Test
void findItems() {
    //given
    Item item1 = new Item("itemA-1", 10000, 10);
    Item item2 = new Item("itemA-2", 20000, 20);
    Item item3 = new Item("itemB-1", 30000, 30);
  
    itemRepository.save(item1);
    itemRepository.save(item2);
    itemRepository.save(item3);
  
    //여기서 3개 이상이 조회되는 문제가 발생
    test(null, null, item1, item2, item3);
}
```

결과적으로 테스트에서 저정한 3개의 데이터가 조회되어야 하는데, 기대보다 더 많은 데이터가 조회됩니다.

**실패 원인**

테스트를 실행할 때 `TestDataInit`이 실행되어서 실패한 것일까요? 

이 문제는 아닙니다. `TestDataInit`은 프로필이 `local`일 때만 동작하는데, 테스트 케이스를 실행할 때는 프로필이 `spring.profiles.active=test`이기 때문에 초기화 데이터가 추가되지 않습니다.

문제는 H2 데이터베이스에 이미 과거에 서버를 실행하면서 저장했던 데이터가 보관되어 있기 때문입니다. 이 데이터가 현재 테스트에 영향을 줍니다.

### **H2 데이터베이스 데이터 확인**

http://localhost:8082

`SELECT * FROM ITEM`을 실행하면 이미 서버를 실행해서 확인 할 때의 데이터가 저장되어 있는 것을 확인할 수 있습니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f240921e-85b9-4282-8dad-4725ff2b4513/Untitled.png)

# 2. 테스트 - 데이터베이스 분리

로컬에서 사용하는 애플리케이션 서버와 테스트에서 같은 데이터베이스를 사용하고 있으니 테스트에서 문제가 발생하였습니다. 이런 문제를 해결하기 위해 테스트를 다른 환경과 철저하게 분리해야 합니다.

가장 간단한 방법은 테스트 전용 데이터베이스를 별도로 운영하는 것입니다.

**H2 데이터베이스를 용도에 따라 2가지로 구분합시다.**

- `jdbc:h2:tcp://localhost/~/test` : local에서 접근하는 서버 전용 데이터베이스
- `jdbc:h2:tcp://localhost/~/testcase` : test 케이스에서 사용하는 전용 데이터베이스

### **데이터베이스 파일 생성 방법**

데이터베이스 서버를 종료하고 다시 실행합시다.

- 사용자명:  sa
- JDBC URL: `jdbc:h2:~/testcase` (최초 한번)

`~/testcase.mv.db` 파일 생성을 확인합니다.

이후부터는 `jdbc:h2:tcp://localhost/~/testcase` 접속하면 됩니다.

### **테이블 생성하기**

`testcase` 데이터베이스에도 `item` 테이블을 생성합시다.

`sql/schema.sql` 파일 참고

```sql
drop table if exists item CASCADE;
create table item
(
    id bigint generated by default as identity,
    item_name varchar(10),
    price integer,
    quantity integer,
    primary key (id)
);
```

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/c9b6fe96-45a0-4070-ab87-18f423c960a9/Untitled.png)

### **접속 정보 변경**

프로젝트의 `main`에 있는 `application.properties`는 그대로 유지하고, `test`에 있는 `application.properties`만 변경해야 합니다.

**main - application.properties** 

`src/main/resources/application.properties`

```sql
spring.profiles.active=local
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
```

**test - application.properties** 

`src/test/resources/application.properties`

```sql
spring.profiles.active=test
spring.datasource.url=jdbc:h2:tcp://localhost/~/testcase
spring.datasource.username=sa
```

접속 정보가 `jdbc:h2:tcp://localhost/~/test` → `jdbc:h2:tcp://localhost/~/testcase`로 변경되었습니다.

`findItems()`: 테스트만 단독으로 실행

처음에는 실행에 성공

그런데 같은 `findItems()` 테스트를 다시 실행하면 테스트에 실패합니다.

테스트를 2번째 실행할 때 실패하는 이유는 `testcase` 데이터베이스에 접속해서 `item` 테이블의 데이터를 확인하면 알 수 있습니다.

처음 테스트를 실행할 때 저장한 데이터가 계속 남아있기 때문에 두번째 테스트에 영향을 주었습니다.

이 문제는 `save()` 같은 다른 테스트가 먼저 실행되고 나서 `findItems()` 를 실행할 때도 발생합니다. 다른 테스트에서 이미 데이터를 추가했기 때문에 테스트 데이터가 오염됩니다.

이 문제를 해결하려면 각각의 테스트가 끝날 때 마다 해당 테스트에서 추가한 데이터를 삭제해야 합니다.

### **테스트에서 매우 중요한 원칙**

- 테스트는 다른 테스트와 격리해야 함
- 테스트는 반복해서 실행할 수 있어야 함

물론 테스트가 끝날 때 마다 추가한 데이터에 `DELETE SQL`을 사용해도 되겠지만, 이 방법도 궁극적인 해결책은 아닙니다. 만약 테스트 과정에서 데이터를 이미 추가했는데, 테스트가 실행되는 도중에 예외가 발생하거나 애플리케이션이 종료되어 버려서 테스트 종료 시점에 `DELETE SQL`을 호출하지 못할 수도 있습니다. 그렇게 되면 다른 테스트와 격리되지 않습니다.

# 3. 테스트 - 데이터 롤백

### **트랜잭션과 롤백 전략**

테스트가 끝나고 나서 트랜잭션을 강제로 롤백해버리면 데이터가 깔끔하게 제거됩니다. 

테스트를 하면서 데이터를 이미 저장했는데, 중간에 테스트가 실패해서 롤백을 호출하지 못해도 괜찮습니다.

트랜잭션을 커밋하지 않았기 때문에 데이터베이스에 해당 데이터가 반영되지 않습니다. 이렇게 트랜잭션을 활용하면 테스트가 끝나고 나서 데이터를 깔끔하게 원래 상태로 되돌릴 수 있습니다.

예를 들어서 다음 순서와 같이 각각의 테스트 실행 직전에 트랜잭션을 시작하고, 각각의 테스트 실행 직후에 트랜잭션을 롤백해야 합니다. 그래야 다음 테스트에 데이터로 인한 영향을 주지 않습니다.

```sql
1. 트랜잭션 시작
2. 테스트 A 실행
3. 트랜잭션 롤백

4. 트랜잭션 시작
5. 테스트 B 실행
6. 트랜잭션 롤백
```

테스트는 각각의 테스트 실행 전 후로 동작하는 `@BeforeEach`, `@AfterEach`라는 편리한 기능을 제공합니다.

**테스트에 직접 트랜잭션 추가**

```java
@SpringBootTest
class ItemRepositoryTest {
  
    @Autowired
    ItemRepository itemRepository;
  
    //트랜잭션 관련 코드
    @Autowired
    PlatformTransactionManager transactionManager;
    TransactionStatus status;
  
    @BeforeEach
    void beforeEach() {
        //트랜잭션 시작
        status = transactionManager.getTransaction(new DefaultTransactionDefinition());
    }
  
    @AfterEach
    void afterEach() {
        //MemoryItemRepository 의 경우 제한적으로 사용
        if (itemRepository instanceof MemoryItemRepository) {
            ((MemoryItemRepository) itemRepository).clearStore();
        }
        //트랜잭션 롤백
        transactionManager.rollback(status);
    }
    //...
}
```

트랜잭션 관리자는 `PlatformTransactionManager`를 주입 받아서 사용합니다. 참고로 스프링 부트는 자동으로 적절한 트랜잭션 매니저를 스프링 빈으로 등록합니다.

`@BeforeEach`: 각각의 테스트 케이스를 실행하기 직전에 호출됩니다. 따라서 여기서 트랜잭션을 시작하면 각각의 테스트를 트랜잭션 범위 안에서 실행할 수 있습니다.

- `transactionManager.getTransaction(new DefaultTransactionDefinition())`로 트랜잭션을 시작합니다.

`@AfterEach`: 각각의 테스트 케이스가 완료된 직후에 호출됩니다. 따라서 여기서 트랜잭션을 롤백하면 데이터를 트랜잭션 실행 전 상태로 복구할 수 있습니다.

- `transactionManager.rollback(status)`로 트랜잭션을 롤백합니다.

테스트를 실행하기 전에 먼저 테스트에 영향을 주지 않도록 `testcase` 데이터베이스에 접근해서 기존 데이터를 깔끔하게 삭제합니다

**모든 ITEM 데이터 삭제** 

```sql
delete from item
```

**데이터가 모두 삭제되었는지 확인**

```sql
SELECT * FROM ITEM
```

**ItemRepositoryTest 실행** 

이제 `ItemRepositoryTest`의 테스트를 여러번 반복해서 실행해도 테스트가 성공하는 것을 확인할 수 있습니다.

# 4. 테스트 - @Transactional

스프링은 테스트 데이터 초기화를 위해 트랜잭션을 적용하고 롤백하는 방식을 `@Transactional` 애노테이션 하나로 깔끔하게 해결할 수 있습니다.

이전에 테스트에 트랜잭션과 롤백을 위해 추가했던 코드들을 주석 처리합시다.

`ItemRepositoryTest`

```java
@SpringBootTest
class ItemRepositoryTest {
  
    @Autowired
    ItemRepository itemRepository;
  
    //트랜잭션 관련 코드
    /*
    @Autowired
    PlatformTransactionManager transactionManager;
    TransactionStatus status;
    @BeforeEach
    void beforeEach() {
    //트랜잭션 시작
    status = transactionManager.getTransaction(new
    DefaultTransactionDefinition());
    }
    */
  
    @AfterEach
    void afterEach() {
        //MemoryItemRepository 의 경우 제한적으로 사용
        if (itemRepository instanceof MemoryItemRepository) {
            ((MemoryItemRepository) itemRepository).clearStore();
        }
        //트랜잭션 롤백
        //transactionManager.rollback(status);
    }
    //...
}
```

`ItemRepositoryTest` 테스트 코드에 스프링이 제공하는 `@Transactional`를 추가합니다.

`org.springframework.transaction.annotation.Transactional`

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional
@SpringBootTest
class ItemRepositoryTest {}
```

테스트를 실행하기 전에 테스트에 영향을 주지 않도록 `testcase` 데이터베이스에 접근해서 기존 데이터를 삭제합니다. 

**모든 ITEM 데이터 삭제** 

```sql
delete from item
```

**데이터가 모두 삭제되었는지 확인** 

```sql
SELECT * FROM ITEM
```

`ItemRepositoryTest` 실행 

이제 `ItemRepositoryTest`의 테스트를 실행하면 여러 번 반복해서 실행해도 테스트가 성공하는 것을 확인할 수 있습니다.

### **@Transactional 원리**

스프링이 제공하는 `@Transactional` 애노테이션은 로직이 성공적으로 수행되면 커밋하도록 동작합니다. 

그런데 `@Transactional` 애노테이션을 테스트에서 사용하면 아주 특별하게 동작합니다. 

`@Transactional`이 테스트에 있으면 스프링은 테스트를 트랜잭션 안에서 실행하고, 테스트가 끝나면 트랜잭션을 자동으로 롤백합니다.

**@Transactional 이 적용된 테스트 동작 방식**

![https://user-images.githubusercontent.com/52024566/186914460-dfff09d6-aa49-4874-ad2c-179bda2b2127.png](https://user-images.githubusercontent.com/52024566/186914460-dfff09d6-aa49-4874-ad2c-179bda2b2127.png)

1. 테스트에 `@Transactional` 애노테이션이 테스트 메서드나 클래스에 있으면 먼저 트랜잭션을 시작합니다.
2. 테스트 로직을 실행합니다. 테스트가 끝날 때 까지 모든 로직은 트랜잭션 안에서 수행됩니다.
    - 트랜잭션은 기본적으로 전파되기 때문에, 리포지토리에서 사용하는 `JdbcTemplate`도 같은 트랜잭션을 사용합니다.
3. 테스트 실행 중에 INSERT SQL 을 사용해서 `item1`, `item2`, `item3` 를 데이터베이스에 저장합니다.
    - 테스트가 리포지토리를 호출하고, 리포지토리는 `JdbcTemplate`을 사용해서 데이터를 저장합니다.
4. 검증을 위해서 SELECT SQL 로 데이터를 조회합니다. 여기서는 앞서 저장한 `item1`, `item2`, `item3`이 조회됩니다.
    - SELECT SQL도 같은 트랜잭션을 사용하기 때문에 저장한 데이터를 조회할 수 있습니다. 다른 트랜잭션에서는 해당 데이터를 확인할 수 없습니다.
    - 여기서 `assertThat()`으로 검증이 모두 끝납니다.
5. `@Transactional`이 테스트에 있으면 테스트가 끝날때 트랜잭션을 강제로 롤백합니다.
6. 롤백에 의해 앞서 데이터베이스에 저장한 `item1`, `item2`, `item3`의 데이터가 제거합니다.

> 참고 - 테스트 케이스의 메서드나 클래스에 @Transactional 을 직접 붙여서 사용할 때만 이렇게 동작합니다. 그리고 트랜잭션을 테스트에서 시작하기 때문에 서비스, 리포지토리에 있는 @Transactional 도 테스트에서 시작한 트랜잭션에 참여합니다. 
(이 부분은 뒤에 트랜잭션 전파에서 더 자세히 설명. 지금은 테스트에서 트랜잭션을 실행하면 테스트 실행이 종료될 때 까지 테스트가 실행하는 모든 코드가 같은 트랜잭션 범위에 들어간다고 이해하면 됨. 같은 범위라는 뜻은 쉽게 이야기해서 같은 트랜잭션을 사용한다는 뜻이다. 그리고 같은 트랜잭션을 사용한다는 것은 같은 커넥션을 사용한다는 뜻이기도 하다.)
> 

### **정리**

테스트가 끝난 후 개발자가 직접 데이터를 삭제하지 않아도 되는 편리함을 제공합니다.

테스트 실행 중에 데이터를 등록하고 중간에 테스트가 강제로 종료되어도 걱정이 없습니다. 이 경우 트랜잭션을 커밋하지 않기 때문에, 데이터는 자동으로 롤백합니다. (보통 데이터베이스 커넥션이 끊어지면 자동으로 롤백)

트랜잭션 범위 안에서 테스트를 진행하기 때문에 동시에 다른 테스트가 진행되어도 서로 영향을 주지 않는 장점이 있습니다.

`@Transactional` 덕분에 아주 편리하게 다음 원칙을 지킬 수 있음

- 테스트는 다른 테스트와 격리
- 테스트는 반복해서 실행할 수 있음

### **강제로 커밋하기 - @Commit**

`@Transactional`을 테스트에서 사용하면 테스트가 끝나면 바로 롤백되기 때문에 테스트 과정에서 저장한 모든 데이터가 사라집니다. 당연히 이렇게 되어야 하지만, 정말 가끔은 데이터베이스에 데이터가 잘 보관되었는지 최종 결과를 눈으로 확인하고 싶을 때도 있겠죠. 이럴 때는 다음과 같이 `@Commit`을 클래스 또는 메서드에 붙이면 테스트 종료 후 롤백 대신 커밋이 호출됩니다. 참고로 `@Rollback(value = false)`를 사용해도 됩니다.

```java
import org.springframework.test.annotation.Commit;

@Commit
@Transactional
@SpringBootTest
class ItemRepositoryTest {}
```

# 5. 테스트 - 임베디드 모드 DB

**테스트 케이스를 실행하기 위해서 별도의 데이터베이스를 설치하고, 운영하는 것은 상당히 번잡한 작업입니다.** 

단순히 테스트를 검증할 용도로만 사용하기 때문에 테스트가 끝나면 데이터베이스의 데이터를 모두 삭제해도 됩니다. 더 나아가서 테스트가 끝나면 데이터베이스 자체를 제거해도 됩니다.

### **임베디드 모드**

H2 데이터베이스는 자바로 개발되어 있고, JVM 안에서 메모리 모드로 동작하는 특별한 기능을 제공합니다. 그래서 애플리케이션을 실행할 때 H2 데이터베이스도 해당 JVM 메모리에 포함해서 함께 실행할 수 있습니다. 

DB를 애플리케이션에 내장해서 함께 실행한다고 해서 임베디드 모드(Embedded mode)라 합니다. 물론 애플리케이션이 종료되면 임베디드 모드로 동작하는 H2 데이터베이스도 함께 종료되고, 데이터도 모두 사라집니다. 쉽게 이야기해서 애플리케이션에서 자바 메모리를 함께 사용하는 라이브러리처럼 동작하는 것입니다.

### **임베디드 모드 직접 사용**

`ItemServiceApplication` - 추가

```java
@Slf4j
//@Import(MemoryConfig.class)
//@Import(JdbcTemplateV1Config.class)
//@Import(JdbcTemplateV2Config.class)
@Import(JdbcTemplateV3Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {
  
    public static void main(String[] args) {
        SpringApplication.run(ItemServiceApplication.class, args);
    }
  
    @Bean
    @Profile("local")
    public TestDataInit testDataInit(ItemRepository itemRepository) {
        return new TestDataInit(itemRepository);
    }
  
    @Bean
    @Profile("test")
    public DataSource dataSource() {
        log.info("메모리 데이터베이스 초기화");
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl("jdbc:h2:mem:db;DB_CLOSE_DELAY=-1");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        return dataSource;
    }
}
```

`@Profile("test")`

- 프로필이 test 인 경우에만 데이터소스를 스프링 빈으로 등록합니다.
- 테스트 케이스에서만 이 데이터소스를 스프링 빈으로 등록해서 사용하겠다는 의미입니다.

`dataSource()`

- `jdbc:h2:mem:db`: 이 부분이 중요합니다. 데이터소스를 만들때 이렇게만 적으면 임베디드 모드(메모리 모드)로 동작하는 H2 데이터베이스를 사용할 수 있습니다.
- `DB_CLOSE_DELAY=-1`: 임베디드 모드에서는 데이터베이스 커넥션 연결이 모두 끊어지면 데이터베이스도 종료되는데, 그것을 방지하는 설정입니다.
- 이 데이터소스를 사용하면 메모리 DB를 사용할 수 있습니다.

### **실행**

이제 `ItemRepositoryTest` 테스트를 메모리 DB를 통해 실행 테스트를 실행하면 새로 등록한 메모리 DB에 접근하는 데이터소스를 사용합니다. 

확실하게 하기 위해서 H2 데이터베이스 서버를 종료하고 실행합시다.

**실행 결과**

```java
org.springframework.jdbc.BadSqlGrammarException: PreparedStatementCallback; 
bad SQL grammar []; 
nested exception is org.h2.jdbc.JdbcSQLSyntaxErrorException:
Table "ITEM" not found; SQL statement:
insert into item (item_name, price, quantity) values (?, ?, ?) [42102-200]
...
Caused by: org.h2.jdbc.JdbcSQLSyntaxErrorException: Table "ITEM" not found; SQL statement:
insert into item (item_name, price, quantity) values (?, ?, ?) [42102-200]
```

그런데 막상 실행해보면 다음과 같은 오류를 확인할 수 있습니다.

(참고로 오류는 항상 아래에 있는 오류 정보가 더 근본 원인에 가까운 오류 로그입니다.)

`Table "ITEM" not found` 부분이 핵심입니다. 데이터베이스 테이블이 없다는 뜻이죠. 생각해보면 메모리 DB에는 아직 테이블을 만들지 않았습니다.

테스트를 실행하기 전에 테이블을 먼저 생성해야 합니다. 수동으로 할 수도 있지만 스프링 부트는 이 문제를 해결할 아주 편리한 기능을 제공합니다.

### **스프링 부트 - 기본 SQL 스크립트를 사용해서 데이터베이스를 초기화하는 기능**

메모리 DB는 애플리케이션이 종료될 때 함께 사라지기 때문에, 애플리케이션 실행 시점에 데이터베이스 테이블도 새로 생성해야 합니다. JDBC나 JdbcTemplate를 직접 사용해서 테이블을 생성하는 DDL을 호출해도 되지만, 스프링 부트는 SQL 스크립트를 실행해서 애플리케이션 로딩 시점에 데이터베이스를 초기화하는 기능을 제공합니다.

`src/test/resources/schema.sql`

```sql
drop table if exists item CASCADE;
create table item
(
    id bigint generated by default as identity,
    item_name varchar(10),
    price integer,
    quantity integer,
    primary key (id)
);
```

> 참고 - SQL 스크립트를 사용해서 데이터베이스를 초기화하는 자세한 방법은 다음 스프링 부트 공식 메뉴얼을 참고합시다. 
https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.datainitialization.using-basic-sql-scripts
> 

### **실행**

`ItemRepositoryTest`를 실행해보면 테스트가 정상 수행되는 것을 확인할 수 있습니다.

**로그 확인** 

기본 SQL 스크립트가 잘 실행되는지 로그로 확인하려면 다음이 추가되어 있는지 확인합시다.

`src/test/resources/application.properties`

```sql
#schema.sql
logging.level.org.springframework.jdbc=debug
```

**SQL 스트립트 로그**

```sql
..init.ScriptUtils : 0 returned as update count for SQL: drop table if
exists item CASCADE
..init.ScriptUtils : 0 returned as update count for SQL: create table item
( id bigint generated by default as identity, item_name varchar(10), price
integer, quantity integer, primary key (id) )
```



# 6. 테스트 - 스프링 부트와 임베디드 모드

스프링 부트는 개발자에게 정말 많은 편리함을 제공하는데, 임베디드 데이터베이스에 대한 설정도 기본으로 제공합니다. 

스프링 부트는 데이터베이스에 대한 별다른 설정이 없으면 임베디드 데이터베이스를 사용합니다.

앞서 직접 설정했던 메모리 DB용 데이터소스를 주석처리합시다.

`ItemServiceApplication`

```java
@Slf4j
//@Import(MemoryConfig.class)
//@Import(JdbcTemplateV1Config.class)
//@Import(JdbcTemplateV2Config.class)
@Import(JdbcTemplateV3Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ItemServiceApplication.class, args);
    }
  
    @Bean
    @Profile("local")
    public TestDataInit testDataInit(ItemRepository itemRepository) {
        return new TestDataInit(itemRepository);
    }
  
/*
    @Bean
    @Profile("test")
    public DataSource dataSource() {
        log.info("메모리 데이터베이스 초기화");
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("org.h2.Driver");
        dataSource.setUrl("jdbc:h2:mem:db;DB_CLOSE_DELAY=-1");
        dataSource.setUsername("sa");
        dataSource.setPassword("");
        return dataSource;
    }
*/
}
```

그리고 테스트에서 데이터베이스에 접근하는 설정 정보도 주석처리합시다. 

### **test - application.properties**

`src/test/resources/application.properties`

```java
spring.profiles.active=test
#spring.datasource.url=jdbc:h2:tcp://localhost/~/testcase
#spring.datasource.username=sa

#jdbcTemplate sql log
logging.level.org.springframework.jdbc=debug
```

`spring.datasource.url`, `spring.datasource.username`를 사용하지 않도록 `#`을 사용해서 주석처리합시다.

이렇게 하면 데이터베이스에 접근하는 모든 설정 정보가 사라집니다. 이렇게 별다른 정보가 없으면 스프링 부트는 임베디드 모드로 접근하는 데이터소스(`DataSource`)를 만들어서 제공합니다.

### **실행**

`ItemRepositoryTest`를 실행해보면 테스트가 정상 수행되는 것을 확인할 수 있습니다.

참고로 로그를 보면 다음 부분을 확인할 수 있는데 `jdbc:h2:mem` 뒤에 임의의 데이터베이스 이름이 들어가 있습니다.

이것은 혹시라도 여러 데이터소스가 사용될 때 같은 데이터베이스를 사용하면서 발생하는 충돌을 방지하기 위해 스프링 부트가 임의의 이름을 부여한 것입니다.

```java
conn0: url=jdbc:h2:mem:d8fb3a29-caf7-4b37-9b6c-b0eed9985454
```

임베디드 데이터베이스 이름을 스프링 부트가 기본으로 제공하는 `jdbc:h2:mem:testdb`로 고정하고 싶으면 `application.properties`에 다음 설정을 추가하면 됩니다.

```java
spring.datasource.generate-unique-name=false
```

> 참고 - 임베디드 데이터베이스에 대한 스프링 부트의 더 자세한 설정은 다음 공식 메뉴얼을 참고합시다.
https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.sql.datasource.embedded
>

# ==== 3. 데이터 접근 기술 - MyBatis ====

# 1. MyBatis 소개

MyBatis 는 앞서 설명한 JdbcTemplate보다 더 많은 기능을 제공하는 SQL Mapper 입니다. 

기본적으로 JdbcTemplate 이 제공하는 대부분의 기능을 제공합니다. 

JdbcTemplate 과 비교해서 MyBatis 의 가장 매력적인 점은 SQL을 XML에 편리하게 작성할 수 있고 또 동적 쿼리를 매우 편리하게 작성할 수 있다는 점입니다.

**JdbcTemplate - SQL 여러줄**

```java
String sql = "update item " +
  "set item_name=:itemName, price=:price, quantity=:quantity " +
  "where id=:id";
```

**MyBatis - SQL 여러줄**

```xml
<update id="update">
    update item
    set item_name=#{itemName},
        price=#{price},
        quantity=#{quantity}
    where id = #{id}
</update>
```

MyBatis 는 XML 에 작성하기 때문에 라인이 길어져도 문자 더하기에 대한 불편함이 없습니다.

**JdbcTemplate - 동적 쿼리**

```java
String sql = "select id, item_name, price, quantity from item";
//동적 쿼리
if (StringUtils.hasText(itemName) || maxPrice != null) {
    sql += " where";
}

boolean andFlag = false;
if (StringUtils.hasText(itemName)) {
    sql += " item_name like concat('%',:itemName,'%')";
    andFlag = true;
}

if (maxPrice != null) {
    if (andFlag) {
        sql += " and";
    }
    sql += " price <= :maxPrice";
}

log.info("sql={}", sql);
return template.query(sql, param, itemRowMapper());
```

**MyBatis - 동적 쿼리**

```xml
<select id="findAll" resultType="Item">
    select id, item_name, price, quantity
    from item
    <where>
        <if test="itemName != null and itemName != ''">
            and item_name like concat('%',#{itemName},'%')
        </if>
        <if test="maxPrice != null">
            and price &lt;= #{maxPrice}
        </if>
    </where>
</select>
```

JdbcTemplate 은 자바 코드로 직접 동적 쿼리를 작성해야 하는 반면에 MyBatis 는 동적 쿼리를 매우 편리하게 작성할 수 있는 다양한 기능들을 제공합니다.

**설정의 장단점** 

JdbcTemplate 은 스프링에 내장된 기능이고, 별도의 설정없이 사용할 수 있는 반면에 MyBatis 는 약간의 설정이 필요합니다.

### **정리**

프로젝트에서 동적 쿼리와 복잡한 쿼리가 많다면 MyBatis 를 사용하고, 단순한 쿼리들이 많으면 JdbcTemplate 을 선택해서 사용하면 됩니다. 물론 둘을 함께 사용해도 되지만 MyBatis 를 선택했다면 굳이 JdbcTemplate 을 함께 사용할 필요 없이 MyBatis 만으로 충분합니다.

> 참고 - 강의에서는 MyBatis의 기능을 하나하나를 자세하게 다루지는 않습니다. MyBatis 를 왜 사용하는지, 그리고 주로 사용하는 기능 위주로 다룰 것입니다. 그래도 이 강의를 듣고 나면 MyBatis 로 개발을 할 수 있게 되고 추가로 필요한 내용을 공식 사이트에서 찾아서 사용할 수 있게 될 것입니다. 
MyBatis는 기능도 단순하고 또 공식 사이트가 한글로 잘 번역되어 있어서 원하는 기능을 편리하게 찾아볼 수 있습니다.
> 
> 
> 공식 사이트 https://mybatis.org/mybatis-3/ko/index.html
>

# 2. MyBatis 설정

`mybatis-spring-boot-starter` 라이브러리를 사용하면 MyBatis 를 스프링과 통합하고, 설정도 아주 간단히 할 수 있습니다.

`build.gradle`에 다음 의존 관계를 추가합시다.

```groovy
//MyBatis 추가
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
```

참고로 뒤에 버전 정보가 붙는 이유는 스프링 부트가 버전을 관리해주는 공식 라이브러리가 아니기 때문입니다. 

만약 스프링 부트가 버전을 관리해주는 경우에는 버전 정보를 붙이지 않아도 최적의 버전을 자동으로 찾아줍니다.

**build.gradle - 의존관계 전체**

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    
    //JdbcTemplate 추가
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    //MyBatis 추가
    implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
    
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

다음과 같은 라이브러리가 추가될 수 있습니다.

- `mybatis-spring-boot-starter`: MyBatis 를 스프링 부트에서 편리하게 사용할 수 있게 시작하는 라이브러리입니다.
- `mybatis-spring-boot-autoconfigure`: MyBatis 와 스프링 부트 설정 라이브러리입니다.
- `mybatis-spring`: MyBatis와 스프링을 연동하는 라이브러리
- `mybatis`: MyBatis 라이브러리

**설정** 

`application.properties`에 다음 설정을 추가합시다. `#MyBatis`를 참고

> **주의 -** 웹 애플리케이션을 실행하는 `main` 과 테스트를 실행하는 `test` 각각의 위치의 `application.properties`를 모두 수정해주어야 합니다.
> 

**main - application.properties**

```groovy
spring.profiles.active=local
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa

logging.level.org.springframework.jdbc=debug

#MyBatis
mybatis.type-aliases-package=hello.itemservice.domain
mybatis.configuration.map-underscore-to-camel-case=true
logging.level.hello.itemservice.repository.mybatis=trace
```

**test - application.properties**

```groovy
spring.profiles.active=test
#spring.datasource.url=jdbc:h2:tcp://localhost/~/testcase
#spring.datasource.username=sa

logging.level.org.springframework.jdbc=debug

#MyBatis
mybatis.type-aliases-package=hello.itemservice.domain
mybatis.configuration.map-underscore-to-camel-case=true
logging.level.hello.itemservice.repository.mybatis=trace
```

`mybatis.type-aliases-package`

- MyBatis 에서 타입 정보를 사용할 때는 패키지 이름을 적어주어야 하는데, 여기에 명시하면 패키지 이름을 생략할 수 있습니다.
- 지정한 패키지와 그 하위 패키지가 자동으로 인식됩니다.
- 여러 위치를 지정하려면 `,`, `;`로 구분하면 됩니다.

`mybatis.configuration.map-underscore-to-camel-case`

- JdbcTemplate 의 `BeanPropertyRowMapper`에서 처럼 언더바를 카멜로 자동 변경해주는 기능을 활성화시키는 설정입니다. 바로 다음에 설명하는 관례의 불일치 내용에서 자세히 설명합니다.

`logging.level.hello.itemservice.repository.mybatis=trace`

- MyBatis 에서 실행되는 쿼리 로그를 확인할 수 있습니다.

### **관례의 불일치**

자바 객체에는 주로 카멜(`camelCase`) 표기법을 사용합니다.

반면에 관계형 데이터베이스에서는 주로 언더스코어를 사용하는 `snake_case`표기법을 사용합니다. 

이렇게 관례로 많이 사용하다 보니 `map-underscore-to-camel-case` 기능을 활성화 하면 언더스코어 표기법을 카멜로 자동 변환합니다. 따라서 DB에서 `select item_name`으로 조회해도 객체의 `itemName`( `setItemName()`) 속성에 값이 정상 입력됩니다. 

정리하면 해당 옵션을 켜면 `snake_case`는 자동으로 해결되니 그냥 두면 되고, 컬럼 이름과 객체 이름이 완전히 다른 경우에는 조회 SQL에서 별칭을 사용하면 됩니다.

**예)**

- DB `select item_name`
- 객체 `name`
- 별칭을 통한 해결방안: `select item_name as name`


# 3. MyBatis 적용1 - 기본

`ItemMapper`

```groovy
@Mapper
public interface ItemMapper {
  
    void save(Item item);
  
    void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto updateParam);
  
    Optional<Item> findById(Long id);
  
    List<Item> findAll(ItemSearchCond itemSearch);
}
```

마이바티스 매핑 XML을 호출해주는 매퍼 인터페이스입니다.

이 인터페이스에는 `@Mapper` 애노테이션을 붙여주어야 MyBatis 에서 인식할 수 있습니다.

이 인터페이스의 메서드를 호출하면 다음에 보이는 xml 의 해당 SQL을 실행하고 결과를 반환합니다.

이제 같은 위치에 실행할 SQL 이 있는 XML 매핑 파일을 생성합니다. 

참고로 자바 코드가 아니기 때문에 `src/main/resources` 하위에 만들되, 패키지 위치는 맞추어 주어야 합니다.

`src/main/resources/hello/itemservice/repository/mybatis/ItemMapper.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">
    <insert id="save" useGeneratedKeys="true" keyProperty="id">
        insert into item (item_name, price, quantity)
        values (#{itemName}, #{price}, #{quantity})
    </insert>
  
    <update id="update">
        update item
        set item_name=#{updateParam.itemName},
            price=#{updateParam.price},
            quantity=#{updateParam.quantity}
        where id = #{id}
    </update>
  
    <select id="findById" resultType="Item">
        select id, item_name, price, quantity
        from item
        where id = #{id}
    </select>
  
    <select id="findAll" resultType="Item">
        select id, item_name, price, quantity
        from item
        <where>
            <if test="itemName != null and itemName != ''">
                and item_name like concat('%',#{itemName},'%')
            </if>
            <if test="maxPrice != null">
                and price &lt;= #{maxPrice}
            </if>
        </where>
    </select>
</mapper>
```

- `namespace` : 앞서 만든 매퍼 인터페이스를 지정합니다.

> **주의 -**  경로와 파일 이름에 주의해야 합니다.
> 

> 참고 - XML 파일 경로 수정하기 XML 파일을 원하는 위치에 두고 싶으면 application.properties 에 다음과 같이 설정하면 됩니다.
> 
> 
> ```xml
> mybatis.mapper-locations=classpath:mapper/**/*.xml
> ```
> 
> 이렇게 하면 `resources/mapper` 를 포함한 그 하위 폴더에 있는 XML 을 XML 매핑 파일로 인식합니다. 이 경우 파일 이름은 자유롭게 설정해도 됩니다.
> 참고로 테스트의 `application.properties` 파일도 함께 수정해야 테스트를 실행할 때 인식할 수 있습니다.
> 

insert - `save`

```xml
void save(Item item);

<insert id="save" useGeneratedKeys="true" keyProperty="id">
    insert into item (item_name, price, quantity)
    values (#{itemName}, #{price}, #{quantity})
</insert>
```

Insert SQL은 `<insert>`를 사용합니다.

`id`에는 매퍼 인터페이스에 설정한 메서드 이름을 지정합니다. 여기서는 메서드 이름이 `save()`이므로 `save`로 지정하였습니다.

파라미터는 `#{}` 문법을 사용합니다. 그리고 매퍼에서 넘긴 객체의 프로퍼티 이름을 기입해야 합니다.

`#{}` 문법을 사용하면 `PreparedStatement`를 사용하여 JDBC의 `?` 를 치환한다 생각하면 됩니다.

`useGeneratedKeys`는 데이터베이스가 키를 생성해 주는 IDENTITY 전략일 때 사용합니다. `keyProperty`는 생성되는 키의 속성 이름을 지정합니다. Insert 가 끝나면 `item` 객체의 `id` 속성에 생성된 값이 입력됩니다.

update - `update`

```xml
import org.apache.ibatis.annotations.Param;

void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto
updateParam);

<update id="update">
    update item
    set item_name=#{updateParam.itemName},
        price=#{updateParam.price},
        quantity=#{updateParam.quantity}
    where id = #{id}
</update>
```

Update SQL은 `<update>`를 사용합니다.

여기서는 파라미터가 `Long id`, `ItemUpdateDto updateParam`으로 2개입니다. 

파라미터가 1개만 있으면 `@Param`을 지정하지 않아도 되지만, 파라미터가 2개 이상이면 `@Param`으로 이름을 지정해서 파라미터를 구분해야 합니다.

select - `findById`

```xml
Optional<Item> findById(Long id);

<select id="findById" resultType="Item">
    select id, item_name, price, quantity
    from item
    where id = #{id}
</select>
```

Select SQL은 `<select>`를 사용합니다.

`resultType`은 반환 타입을 명시합니다. 여기서는 결과를 `Item` 객체에 매핑하고 있습니다.

앞서 `application.properties 에 mybatis.type-aliasespackage=hello.itemservice.domain` 속성을 지정한 덕분에 모든 패키지 명을 다 적지는 않아도 됩니다. 만약 속성을 지정하지 않았다면 모든 패키지 명을 다 적어야 합니다.

JdbcTemplate 의 `BeanPropertyRowMapper`처럼 SELECT SQL 의 결과를 편리하게 객체로 바로 변환합니다.

- `mybatis.configuration.map-underscore-to-camel-case=true` 속성을 지정한 덕분에 언더스코어를 카멜 표기법으로 자동으로 처리합니다. (`item_name` → `itemName`)

자바 코드에서 반환 객체가 하나이면 `Item`, `Optional<Item>`과 같이 사용하면 되고, 반환 객체가 하나 이상이면 컬렉션을 사용하면 됩니다. 주로 `List`를 사용합니다.

select - `findAll`

```xml
List<Item> findAll(ItemSearchCond itemSearch);

<select id="findAll" resultType="Item">
    select id, item_name, price, quantity
    from item
    <where>
        <if test="itemName != null and itemName != ''">
            and item_name like concat('%',#{itemName},'%')
        </if>
        <if test="maxPrice != null">
            and price &lt;= #{maxPrice}
        </if>
    </where>
</select>
```

Mybatis는 `<where>`, `<if>` 같은 동적 쿼리 문법을 통해 편리한 동적 쿼리를 지원합니다.

`<if>`는 해당 조건이 만족하면 구문을 추가합니다.

`<where>`은 적절하게 `where` 문장을 생성합니다.

- 예제에서 `<if>`가 모두 실패하게 되면 SQL `where`를 만들지 않음
- 예제에서 `<if>`가 하나라도 성공하면 처음 나타나는 `and`를 `where`로 변환

### **XML 특수문자**

가격을 비교하는 조건에서 `and price &lt;= #{maxPrice}` 여기에 보면 `<=`를 사용하지 않고 `&lt;=`를 사용합니다. 

그 이유는 XML에서는 데이터 영역에 `<`, `>` 같은 특수 문자를 사용할 수 없기 때문입니다. 이유는 간단한데, XML에서 TAG가 시작하거나 종료할 때 `<`, `>` 와 같은 특수문자를 사용하기 때문이죠.

`< : &lt;
> : &gt;
& : &amp;`

다른 해결 방안으로는 XML에서 지원하는 CDATA 구문 문법을 사용할 수 있습니다. 이 구문 안에서는 특수문자를 사용할 수 있는 대신 이 구문 안에서는 XML TAG 가 단순 문자로 인식되기 때문에 `<if>`, `<where>` 등이 적용되지 않습니다.

**XML CDATA 사용**

```xml
<select id="findAll" resultType="Item">
    select id, item_name, price, quantity
    from item
    <where>
        <if test="itemName != null and itemName != ''">
            and item_name like concat('%',#{itemName},'%')
        </if>
        <if test="maxPrice != null">
            <![CDATA[
            and price <= #{maxPrice}
            ]]>
        </if>
    </where>
</select>
```

특수문자와 CDATA 각각 상황에 따른 장단점이 있으므로 원하는 방법을 그때그때 선택하면 됩니다.

# 4. MyBatis 적용2 - 설정과 실행

`MyBatisItemRepository`

```java
@Repository
@RequiredArgsConstructor
public class MyBatisItemRepository implements ItemRepository {
  
    private final ItemMapper itemMapper;
  
    @Override
    public Item save(Item item) {
        itemMapper.save(item);
        return item;
    }
  
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        itemMapper.update(itemId, updateParam);
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        return itemMapper.findById(id);
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        return itemMapper.findAll(cond);
    }
}
```

`ItemRepository`를 구현해서 `MyBatisItemRepository`를 개발하겠습니다.

`MyBatisItemRepository`는 단순히 `ItemMapper`에 기능을 위임하였습니다.

`MyBatisConfig`

```java
@Configuration
@RequiredArgsConstructor
public class MyBatisConfig {
  
    private final ItemMapper itemMapper;
  
    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
  
    @Bean
    public ItemRepository itemRepository() {
        return new MyBatisItemRepository(itemMapper);
    }
}
```

`MyBatisConfig`는 `ItemMapper`를 주입받고, 필요한 의존관계를 생성합니다.

`ItemServiceApplication` - 변경

```java
@Slf4j
//@Import(MemoryConfig.class)
//@Import(JdbcTemplateV1Config.class)
//@Import(JdbcTemplateV2Config.class)
//@Import(JdbcTemplateV3Config.class)
@Import(MyBatisConfig.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {}
```

`@Import(MyBatisConfig.class)`: 앞서 설정한 `MyBatisConfig.class`를 사용하도록 설정했습니다.

### **테스트 실행**

`ItemRepositoryTest`를 통해서 리포지토리가 정상 동작하는지 확인해봅시다. 테스트가 모두 성공해야 합니다.

### **애플리케이션 실행**

`ItemServiceApplication`를 실행해서 애플리케이션이 정상 동작하는지 확인해봅시다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9a15d6d7-3957-48ff-a36b-bc5447faee8e/Untitled.png)

> **주의 -** H2 데이터베이스 서버를 먼저 실행해야 함
> 

# 5. MyBatis 적용3 - 분석

`ItemMapper` 인터페이스

```java
@Mapper
public interface ItemMapper {
  void save(Item item);
  
  void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto updateParam);
  
  List<Item> findAll(ItemSearchCond itemSearch);
  
  Optional<Item> findById(Long id);
}
```

그런데 `ItemMapper` 매퍼 인터페이스의 구현체가 없는데 어떻게 동작한 것일까요? 

이 부분은 MyBatis 스프링 연동 모듈에서 자동으로 처리되는 것입니다.

### **설정 원리**

![https://user-images.githubusercontent.com/52024566/187686571-ba8fa9fa-4732-469c-8487-e16398d329cf.png](https://user-images.githubusercontent.com/52024566/187686571-ba8fa9fa-4732-469c-8487-e16398d329cf.png)

1. 애플리케이션 로딩 시점에 MyBatis 스프링 연동 모듈은 `@Mapper`가 붙어있는 인터페이스를 조사합니다.
2. 해당 인터페이스가 발견되면 동적 프록시 기술을 사용해서 `ItemMapper` 인터페이스의 구현체를 생성합니다.
3. 생성된 구현체를 스프링 빈으로 등록합니다.

실제 동적 프록시 기술이 사용되었는지 간단히 확인해봅시다.

`MyBatisItemRepository` - 로그 추가

```java
@Slf4j
@Repository
@RequiredArgsConstructor
public class MyBatisItemRepository implements ItemRepository {
  
    private final ItemMapper itemMapper;
  
    @Override
    public Item save(Item item) {
        log.info("itemMapper class={}", itemMapper.getClass());
        itemMapper.save(item);
        return item;
    }
}
```

실행해서 주입 받은 `ItemMapper` 의 클래스를 출력해봅시다.

**실행 결과**

```java
itemMapper class=class jdk.proxy.$Proxy66
```

출력해보면 JDK 동적 프록시가 적용된 것을 확인할 수 있습니다.

### **매퍼 구현체**

마이바티스 스프링 연동 모듈이 만들어주는 `ItemMapper`의 구현체 덕분에 인터페이스 만으로 편리하게 XML의 데이터를 찾아서 호출할 수 있습니다.

원래 마이바티스를 사용하려면 더 번잡한 코드를 거쳐야 하는데, 이런 부분을 인터페이스 하나로 매우 깔끔하고 편리하게 사용할 수 있습니다.

매퍼 구현체는 예외 변환까지 처리해줍니다. MyBatis 에서 발생한 예외를 스프링 예외 추상화인 `DataAccessException`에 맞게 변환해서 반환해줍니다. JdbcTemplate 이 제공하는 예외 변환 기능을 여기서도 제공한다고 이해하면 됩니다.

### **정리**

매퍼 구현체 덕분에 마이바티스를 스프링에 편리하게 통합해서 사용할 수 있습니다.

매퍼 구현체를 사용하면 스프링 예외 추상화도 함께 적용할 수 있습니다.

마이바티스 스프링 연동 모듈이 많은 부분을 자동으로 설정해주는데, 데이터베이스 커넥션, 트랜잭션과 관련된 기능도 마이바티스와 함께 연동하고, 동기화됩니다.

> 참고 - 마이바티스 스프링 연동 모듈이 자동으로 등록해주는 부분은 MybatisAutoConfiguration 클래스를 참고하면 됩니다.
>

# 6. MyBatis 기능 정리1 -동적 쿼리

- MyBatis 공식 메뉴얼: https://mybatis.org/mybatis-3/ko/index.html
- MyBatis 스프링 공식 메뉴얼: https://mybatis.org/spring/ko/index.html

### **동적 SQL**

**마이바티스가 제공하는 최고의 기능이자 마이바티스를 사용하는 이유는 바로 동적 SQL 기능 때문입니다.** 

동적 쿼리를 위해 제공되는 기능은 다음과 같습니다.

- `if`
- `choose (when, otherwise)`
- `trim (where, set)`
- `foreach`

**if**

```xml
<select id="findActiveBlogWithTitleLike"
    resultType="Blog">
    SELECT * FROM BLOG
    WHERE state = ‘ACTIVE’
    <if test="title != null">
      AND title like #{title}
    </if>
</select>
```

해당 조건에 따라 값을 추가할지 말지 판단합니다.

내부의 문법은 [OGNL](https://ko.wikipedia.org/wiki/OGNL)을 사용합니다.

**choose, when, otherwise**

```xml
<select id="findActiveBlogLike"
    resultType="Blog">
    SELECT * FROM BLOG WHERE state = ‘ACTIVE’
    <choose>
      <when test="title != null">
        AND title like #{title}
      </when>
      <when test="author != null and author.name != null">
        AND author_name like #{author.name}
      </when>
      <otherwise>
        AND featured = 1
      </otherwise>
    </choose>
</select>
```

자바의 `switch` 구문과 유사한 구문입니다.

**trim, where, set**

```xml
<select id="findActiveBlogLike"
resultType="Blog">
    SELECT * FROM BLOG
    WHERE
    <if test="state != null">
      state = #{state}
    </if>
    <if test="title != null">
      AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
      AND author_name like #{author.name}
    </if>
</select>
```

이 예제의 문제점은 문장을 모두 만족하지 않을 때 발생합니다.

```sql
SELECT * FROM BLOG
WHERE
```

이렇게 SQL 문이 WHERE 로 끝나버립니다.

`title`만 만족할 때도 문제가 발생합니다.

```sql
SELECT * FROM BLOG
WHERE
AND title like ‘someTitle’
```

결국 `WHERE` 문을 언제 넣어야 할지 상황에 따라서 동적으로 달라지는 문제가 있습니다. 

이 때 `<where>`를 사용하면 이런 문제를 해결할 수 있습니다.

```xml
<select id="findActiveBlogLike"
    resultType="Blog">
    SELECT * FROM BLOG
    <where>
    <if test="state != null">
        state = #{state}
    </if>
    <if test="title != null">
        AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
        AND author_name like #{author.name}
    </if>
    </where>
</select>
```

`<where>`는 문장이 없으면 `where`를 추가하지 않습니다. 문장이 있으면 `where`를 추가합니다. 만약 `and`가 먼저 시작된다면 `and`를 삭제하게 됩니다.

다음과 같이 `trim`이라는 기능으로 사용해도 이렇게 정의하면 `<where>`와 같은 기능을 수행합니다.

```xml
<trim prefix="WHERE" prefixOverrides="AND |OR ">
...
</trim>
```

**foreach**

```xml
<select id="selectPostIn" resultType="domain.blog.Post">
    SELECT *
    FROM POST P
    <where>
      <foreach item="item" index="index" collection="list"
        open="ID in (" separator="," close=")" nullable="true">
        #{item}
      </foreach>
    </where>
</select>
```

컬렉션을 반복 처리할 때 사용합니다. `where in (1,2,3,4,5,6)`와 같은 문장을 쉽게 완성할 수 있습니다.

파라미터로 `List`를 전달합니다.

> 참고 - 동적 쿼리에 대한 자세한 내용은 다음을 참고합시다. 
https://mybatis.org/mybatis-3/ko/dynamic-sql.html
>

# 7. MyBatis 기능 정리2 - 기타 기능

### **애노테이션으로 SQL 작성**

다음과 같이 XML 대신에 애노테이션에 SQL을 작성할 수 있습니다.

```java
@Select("select id, item_name, price, quantity from item where id=#{id}")
Optional<Item> findById(Long id);
```

`@Insert`, `@Update`, `@Delete`, `@Select` 기능이 제공됩니다.

이 경우 XML에는 `<select id="findById"> ~ </select>`는 제거해야 합니다.

동적 SQL이 해결되지 않으므로 간단한 경우에만 사용하는 것입니다.

> 참고: https://mybatis.org/mybatis-3/ko/java-api.html
> 

### **문자열 대체(String Substitution)**

`#{}` 문법은 `‘?’` 를 넣고 파라미터를 바인딩하는 `PreparedStatement`를 사용합니다. 

파라미터 바인딩이 아니라 문자 그대로를 처리하고 싶은 경우 `${}` 를 사용합니다.

`ORDER BY ${columnName}`

```java
@Select("select * from user where ${column} = #{value}")
User findByColumn(@Param("column") String column, @Param("value") String value);
```

> **주의 -** `${}` 를 사용하면 SQL 인젝션 공격을 당할 수 있으므로 가급적 사용하면 안 됩니다. 사용하더라도 매우 주의깊게 사용해야 합니다.
> 

### **재사용 가능한 SQL 조각**

`<sql>`을 사용하면 SQL 코드를 재사용 할 수 있습니다.

```xml
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
```

```xml
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```

`<include>`를 통해서 `<sql>` 조각을 찾아서 사용할 수 있습니다.

```xml
<sql id="sometable">
  ${prefix}Table
</sql>

<sql id="someinclude">
  from
    <include refid="${include_target}"/>
</sql>

<select id="select" resultType="map">
  select
    field1, field2, field3
  <include refid="someinclude">
    <property name="prefix" value="Some"/>
    <property name="include_target" value="sometable"/>
  </include>
</select>
```

프로퍼티 값을 전달할 수 있고, 해당 값은 내부에서 사용할 수 있습니다.

### **Result Maps**

컬럼명과 객체의 프로퍼티 명이 다를 경우, 다음과 같이 별칭(`as`)을 사용합니다.

```xml
<select id="selectUsers" resultType="User">
  select
    user_id as "id",
    user_name as "userName",
    hashed_password as "hashedPassword"
  from some_table
  where id = #{id}
</select>
```

별칭을 사용하지 않고도 문제를 해결할 수 있는데, 다음과 같이 `resultMap`을 선언해서 사용하면 됩니다.

```xml
<resultMap id="userResultMap" type="User">
  <id property="id" column="user_id" />
  <result property="username" column="username"/>
  <result property="password" column="password"/>
</resultMap>

<select id="selectUsers" resultMap="userResultMap">
  select user_id, user_name, hashed_password
  from some_table
  where id = #{id}
</select>
```

**복잡한 결과매핑** 

MyBatis도 매우 복잡한 결과에 객체 연관관계를 고려해서 데이터를 조회하는 것이 가능합니다. 

이 때는 `<association>`, `<collection>` 등을 사용합니다. 

이 부분은 성능과 실효성에서 측면에서 많은 고민이 필요합니다. 

JPA는 객체와 관계형 데이터베이스를 ORM 개념으로 매핑하기 때문에 이런 부분이 자연스럽지만, MyBatis 에서는 들어가는 공수도 많고, 성능을 최적화하기도 어렵습니다.. 따라서 해당기능을 사용할 때는 신중하게 사용해야 합니다.

> 참고 - 결과 매핑에 대한 자세한 내용은 다음을 참고하자. https://mybatis.org/mybatis-3/ko/sqlmap-xml.html#Result_Maps
>

# ==== 4. 데이터 접근 기술 - JPA ====
# 1. JPA 시작

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a3e26c4a-f4c7-4ff9-8821-8f7a925870dc/Untitled.png)

스프링과 JPA는 자바 엔터프라이즈(기업) 시장의 주력 기술입니다. 

스프링이 DI 컨테이너를 포함한 애플리케이션 전반의 다양한 기능을 제공한다면, JPA는 ORM 데이터 접근 기술을 제공합니다.

스프링 + 데이터 접근기술의 조합을 구글 트랜드로 비교했을 때 아래와 같습니다.

- 글로벌에서는 스프링+JPA 조합을 80%이상 사용
- 국내에서도 스프링 + JPA 조합을 50%정도 사용하고, 2015년 부터 점점 그 추세가 증가

JPA는 스프링 만큼이나 방대하고, 학습해야 할 분량도 많습니다. 하지만 한번 배워두면 데이터 접근 기술에서 매우 큰 생산성 향상을 얻을 수 있습니다. 대표적으로 JdbcTemplate 이나 MyBatis 같은 SQL 매퍼 기술은 SQL을 개발자가 직접 작성해야 하지만, JPA를 사용하면 SQL도 JPA가 대신 작성하고 처리해줍니다.

실무에서는 JPA를 더욱 편리하게 사용하기 위해 스프링 데이터 JPA 와 Querydsl 이라는 기술을 함께 사용합니다. 

근본적으로 중요한 것은 JPA입니다. 스프링 데이터 JPA, Querydsl은 JPA를 편리하게 사용하도록 도와주는 도구인 것이죠.

# 2. ORM 개념 1 - SQL 중심적인 개발의 문제점

관계형 DB (Oracle, MySQL)에서는 SQL만 사용할 수 있으므로 SQL 의존적인 개발을 할 수 밖에 없습니다.

관계형 DB의 목적과 객체지향 프로그래밍의 목적이 일치하지 않습니다. 

그러나, **객체를 저장할 수 있는 가장 현실적인 방안은 관계형 DB**입니다.

### **객체와 관계형 데이터베이스의 차이**

1. 상속

![https://user-images.githubusercontent.com/52024566/132992161-2db24f9c-3106-4d50-bdf1-9b83088d63a3.png](https://user-images.githubusercontent.com/52024566/132992161-2db24f9c-3106-4d50-bdf1-9b83088d63a3.png)

### **DB에서**

`Album` 객체를 저장할 경우 - 객체를 분해하여 `artist`는 `album`에 저장, `name`, `price`, `dtype`은 `item`에 저장합니다.

`Album` 객체를 조회할 경우 - 각 테이블을 조회하는 Join SQL 을 작성하고, 각각의 객체를 생성하고 …. 이런식으로 복잡합니다. **그래서 DB에 저장할 객체에는 상속 관계를 쓰지 않습니다.**

### **자바에서**

`Album` 객체를 저장할 경우

```java
list.add(album);
```

Album 객체를 조회할 경우

```java
Album album = list.get(albumId);

⬇️

Item item = list.get(albumId);
```

부모 타입으로 조회 후에 다형성을 활용할 수도 있습니다.

### 연관관계

객체 연관관계와 테이블의 연관관계를 비교해봅시다.

![https://user-images.githubusercontent.com/52024566/132992295-e91aa5be-9080-47de-b9e5-efa4ab6a40a2.png](https://user-images.githubusercontent.com/52024566/132992295-e91aa5be-9080-47de-b9e5-efa4ab6a40a2.png)

객체는 참조를 사용 - `member.getTeam()`

테이블은 외래 키를 사용 - `JOIN ON M.TEAM_ID = T.TEAM_ID`

**객체를 테이블에 맞추어 모델링했을 때 형태는 아래와 같습니다.**

```java
class Member {
	String id;		// MEMBER_ID
	Long teamId;	// TEAM_ID FK
  String username;// USERNAME
}

class Team {
	Long id;		// TEAM_ID PK
	String name;	// NAME
}
```

하지만 **객체답게 모델링했을 때는 아래와 같습니다.**

```java
class Member {
	String id;		// MEMBER_ID
	Team team;		// 참조로 연관관계를 맺는다
  String username;// USERNAME
    
  Team getTeam() {
      return team;
  }
}

class Team {
	Long id;		// TEAM_ID PK
	String name;	// NAME
}
```

**객체 모델링 조회**

```java
SELECT M.*, T.*
  FROM MEMBER M
  JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID
```

```java
public Member find(String memberId) {
    // SQL 실행
    Member member = new Member();
    // 데이터베이스에서 조회한 회원 관련 정보 입력
    Team team = new Team();
    // 데이터베이스에서 조회한 팀 관련 정보 입력
    
    // 회원과 팀 관계 설정
    member.setTeam(team);
    return member;
}
```

이런 식으로 굉장히 번거로운 작업을 거쳐야 합니다.

**객체 그래프 탐색**

객체는 자유롭게 객체 그래프를 탐색할 수 있어야 합니다. 하지만 실행하는 SQL 에 따라 탐색 범위가 결정되기 때문에 **SQL 중심적인 개발에서는 객체 그래프 탐색이 불가능**합니다.

**엔티티 신뢰 문제**

```java
class MemberService {
    public void process() {
        Member member = memberDAO.find(memberId);
        member.getTeam(); // 사용 가능한가?
        member.getOrder().getDelivery(); // 사용 가능한가?
    }
}
```

SQL에서 탐색된 객체 이외에는 사용할 수 없으므로 **엔티티를 신뢰할 수 없습니다.** 신뢰할 수 없다는 것은 get 문법을 통해서 불러왔을 때 `null` 이 아닌지 등의 문제를 말합니다. 

계층 아키텍쳐에서는 이전 계층에서 넘어온 내용을 신뢰할 수 있어야 하는데 SQL 중심적인 개발에서는 그것이 불가능하므로 객체가 SQL에 의존하게 됩니다.

그렇다고 모든 객체를 미리 로딩할 수 없으므로 상황에 따라 동일한 회원 조회 메서드를 여러 벌 생성해야 합니다.

### 생성한 객체 비교

**SQL 중심적인 개발**

```java
String memberId = "100";
Member member1 = memberDAO.getMember(memberId);
Member member2 = memberDAO.getMember(memberId);

member1 == member2; // 다르다.
```

**자바 컬렉션**

```java
String memberId = "100";
Member member1 = list.get(memberId);
Member member2 = list.get(memberId);

member1 == member2; // 같다.
```

SQL 중심적인 개발에서는 `getMember()`을 호출할 때 `New Member()`로 객체를 생성하기 때문에 `member1`과 `member2`가 다릅니다. 

그러나 자바 컬렉션에서 조회할 경우 `member1`과 `member2`의 참조 값이 같기 때문에 두 객체는 같습니다.

**이렇게 객체를 객체답게 모델링할수록 불필요해보이는 매핑 작업만 늘어납니다.**

# 3. ORM 개념 2 - JPA 소개

이제 JPA 을 개념적으로 설명하겠습니다.

### J**PA 이란?**

JPA 는 Java Persistence API 의 약자로 자바 진영의 **ORM** 기술 표준입니다.

### **ORM 이란?**

ORM 은 Object-relational mapping(객체 관계 매핑)의 약자입니다.

**객체는 객체대로 설계**하고 **관계형 데이터베이스는 관계형 데이터베이스대로 설계**하는 것이 핵심입니다.

ORM 프레임워크가 중간에서 매핑을 담당해줍니다. 대중적인 언어에는 대부분 ORM 기술이 존재합니다. 당연히 파이썬에도 ORM 기술이 따로 있습니다.

### **JPA 동작**

**저장**

![https://user-images.githubusercontent.com/52024566/132992893-ccfa7103-2a55-4f81-80c2-4e4bd269fefd.png](https://user-images.githubusercontent.com/52024566/132992893-ccfa7103-2a55-4f81-80c2-4e4bd269fefd.png)

**조회**

![https://user-images.githubusercontent.com/52024566/132992894-d55e1e4b-5833-44cc-bb30-4a1006a840ac.png](https://user-images.githubusercontent.com/52024566/132992894-d55e1e4b-5833-44cc-bb30-4a1006a840ac.png)

### **JPA는 표준 명세**

JPA는 인터페이스의 모음입니다.

JPA 2.1 표준 명세를 구현한 3가지 구현체가 아래 그림처럼 존재합니다.

![https://user-images.githubusercontent.com/52024566/132992940-b11dc52d-524e-4897-8b8a-ff101b5af5d4.png](https://user-images.githubusercontent.com/52024566/132992940-b11dc52d-524e-4897-8b8a-ff101b5af5d4.png)

보통 Hibernate 을 가장 대중적으로 사용합니다.

### **JPA를 왜 사용해야 하는가?**

SQL 중심적인 개발에서 객체 중심으로 개발이 가능해집니다.

JPA 을 사용하여 **생산성**이 향상됩니다. 간단하게 아래 코드로 저장, 조회, 수정, 삭제를 구현할 수 있습니다.

- 저장: jpa.persist(member)
- 조회: Member member = jpa.find(memberId)
- 수정: member.setName(“변경할 이름”)
- 삭제: jpa.remove(member)

또한 **유지보수**에도 이점이 있습니다.

- 기존에는 필드 변경시 모든 SQL 문을 수정해야 했지만, JPA에서는 필드만 추가하면 SQL은 JPA가 처리해줍니다.

**패러다임의 불일치를 해결**할 수 있습니다.

1. JPA와 상속
특정 객체를 저장할 경우 상속 관계를 JPA가 분석하여 필요한 쿼리를 JPA가 생성해줍니다.
2. JPA와 연관관계, JPA와 객체 그래프 탐색
지연 로딩을 사용하여 신뢰할 수 있는 엔티티, 계층을 제공해줍니다.
3. JPA와 비교하기
동일한 트랜잭션에서 조회한 엔티티는 같음을 보장합니다.

또한 **성능 최적화 기능**도 있습니다.

1차 캐시와 동일성(identity)을 보장합니다.

1. 같은 트랜잭션 안에서는 같은 엔티티를 반환하여 약간의 조회 성능이 향상됩니다.
2. DB Isolation Level 이 Read Commit 이어도 애플리케이션에서 Repeatable Read 을 보장합니다.
3. 트랜잭션을 지원하는 쓰기 지연(transactional write-behind)
    1. 트랜잭션을 커밋할 때까지 INSERT SQL을 모읍니다.
    2. JDBC BATCH SQL 기능을 사용해서 모았던 INSERT SQL 을 한 번에 전송합니다.
    3. UPDATE, DELETE로 인한 로우(ROW)락 시간을 최소화합니다.
    4. 트랜잭션 커밋 시 UPDATE, DELETE SQL 실행하고, 바로 커밋해줍니다.
    5. 지연 로딩(Lazy Loading) 기능을 수행해줍니다.
        
        (지연 로딩: 객체가 실제 사용될 때 로딩 )
        
        (즉시 로딩: JOIN SQL 로 한번에 연관된 객체까지 미리 조회)
        
    
    ![https://user-images.githubusercontent.com/52024566/132993188-add758c8-5c57-4be8-ae05-ee5a6b6ea74b.png](https://user-images.githubusercontent.com/52024566/132993188-add758c8-5c57-4be8-ae05-ee5a6b6ea74b.png)
    

또한 데이터 접근 추상화을 제공하며 벤더 독립성을 제공합니다.

그리고 결국 JPA 는 자바의 표준입니다!

이렇게 JPA, ORM 을 소개하고 사용해야 하는 이유를 설명했습니다. 다음 글에서는 본격적으로 JPA 을 프로젝트에 직접 적용해 볼 것입니다.

# 4. JPA 설정
이전 글에 이어서 JPA 을 프로젝트에 실제로 적용해봅시다. 먼저 설정부터 해야합니다.

`spring-boot-starter-data-jpa` 라이브러리를 사용하면 JPA와 스프링 데이터 JPA를 스프링 부트와 통합하고, 설정도 아주 간단히 할 수 있습니다. (스프링 데이터 JPA 는 다음 글에서 구체적으로 설명할 것입니다.)

`build.gradle`에 다음 의존 관계를 추가합니다.

```java
//JPA, 스프링 데이터 JPA 추가
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

`build.gradle`에서 다음 의존 관계를 제거합니다.

```java
//JdbcTemplate 추가
//implementation 'org.springframework.boot:spring-boot-starter-jdbc'
```

`spring-boot-starter-data-jpa`는 `spring-boot-starter-jdbc`도 함께 포함(의존)하므로 해당 라이브러리 의존관계를 제거해도 됩니다. 

`build.gradle` - 의존관계 전체

```java
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
  implementation 'org.springframework.boot:spring-boot-starter-web'
  
  //JdbcTemplate 추가
  //implementation 'org.springframework.boot:spring-boot-starter-jdbc'
  //MyBatis 추가
  implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
  //JPA, 스프링 데이터 JPA 추가
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
  
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

다음과 같은 라이브러리가 추가되었습니다.

- `hibernate-core`: JPA 구현체인 하이버네이트 라이브러리
- `jakarta.persistence-api`: JPA 인터페이스
- `spring-data-jpa`: 스프링 데이터 JPA 라이브러리

`**application.properties`에 다음 설정을 추가합니다.**

`main, test - application.properties`

```java
#JPA log
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

- `org.hibernate.SQL=DEBUG`: 하이버네이트가 생성하고 실행하는 SQL을 로그 메시지로 확인할 수 있습니다.
- `org.hibernate.type.descriptor.sql.BasicBinder=TRACE`: SQL에 바인딩 되는 파라미터를 확인할 수 있습니다.

# 5. JPA 적용 1 - 개발

JPA에서 가장 중요한 부분은 객체와 테이블을 매핑하는 것입니다. 

JPA가 제공하는 애노테이션을 사용해서 `Item` 객체와 테이블을 매핑해줍니다.

`Item` - ORM 매핑

```java
import lombok.Data;

import javax.persistence.*;

@Data
@Entity
public class Item {
  
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
  
    @Column(name = "item_name", length = 10)
    private String itemName;
    private Integer price;
    private Integer quantity;
  
    public Item() {
    }
  
    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

`@Entity`: JPA가 사용하는 객체라는 의미로 이 에노테이션이 있어야 JPA가 인식할 수 있습니다. 이렇게 `@Entity`가 붙은 객체를 JPA에서는 엔티티라고 합니다.

`@Id`: 테이블의 PK와 해당 필드를 매핑합니다.

`@GeneratedValue(strategy = GenerationType.IDENTITY)`: PK 생성 값을 데이터베이스에서 생성하는 `IDENTITY` 방식을 사용합니다. 
예) MySQL auto increment

`@Column`: 객체의 필드를 테이블의 컬럼과 매핑합니다.

- `name = "item_name"`: 객체는 `itemName`이지만 테이블의 컬럼은 `item_name`이므로 이렇게 매핑됩니다.
- `length = 10`: JPA의 매핑 정보로 DDL(`create table`)도 생성할 수 있는데, 그때 컬럼의 길이 값으로 활용합니다. (`varchar 10`)

`@Column`을 생략할 경우 필드의 이름을 테이블 컬럼 이름으로 사용합니다. 참고로 지금처럼 스프링 부트와 통합해서 사용하면 필드 이름을 테이블 컬럼 명으로 변경할 때 객체 필드의 카멜 케이스를 테이블 컬럼의 언더스코어로 자동으로 변환합니다.

- `itemName` → `item_name`, 따라서 위 예제의 `@Column(name = "item_name")`를 생략해도 됩니다.

JPA는 `public` 또는 `protected`의 기본 생성자가 필수입니다.

`public Item() {}`

`JpaItemRepositoryV1`

```java
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import javax.persistence.EntityManager;
import javax.persistence.TypedQuery;

@Slf4j
@Repository
@Transactional
public class JpaItemRepositoryV1 implements ItemRepository {
  
    private final EntityManager em;
  
    public JpaItemRepositoryV1(EntityManager em) {
        this.em = em;
    }
  
    @Override
    public Item save(Item item) {
        em.persist(item);
        return item;
    }
  
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = em.find(Item.class, itemId);
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        Item item = em.find(Item.class, id);
        return Optional.ofNullable(item);
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String jpql = "select i from Item i";
      
        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();
      
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            jpql += " where";
        }
      
        boolean andFlag = false;
        List<Object> param = new ArrayList<>();
        if (StringUtils.hasText(itemName)) {
            jpql += " i.itemName like concat('%',:itemName,'%')";
            param.add(itemName);
            andFlag = true;
        }
      
        if (maxPrice != null) {
            if (andFlag) {
                jpql += " and";
            }
            jpql += " i.price <= :maxPrice";
            param.add(maxPrice);
        }
      
        log.info("jpql={}", jpql);
      
        TypedQuery<Item> query = em.createQuery(jpql, Item.class);
        if (StringUtils.hasText(itemName)) {
            query.setParameter("itemName", itemName);
        }
      
        if (maxPrice != null) {
            query.setParameter("maxPrice", maxPrice);
        }
        return query.getResultList();
    }
}
```

`private final EntityManager em`: 생성자를 보면 스프링을 통해 엔티티 매니저(`EntityManager`) 라는 것을 주입받은 것을 확인할 수 있습니다. 

JPA의 모든 동작은 엔티티 매니저를 통해서 이루어집니다. 엔티티 매니저는 내부에 **데이터소스**를 가지고 있고, **데이터베이스에 접근**할 수 있습니다.

`@Transactional`: JPA의 모든 데이터 변경(등록, 수정, 삭제)은 트랜잭션 안에서 이루어져야 합니다. 

조회는 트랜잭션이 없어도 가능합니다. 

변경의 경우 일반적으로 서비스 계층에서 트랜잭션을 시작하기 때문에 문제가 없습니다. 

하지만 이번 예제에서는 복잡한 비즈니스 로직이 없어서 서비스 계층에서 트랜잭션을 걸지 않았습니다. JPA에서는 데이터 변경 시 트랜잭션이 필수이므로 리포지토리에 트랜잭션을 걸었습니다. 

다시 한 번 강조하지만 **일반적으로는 비즈니스 로직을 시작하는 서비스 계층에 트랜잭션을 걸어주는 것이 맞습니다.**

> 참고 - JPA를 설정하려면 `EntityManagerFactory`, JPA 트랜잭션 매니저(`JpaTransactionManager`), 데이터소스 등등 다양한 설정을 해야 하지만 스프링 부트는 이 과정을 모두 자동화 해줍니다. 
스프링 부트의 자동 설정은 `JpaBaseConfiguration`를 참고합시다.
> 

`JpaConfig`

```java
@Configuration
public class JpaConfig {
  
    private final EntityManager em;
  
    public JpaConfig(EntityManager em) {
        this.em = em;
    }
  
    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
  
    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepositoryV1(em);
    }
}
```

`ItemServiceApplication` - 변경

```java
//@Import(MyBatisConfig.class)
@Import(JpaConfig.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {}
```

`JpaConfig`를 사용하도록 변경였습니다.

**테스트 실행** 

먼저 `ItemRepositoryTest`를 통해서 리포지토리가 정상 동작하는지 확인합시다. 테스트가 모두 성공해야 합니다.

**애플리케이션 실행** 

`ItemServiceApplication`를 실행해서 애플리케이션이 정상 동작하는지 확인합니다.

정상적으로 동작하네요!

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f75e3b46-a48e-40da-902d-cd700bc20172/Untitled.png)

# 6. JPA 적용 2 - 리포지토리 분석

`save()` - 저장

```java
public Item save(Item item) {
    em.persist(item);
    return item;
}
```

`em.persist(item)`: JPA에서 객체를 테이블에 저장할 때는 엔티티 매니저가 제공하는 `persist()` 메서드를 사용합니다.

**JPA가 만들어서 실행한 SQL**

```sql
insert into item (id, item_name, price, quantity) values (null, ?, ?, ?)
		또는
insert into item (id, item_name, price, quantity) values (default, ?, ?, ?)
		또는
insert into item (item_name, price, quantity) values (?, ?, ?)
```

JPA가 만들어서 실행한 SQL을 보면 id 에 값이 빠져있는 것을 확인할 수 있습니다. 

PK 키 생성 전략을 `IDENTITY`로 사용했기 때문에 JPA가 이런 쿼리를 만들어서 실행한 것이지요. 

물론 쿼리 실행 이후에 `Item` 객체의 `id` 필드에 데이터베이스가 생성한 PK값이 들어갑니다. (JPA가 INSERT SQL 실행 이후에 생성된 ID 결과를 받아서 넣어줍니다.)

**PK 매핑 참고**

```java
@Entity
public class Item {
  
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

`update`() - 수정

```java
public void update(Long itemId, ItemUpdateDto updateParam) {
    Item findItem = em.find(Item.class, itemId);
    findItem.setItemName(updateParam.getItemName());
    findItem.setPrice(updateParam.getPrice());
    findItem.setQuantity(updateParam.getQuantity());
}
```

**JPA가 만들어서 실행한 SQL**

```sql
update item set item_name=?, price=?, quantity=? where id=?
```

`em.update()` 같은 메서드를 전혀 호출하지 않았는데 UPDATE SQL이 실행됩니다.

JPA는 트랜잭션이 커밋되는 시점에, 변경된 엔티티 객체가 있는지 확인을 하고 특정 엔티티 객체가 변경된 경우에는 UPDATE SQL을 실행해줍니다.

JPA가 어떻게 변경된 엔티티 객체를 찾는지 명확하게 이해하려면 영속성 컨텍스트라는 JPA 내부 원리를 이해해야 합니다. 지금은 트랜잭션 커밋 시점에 JPA가 변경된 엔티티 객체를 찾아서 UPDATE SQL을 수행한다고 이해만 해둡시다.

테스트의 경우 마지막에 트랜잭션이 롤백되기 때문에 JPA는 UPDATE SQL을 실행하지 않습니다. 테스트에서 UPDATE SQL을 확인하려면 `@Commit`을 붙이면 확인할 수 있습니다.

`findById()` - 단일 조회

```java
public Optional<Item> findById(Long id) {
    Item item = em.find(Item.class, id);
    return Optional.ofNullable(item);
}
```

JPA 에서 엔티티 객체를 PK를 기준으로 조회할 때는 `find()`를 사용하고 조회 타입과 PK 값을 주면 됩니다. 그러면 JPA가 다음과 같은 조회 SQL을 만들어서 실행하고 결과를 객체로 바로 변환해줍니다.

**JPA가 만들어서 실행한 SQL**

```sql
select
  item0_.id as id1_0_0_,
  item0_.item_name as item_nam2_0_0_,
  item0_.price as price3_0_0_,
  item0_.quantity as quantity4_0_0_
from item item0_
where item0_.id=?
```

JPA(하이버네이트)가 만들어서 실행한 SQL은 별칭이 조금 복잡합니다. 조인이 발생하거나 복잡한 조건에서도 문제 없도록 기계적으로 만들다 보니 이런 결과가 나온 것일겁니다.

`findAll` - 목록 조회

```java
public List<Item> findAll(ItemSearchCond cond) {
    String jpql = "select i from Item i";
    //동적 쿼리 생략
    TypedQuery<Item> query = em.createQuery(jpql, Item.class);
    return query.getResultList();
}
```

### **JPQL**

JPA는 JPQL(Java Persistence Query Language)이라는 객체지향 쿼리 언어를 제공합니다. 주로 여러 데이터를 복잡한 조건으로 조회할 때 사용합니다. 

SQL이 테이블을 대상으로 한다면, JPQL은 엔티티 객체를 대상으로 SQL을 실행한다 생각하면 됩니다. 

엔티티 객체를 대상으로 하기 때문에 `from` 다음에 `Item` 엔티티 객체 이름이 들어갑니다. 엔티티 객체와 속성의 대소문자는 구분해야 합니다. 

JPQL은 SQL과 문법이 거의 비슷하기 때문에 개발자들이 쉽게 적응할 수 있습니다.

결과적으로 JPQL을 실행하면 그 안에 포함된 엔티티 객체의 매핑 정보를 활용해서 SQL을 생성합니다.

**실행된 JPQL**

```sql
select i from Item i
where i.itemName like concat('%',:itemName,'%')
  and i.price <= :maxPrice
```

**JPQL을 통해 실행된 SQL**

```sql
select
  item0_.id as id1_0_,
  item0_.item_name as item_nam2_0_,
  item0_.price as price3_0_,
  item0_.quantity as quantity4_0_
from item item0_
where (item0_.item_name like ('%'||?||'%'))
  and item0_.price<=?
```

**파라미터**

JPQL에서 파라미터는 다음과 같이 입력한다면 파라미터 바인딩은 그 아래처럼 사용됩니다.

```sql
where price <= :maxPrice
⬇️
query.setParameter("maxPrice", maxPrice)
```

- `where price <= :maxPrice`
- `query.setParameter("maxPrice", maxPrice)`

**동적 쿼리 문제** 

실무에서는 동적 쿼리 문제 때문에, JPA 사용할 때 Querydsl 도 함께 선택합니다.


# 7. JPA 적용 3 - 예외 변환

JPA의 경우 예외가 발생하면 **JPA 예외**가 발생합니다.

```java
@Repository
@Transactional
public class JpaItemRepositoryV1 implements ItemRepository {
  
    private final EntityManager em;
  
    @Override
    public Item save(Item item) {
        em.persist(item);
        return item;
    }
}
```

`EntityManager`는 순수한 JPA 기술이고, 스프링과는 관계가 없습니다. 따라서 엔티티 매니저는 예외가 발생하면 JPA 관련 예외를 발생시킵니다.

JPA는 `PersistenceException`과 그 하위 예외를 발생시킵니다.

- 추가로 JPA는 `IllegalStateException`, `IllegalArgumentException`을 발생시킬 수 있습니다.

### **예외 변환 전**

![https://user-images.githubusercontent.com/52024566/188884085-d4b7e7d1-ca10-421d-8fd2-90053bb3b075.png](https://user-images.githubusercontent.com/52024566/188884085-d4b7e7d1-ca10-421d-8fd2-90053bb3b075.png)

**@Repository의 기능**

`@Repository`가 붙은 클래스는 컴포넌트 스캔의 대상입니다. 그리고 예외 변환 AOP 의 적용 대상이기도 합니다.

스프링과 JPA를 함께 사용하는 경우 스프링은 JPA 예외 변환기 (`PersistenceExceptionTranslator`)를 등록합니다.

예외 변환 AOP 프록시는 JPA 관련 예외가 발생하면 JPA 예외 변환기를 통해 발생한 예외를 **스프링 데이터 접근 예외로 변환**합니다.

### **예외 변환 후**

![https://user-images.githubusercontent.com/52024566/188884100-a19e92ca-c0f4-481f-9977-437643b83af5.png](https://user-images.githubusercontent.com/52024566/188884100-a19e92ca-c0f4-481f-9977-437643b83af5.png)

결과적으로 리포지토리에 `@Repository` 애노테이션만 있으면 스프링이 예외 변환을 처리하는 AOP를 생성합니다.

> 참고 - 스프링 부트는 `PersistenceExceptionTranslationPostProcessor`를 자동으로 등록하는데, 여기에서 `@Repository`를 AOP 프록시로 만드는 어드바이저가 등록됩니다.
> 
> 
> 복잡한 과정을 거쳐서 실제 예외를 변환하는데, 실제 JPA 예외를 변환하는 코드는 아래와 같습니다.
> 
> ```java
> EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible()
> ```
>

# 1. 스프링 데이터 JPA 소개1 - 등장 이유

이제는 스프링 데이터 JPA 에 대해서 알아봅니다. 

스프링 데이터 JPA 는 스프링 프레임워크에서 JPA을 한 단계 더 추상화시켜서 편리하게 사용할 수 있도록 지원하는 프로젝트(모듈)입니다. 

Spring Data JPA의 목적은 JPA를 사용할 때 필수적으로 생성해야하나, **예상가능하고 반복적인 코드들을 대신 작성**해줘서 코드를 줄여주는 것입니다. 

이는 JPA를 한 단계 추상화시킨 ****`JpaRepository`라는 인터페이스를 제공함으로써 이루어집니다.

Spring Data JPA는 JPA Provider 가 아닙니다. 단지 데이터 계층에 접근하기 위해 필요한 뻔한 코드들의 사용을 줄여주도록 하는 인터페이스입니다다. 

즉, 중요한 점은 **Spring Data JPA는 항상 하이버네이트와 같은 JPA provider가 필요하다**는 것입니다.

# 2. 스프링 데이터 JPA 소개2 - 기능

Spring Data JPA 는 CRUD 처리를 위한 공통 인터페이스 `JpaRepository`을 제공합니다. repository 개발 시 인터페이스만 작성하면 실행 시점에 스프링 데이터 JPA 가 알아서 구현 객체를 동적으로 생성해서 주입해줍니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0af778a1-de15-42f0-bd2d-b31f5ced17fe/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/1d816e79-1732-43be-8562-a6ed7abc72f9/Untitled.png)

```java
public interface ItemRepository extends JpaRepository<Member, Long> {}
```

하지만 단점도 있습니다. 일단 Spring Data JPA는 자세한 쿼리문을 작성하기 어려워집니다. 

이 경우에는 JPA 나 JDBCTemplate 로 쿼리문을 작성하면 됩니다. 

### **공통 인터페이스 기능**

![https://user-images.githubusercontent.com/52024566/200572119-fbf39588-8ca8-4ee5-a044-f061ab1eae82.png](https://user-images.githubusercontent.com/52024566/200572119-fbf39588-8ca8-4ee5-a044-f061ab1eae82.png)

`JpaRepository` 인터페이스를 통해서 기본적인 CRUD 기능을 제공합니다.

공통화 가능한 기능이 거의 모두 포함되어 있습니다.

`CrudRepository`에서의 `fineOne()` 이 `JpaRepository` 에서는(스프링 데이터 JPA 계층) `findById()`로 변경되는 것을 확인할 수 있습니다.

**JpaRepository 사용법**

```java
public interface ItemRepository extends JpaRepository<Member, Long> {
}
```

`JpaRepository` 인터페이스를 인터페이스 상속받고, 제네릭에 관리할 `<엔티티, 엔티티ID>`를 줍니다.

그러면 `JpaRepository`가 제공하는 기본 CRUD 기능을 모두 사용할 수 있습니다.

**스프링 데이터 JPA가 구현 클래스를 대신 생성**

![https://user-images.githubusercontent.com/52024566/200572124-92a4dbd1-a6e8-4ed4-85a1-1cc15ca1423c.png](https://user-images.githubusercontent.com/52024566/200572124-92a4dbd1-a6e8-4ed4-85a1-1cc15ca1423c.png)

`JpaRepository` 인터페이스만 상속받으면 스프링 데이터 JPA가 프록시 기술을 사용해서 구현 클래스를 생성합니다. 그리고 만든 구현 클래스의 인스턴스를 만들어서 스프링 빈으로 등록해줍니다.

따라서 개발자는 구현 클래스 없이 인터페이스만 만들면 기본 CRUD 기능을 사용할 수 있습니다.

### **쿼리 메서드 기능**

스프링 데이터 JPA는 인터페이스에 메서드만 적어두면, 메서드 이름을 분석해서 쿼리를 자동으로 만들고 실행해주는 기능을 제공합니다.

**순수 JPA 리포지토리**

```java
public List<Member> findByUsernameAndAgeGreaterThan(String username, int age) {
    return em.createQuery("select m from Member m where m.username = :username and m.age > :age")
      .setParameter("username", username)
      .setParameter("age", age)
      .getResultList();
}
```

순수 JPA를 사용하면 직접 JPQL을 작성하고, 파라미터도 직접 바인딩 해야 해야 합니다. 사실 JPA 을 사용해도 굉장히 편리하긴 합니다. 하지만 스프링 데이터 JPA 을 사용한다면 훨씬 더 간단해집니다.

**스프링 데이터 JPA**

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}
```

스프링 데이터 JPA는 메서드 이름을 분석해서 필요한 JPQL을 만들고 실행합니다. 물론 JPQL은 JPA가 다시 SQL로 번역해서 실행하겠죠.

이렇게 코드가 굉장히 간단해집니다.

당연히 그냥 아무 이름이나 사용하는 것은 아니고 다음과 같은 규칙을 따라야 합니다!

### **스프링 데이터 JPA가 제공하는 쿼리 메소드 기능**

조회: `find…By`, `read…By`, `query…By`, `get…By`

- 예:) `findHelloBy`처럼 …에 식별하기 위한 내용(설명)이 들어가도 됨
- COUNT: `count…By` 반환타입은 `long`
- EXISTS: `exists…By` 반환타입은 `boolean`

삭제: `delete…By`, `remove…By` 반환타입 `long`

- DISTINCT: `findDistinct`, `findMemberDistinctBy`
- LIMIT: `findFirst3`, `findFirst`, `findTop`, `findTop3`

> 쿼리 메소드 필터 조건은 스프링 데이터 JPA 공식 문서를 참고합시다. 

https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.querymethods.query-creation https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.limitquery-result
> 

**JPQL 직접 사용하기**

```java
public interface SpringDataJpaItemRepository extends JpaRepository<Item, Long>
{
    //쿼리 메서드 기능
    List<Item> findByItemNameLike(String itemName);
    
    //쿼리 직접 실행
    @Query("select i from Item i where i.itemName like :itemName and i.price <= :price")
    List<Item> findItems(@Param("itemName") String itemName, @Param("price") Integer price);
}
```

쿼리 메서드 기능 대신에 직접 JPQL을 사용하고 싶을 때는 `@Query`와 함께 JPQL 을 작성합니다. 이때는 메서드 이름으로 실행하는 규칙은 무시합니다.

참고로 스프링 데이터 JPA는 JPQL 뿐만 아니라 JPA의 네이티브 쿼리 기능도 지원하는데, JPQL 대신에 SQL을 직접 작성할 수 있습니다.

> 중요 - 스프링 데이터 JPA는 JPA를 편리하게 사용하도록 도와주는 도구입니다. 따라서 JPA 자체를 잘 이해하는 것이 가장 중요합니다.
> 

다음 글에서는 직접 스프링 데이터 JPA 을 프로젝트에 적용해보도록 하겠습니다.

# 4. 스프링 데이터 JPA 적용

먼저 스프링 데이터 JPA 을 적용하기 전에 설정부터 해줍니다.

### **설정**

`build.gradle` 추가

```java
//JPA, 스프링 데이터 JPA 추가
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

그런데 이미 이전 글에서 JPA를 설정하면서 `spring-boot-starter-data-jpa` 라이브러리를 넣어주었습니다. 여기에는 JPA , 하이버네이트, 스프링 데이터 JPA( spring-data-jpa ), 그리고 스프링 JDBC 관련 기능도 모두 포함되어 있습니다. 

따라서 스프링 데이터 JPA가 이미 추가되어있으므로 별도의 라이브러리 설정은 하지 않아도 됩니다.

### **스프링 데이터 JPA 적용**

`SpringDataJpaItemRepository`

```java
import hello.itemservice.domain.Item;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface SpringDataJpaItemRepository extends JpaRepository<Item, Long> {
    List<Item> findByItemNameLike(String itemName);
  
    List<Item> findByPriceLessThanEqual(Integer price);
  
    //쿼리 메서드 (아래 메서드와 같은 기능 수행)
    List<Item> findByItemNameLikeAndPriceLessThanEqual(String itemName, Integer price);
  
    //쿼리 직접 실행
    @Query("select i from Item i where i.itemName like :itemName and i.price <= :price")
    List<Item> findItems(@Param("itemName") String itemName, @Param("price") Integer price);
}
```

스프링 데이터 JPA가 제공하는 `JpaRepository` 인터페이스를 인터페이스 상속 받으면 기본적인 CRUD 기능을 사용할 수 있습니다.

그런데 이름으로 검색하거나, 가격으로 검색하는 기능은 공통으로 제공할 수 있는 기능이 아닙니다. 따라서 쿼리 메서드 기능을 사용하거나 `@Query`를 사용해서 직접 쿼리를 실행하면 됩니다.

여기서는 데이터를 조건에 따라 4가지로 분류해서 검색합니다.

- 모든 데이터 조회
- 이름 조회
- 가격 조회
- 이름 + 가격 조회

동적 쿼리를 사용하면 좋겠지만, 스프링 데이터 JPA는 동적 쿼리에 약하므로 이번에는 직접 4가지 상황을 스프링 데이터 JPA로 구현하였습니다.

> 참고 - 스프링 데이터 JPA도 Example이라는 기능으로 약간의 동적 쿼리를 지원하지만, 실무에서 사용하기는 기능이 빈약합니다. 실무에서 JPQL 동적 쿼리는 Querydsl을 사용하는 것이 좋습니다.
> 

`findAll()`

코드에서는 보이지 않지만 `JpaRepository` 공통 인터페이스가 제공하는 기능입니다. 모든 `Item`을 조회 다음과 같은 JPQL이 실행됩니다. 

```java
select i from Item i
```

`findByItemNameLike()` 

이름 조건만 검색했을 때 사용하는 쿼리 메서드입니다. 다음과 같은 JPQL이 실행됩니다. 

```java
select i from Item i where i.name like ?
```

`findByPriceLessThanEqual()` 

가격 조건만 검색했을 때 사용하는 쿼리 메서드입니다. 다음과 같은 JPQL이 실행됩니다. 

```java
select i from Item i where i.price <= ?
```

`findByItemNameLikeAndPriceLessThanEqual()` 

상품 이름과 가격 조건을 검색했을 때 사용하는 쿼리 메서드입니다. 다음과 같은 JPQL이 실행됩니다. 

```java
select i from Item i where i.itemName like ? and i.price <= ?
```

`findItems()` 

`@Query` 애노테이션을 사용해서 직접 JPQL 을 작성한 메서드입니다.

메서드 이름으로 쿼리를 실행하는 기능은 다음과 같은 단점이 있습니다.

1. 조건이 많으면 메서드 이름이 너무 길어짐.
2. 조인 같은 복잡한 조건을 사용할 수 없음. 
메서드 이름으로 쿼리를 실행하는 기능은 간단한 경우에는 매우 유용하지만, 복잡해지면  직접 JPQL 쿼리를 작성하는 것이 좋음.
    - 쿼리를 직접 실행하려면 `@Query` 애노테이션을 사용해야 함.
    - 메서드 이름으로 쿼리를 실행할 때는 파라미터를 순서대로 입력하면 되지만, 쿼리를 직접 실행할 때는 파라미터를 명시적으로 바인딩 해야 함.
    - 파라미터 바인딩은 `@Param("itemName")` 애노테이션을 사용하고, 애노테이션의 값에 파라미터 이름을 주면 됨.

`JpaItemRepositoryV2`

```java
import hello.itemservice.domain.Item;
import hello.itemservice.repository.ItemRepository;
import hello.itemservice.repository.ItemSearchCond;
import hello.itemservice.repository.ItemUpdateDto;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import java.util.List;
import java.util.Optional;

@Repository
@Transactional
@RequiredArgsConstructor
public class JpaItemRepositoryV2 implements ItemRepository {
  
    private final SpringDataJpaItemRepository repository;
  
    @Override
    public Item save(Item item) {
        return repository.save(item);
    }
  
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = repository.findById(itemId).orElseThrow();
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        return repository.findById(id);
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
      
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();
      
        if (StringUtils.hasText(itemName) && maxPrice != null) {
            //return repository.findByItemNameLikeAndPriceLessThanEqual("%" + itemName + "%", maxPrice);
            return repository.findItems("%" + itemName + "%", maxPrice);
        } else if (StringUtils.hasText(itemName)) {
            return repository.findByItemNameLike("%" + itemName + "%");
        } else if (maxPrice != null) {
            return repository.findByPriceLessThanEqual(maxPrice);
        } else {
            return repository.findAll();
        }
    }
}
```

**의존관계와 구조**

`ItemService`는 `ItemRepository`에 의존하기 때문에 `ItemService`에서 `SpringDataJpaItemRepository`를 그대로 사용할 수 없습니다.

물론 `ItemService`가 `SpringDataJpaItemRepository`를 직접 사용하도록 코드를 고치면 되겠지만, 

우리는 `ItemService` 코드의 변경없이 `ItemService`가 `ItemRepository`에 대한 의존을 유지하면서 DI를 통해 구현 기술을 변경하고 싶습니다. 그것이 객체 지향적인 설계이기도 하고요.

조금 복잡하지만, 새로운 리포지토리를 만들어서 이 문제를 해결합시다.

여기서는 `JpaItemRepositoryV2`가 `MemberRepository`와 `SpringDataJpaItemRepository` 사이를 맞추기 위한 어댑터 처럼 사용됩니다.

**클래스 의존 관계**

![https://user-images.githubusercontent.com/52024566/200841496-05801f83-098a-43aa-8f71-161dd2427c2c.png](https://user-images.githubusercontent.com/52024566/200841496-05801f83-098a-43aa-8f71-161dd2427c2c.png)

`JpaItemRepositoryV2`는 `ItemRepository`를 구현합니다. 

그리고 `SpringDataJpaItemRepository`를 사용합니다.

**런타임 객체 의존 관계**

![https://user-images.githubusercontent.com/52024566/200841505-a3a3865e-40c9-40b5-a03c-7c5c9103b42f.png](https://user-images.githubusercontent.com/52024566/200841505-a3a3865e-40c9-40b5-a03c-7c5c9103b42f.png)

런타임의 객체 의존관계는 다음과 같이 동작합니다.

`itemService` → `jpaItemRepositoryV2` → `springDataJpaItemRepository(프록시 객체)`

이렇게 중간에서 `JpaItemRepositoryV2`가 어댑터 역할을 해준 덕분에 `MemberService`가 사용하는 `MemberRepository` 인터페이스를 그대로 유지할 수 있고 클라이언트인 `MemberService`의 코드를 변경하지 않아도 되는 장점이 있습니다.

`save()`

```java
repository.save(item)
```

스프링 데이터 JPA가 제공하는 save()를 호출합니다.

`update()` 

스프링 데이터 JPA가 제공하는 `findById()` 메서드를 사용해서 엔티티를 찾습니다.

그리고 데이터를 수정한 이후 트랜잭션이 커밋될 때 변경 내용이 데이터베이스에 반영되도록 합니다. (JPA가 제공하는 기능)

`findById()` 

```java
repository.findById(itemId) 
```

스프링 데이터 JPA가 제공하는 `findById()` 메서드를 사용해서 엔티티를 찾음

`findAll()` 데이터를 조건에 따라 4가지로 분류해서 검색

- 모든 데이터 조회
- 이름 조회
- 가격 조회
- 이름 + 가격 조회

모든 조건에 부합할 때는 `findByItemNameLikeAndPriceLessThanEqual()`를 사용해도 되고, `repository.findItems()`를 사용해도 됩니다. 그런데 보는 것 처럼 조건이 2개만 되어도 이름이 너무 길어지는 단점이 있습니다. 따라서 스프링 데이터 JPA가 제공하는 메서드 이름으로 쿼리를 자동으로 만들어주는 기능과 `@Query`로 직접 쿼리를 작성하는 기능 중에 적절한 선택이 필요합니다.

추가로 코드를 잘 보면 동적 쿼리가 아니라 상황에 따라 각각 스프링 데이터 JPA의 메서드를 호출해서 상당히 비효율적인 코드인 것을 알 수 있습니다. 

앞서 이야기했듯이 **스프링 데이터 JPA는 동적 쿼리 기능에 대한 지원이 매우 약합니다.**

`SpringDataJpaConfig`

```java
@Configuration
@RequiredArgsConstructor
public class SpringDataJpaConfig {
  
    private final SpringDataJpaItemRepository springDataJpaItemRepository;
  
    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
  
    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepositoryV2(springDataJpaItemRepository);
    }
}
```

스프링 데이터 JPA가 `SpringDataJpaItemRepository`를 프록시 기술로 만들어주고 스프링 빈으로도 등록해줍니다.

`ItemServiceApplication` - 변경

```java
//@Import(JpaConfig.class)
@Import(SpringDataJpaConfig.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {}
```

`SpringDataJpaConfig`를 사용하도록 변경합니다.

**예외 변환** 

스프링 데이터 JPA도 스프링 예외 추상화를 지원합니다. 스프링 데이터 JPA가 만들어주는 **프록시에서 이미 예외 변환을 처리**하기 때문에, `@Repository`와 관계없이 **예외가 변환됩니다.**

> 주의! - 하이버네이트 버그 
****하이버네이트 `5.6.6` ~ `5.6.7`을 사용하면 `Like` 문장을 사용할 때 다음 예외가 발생합니다. 
`java.lang.IllegalArgumentException: Parameter value [\] did not match expected type [java.lang.String (n/a)]`
만약 문제가 있는 하이버네이트 버전을 사용한다면 `build.gradle`에 다음을 추가해서 하이버네이트 버전을 문제가 없는 `5.6.5.Final`로 변경해서 사용합시다.
> 
> 
> ```java
> ext["hibernate.version"] = "5.6.5.Final"
> ```
>

# ==== 6. 데이터 접근 기술 - Querydsl =====
# 1. Querydsl 소개 1 - 기존 방식의 문제점

### **기존 방식의 문제점**

Querydsl 을 사용하기 전 JPA 을 사용했을 때의 문제점부터 정리해 봅시다.

**QUERY의 문제점**

QUERY는 문자이므로 **Type-check 가 불가능**합니다.

또 **실행하기 전까지 작동여부를 확인할 수 없습니다.**

JPA에서 QUERY 방법은 크게 3가지가 있었습니다.

 **1. JPQL(HQL)**

![https://user-images.githubusercontent.com/52024566/201093333-219026a4-5541-44f3-aad0-cb06d240992a.png](https://user-images.githubusercontent.com/52024566/201093333-219026a4-5541-44f3-aad0-cb06d240992a.png)

장점

- SQL QUERY와 비슷해서 금방 익숙해질 수 있습니다.

단점

- type-safe 가 아닙니다.
- 동적쿼리 생성이 어렵습니다.

 **2. Criteria API**

Criteria API 는 자바 ORM 표준 프로그래밍에서 **JPQL 을 자바 코드로 작성할 수 있도록 도와주는 빌더 클래스 API** 을 의미합니다. 

![https://user-images.githubusercontent.com/52024566/201093339-f1f17480-e8de-4075-b995-9c0099ddaabf.png](https://user-images.githubusercontent.com/52024566/201093339-f1f17480-e8de-4075-b995-9c0099ddaabf.png)

`getCriteriaBuilder` 함수를 이용해서 Criteria Query Builder 을 생성하고 `createQuery` 함수로 CriteriaQuery 객체를 생성합니다.

Member 엔티티 별칭 root 을 사용해서 20, 40세 사이이며 김으로 시작하는 Member 을 가져옵니다. 이후 정렬 및 최대 결과에 대한 제한을 걸며 결과를 만듭니다.

장점

- 동적쿼리 생성 가능

단점

- type-safe 아님
- 너무 너무 너무 복잡함
- 알아야 할 게 너무 많음

 **3. MetaModel Criteria API(type-safe)**

- `root.get("age")` → `root.get(Member_.age)`
- Criteria API + MetaModel
- Criteria API와 거의 동일
- type-safe
- 복잡하긴 마찬가지

우리는 이러한 문제들을 Querydsl 로 해결할 수 있습니다.

# 2. Querydsl 소개 2 - 해결

### **DSL**

DSL 은 **도메인(Domain) + 특화(Specific) + 언어(Language) 입니다.**

특정한 도메인에 초점을 맞춘 제한적인 표현력을 가진 컴퓨터 프로그래밍 언어입니다.

**단순**하고 **간결**하며 **유창**하다는 특징을 가집니다.

### **QueryDSL**

그렇다면 **QueryDSL 은 쿼리 + 도메인 + 특화 + 언어** 입니다.

쿼리에 특화된 프로그래밍 언어이지요. 역시 **단순, 간결, 유창합니다.**

다양한 저장소 쿼리 기능을 통합하고 있습니다. **JPA**, **MongoDB**, **SQL** 같은 기술들을 위해 **type-safe SQL**을 만드는 프레임워크입니다.

### **데이터 쿼리 기능 추상화**

![https://user-images.githubusercontent.com/52024566/201093340-676a0c16-9aa0-470c-9d4d-5c32922861e7.png](https://user-images.githubusercontent.com/52024566/201093340-676a0c16-9aa0-470c-9d4d-5c32922861e7.png)

**Type-safe Query Type 생성**

![https://user-images.githubusercontent.com/52024566/201093342-581cf921-db93-43a7-9c41-3c9a22ea8a5d.png](https://user-images.githubusercontent.com/52024566/201093342-581cf921-db93-43a7-9c41-3c9a22ea8a5d.png)

**작동 방식**

![https://user-images.githubusercontent.com/52024566/201093343-b2d7bc8f-7f3c-4ca1-9a7a-eb09f98cbb4f.png](https://user-images.githubusercontent.com/52024566/201093343-b2d7bc8f-7f3c-4ca1-9a7a-eb09f98cbb4f.png)

**SpringDataJPA + Querydsl**

SpringDataJPA 프로젝트의 약점은 조회에 있습니다.

Querydsl 로 복잡한 조회 기능을 보완할 수 있습니다.

또한 복잡한 쿼리와 동적 쿼리에 약점을 가지는 Spring Data JPA 에 QueryDSL 을 합치는 것은 매우 좋은 선택이 됩니다. 

- 단순한 경우 : SpringDataJPA
- 복잡한 경우 : Querydsl 직접 사용

# 3. Querydsl 설정

`build.gradle`

Querydsl로 추가된 부분은 다음 두 부분입니다.

```java
dependencies {
  //Querydsl 추가
  implementation 'com.querydsl:querydsl-jpa'
  annotationProcessor "com.querydsl:querydsl-apt:$
  {dependencyManagement.importedProperties['querydsl.version']}:jpa"
  annotationProcessor "jakarta.annotation:jakarta.annotation-api"
  annotationProcessor "jakarta.persistence:jakarta.persistence-api"
}

//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
clean {
  delete file('src/main/generated')
}
```

- 전체코드
    
    ```java
    plugins {
      id 'org.springframework.boot' version '2.6.5'
      id 'io.spring.dependency-management' version '1.0.11.RELEASE'
      id 'java'
    }
    
    group = 'com.example'
    version = '0.0.1-SNAPSHOT'
    sourceCompatibility = '11'
    
    ext["hibernate.version"] = "5.6.5.Final"
    
    configurations {
      compileOnly {
        extendsFrom annotationProcessor
      }
    }
    
    repositories {
      mavenCentral()
    }
    
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
        implementation 'org.springframework.boot:spring-boot-starter-web'
        
        //JdbcTemplate 추가
        //implementation 'org.springframework.boot:spring-boot-starter-jdbc'
        //MyBatis 추가
        implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:2.2.0'
        //JPA, 스프링 데이터 JPA 추가
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
        
        //Querydsl 추가
        implementation 'com.querydsl:querydsl-jpa'
        annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
        annotationProcessor "jakarta.annotation:jakarta.annotation-api"
        annotationProcessor "jakarta.persistence:jakarta.persistence-api"
        
        //H2 데이터베이스 추가
        runtimeOnly 'com.h2database:h2'
        compileOnly 'org.projectlombok:lombok'
        annotationProcessor 'org.projectlombok:lombok'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        
        //테스트에서 lombok 사용
        testCompileOnly 'org.projectlombok:lombok'
        testAnnotationProcessor 'org.projectlombok:lombok'
    }
    
    tasks.named('test') {
      useJUnitPlatform()
    }
    
    //Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
    clean {
      delete file('src/main/generated')
    }
    ```
    

### **검증 - Q 타입 생성 확인 방법**

Preferences → Build, Execution, Deployment → Build Tools → Gradle

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/15211055-daa1-4567-834c-946a1f23f1a9/Untitled.png)

여기에 가면 크게 2가지 옵션을 선택할 수 있는데 옵션은 둘다 같게 맞추어야 합니다.

1. Gradle: Gradle을 통해서 빌드
2. IntelliJ IDEA: IntelliJ가 직접 자바를 실행해서 빌드

### 옵션 선택1 - Gradle - Q타입 생성 확인 방법

**Gradle IntelliJ 사용법**

- `Gradle -> Tasks -> build -> clean`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d18f8c47-2afd-415d-bdf3-733bb9a534cc/Untitled.png)

- `Gradle -> Tasks -> other -> compileJava`
    
    ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/159f7e8e-5187-40c8-a02a-b3328c78d014/Untitled.png)
    

**Gradle 콘솔 사용법**

- `./gradlew clean compileJava`

**Q 타입 생성 확인**

- `build -> generated -> sources -> annotationProcessor -> java/main` 하위에 `hello.itemservice.domain.QItem`이 생성되어 있어야 합니다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0fb53e3e-c96f-4a90-a62c-82254d76745d/Untitled.png)

> 참고 - Q타입은 컴파일 시점에 자동 생성되므로 버전관리(GIT)에 포함하지 않는 것이 좋습니다. 
gradle 옵션을 선택하면 Q타입은 gradle build 폴더 아래에 생성되기 때문에 여기를 포함하지 않아야 합니다.

대부분 gradle build 폴더를 git에 포함하지 않기 때문에 이 부분은 자연스럽게 해결됩니다.
> 

**Q타입 삭제** 

`gradle clean`을 수행하면 `build` 폴더 자체가 삭제되므로 별도의 설정이 필요하지 않습니다.

### 옵션 선택2 - IntelliJ IDEA - Q타입 생성 확인 방법

`Build -> Build Project` 또는 `Build -> Rebuild` 또는 `main()`, 또는 테스트를 실행

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3549b8ba-263d-4bf4-9cf5-55d334097e45/Untitled.png)

`src/main/generated` 하위에 `hello.itemservice.domain.QItem`이 생성되어 있어야 합니다.

> 참고 - Q타입은 컴파일 시점에 자동 생성되므로 버전관리(GIT)에 포함하지 않는 것이 좋습니다. IntelliJ IDEA 옵션을 선택하면 Q타입은 src/main/generated 폴더 아래에 생성되기 때문에 여기를 포함하지 않는 것이 좋습니다.
> 

**Q타입 삭제**

```java
//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거
clean {
  delete file('src/main/generated')
}
```

`IntelliJ IDEA` 옵션을 선택하면 `src/main/generated`에 파일이 생성되고, 필요한 경우 Q파일을 직접 삭제해야 합니다. `gradle`에 해당 스크립트를 추가하면 `gradle clean` 명령어를 실행할 때 `src/main/generated`의 파일도 함께 삭제됩니다.

### **참고**

Querydsl은 이렇게 설정하는 부분이 사용하면서 조금 귀찮은 부분인데, IntelliJ가 버전업 하거나 Querydsl의 Gradle 설정이 버전업 하면서 적용 방법이 조금씩 달라지기도 합니다. 

그리고 본인의 환경에 따라서 잘 동작하지 않기도 합니다. 공식 메뉴얼에 소개 되어 있는 부분이 아니기 때문에, 설정에 수고로움이 있지만 `querydsl gradle` 로 검색하면 본인 환경에 맞는 대안을 금방 찾을 수 있을 것입니다.

# 4. Querydsl 적용

`JpaItemRepositoryV3`

```java
import com.querydsl.core.BooleanBuilder;
import com.querydsl.core.types.dsl.BooleanExpression;
import com.querydsl.jpa.impl.JPAQueryFactory;
import hello.itemservice.domain.Item;
import hello.itemservice.domain.QItem;
import hello.itemservice.repository.ItemRepository;
import hello.itemservice.repository.ItemSearchCond;
import hello.itemservice.repository.ItemUpdateDto;
import org.springframework.stereotype.Repository;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.util.StringUtils;

import javax.persistence.EntityManager;
import java.util.List;
import java.util.Optional;

import static hello.itemservice.domain.QItem.*;

@Repository
@Transactional
public class JpaItemRepositoryV3 implements ItemRepository {
  
    private final EntityManager em;
    private final JPAQueryFactory query;
  
    public JpaItemRepositoryV3(EntityManager em) {
        this.em = em;
        this.query = new JPAQueryFactory(em);
    }
  
    @Override
    public Item save(Item item) {
        em.persist(item);
        return item;
    }
  
    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = findById(itemId).orElseThrow();
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }
  
    @Override
    public Optional<Item> findById(Long id) {
        Item item = em.find(Item.class, id);
        return Optional.ofNullable(item);
    }
  
    public List<Item> findAllOld(ItemSearchCond itemSearch) {
      
        String itemName = itemSearch.getItemName();
        Integer maxPrice = itemSearch.getMaxPrice();
      
        QItem item = QItem.item;
        BooleanBuilder builder = new BooleanBuilder();
        if (StringUtils.hasText(itemName)) {
            builder.and(item.itemName.like("%" + itemName + "%"));
        }
        if (maxPrice != null) {
            builder.and(item.price.loe(maxPrice));
        }
      
        List<Item> result = query
          .select(item)
          .from(item)
          .where(builder)
          .fetch();
        return result;
    }
  
    @Override
    public List<Item> findAll(ItemSearchCond cond) {
      
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();
      
        List<Item> result = query
          .select(item)
          .from(item)
          .where(likeItemName(itemName), maxPrice(maxPrice))
          .fetch();
      
        return result;
    }
  
    private BooleanExpression likeItemName(String itemName) {
        if (StringUtils.hasText(itemName)) {
            return item.itemName.like("%" + itemName + "%");
        }
        return null;
    }
  
    private BooleanExpression maxPrice(Integer maxPrice) {
        if (maxPrice != null) {
            return item.price.loe(maxPrice);
        }
        return null;
    }
}
```

**공통**

Querydsl 을 사용하려면 `JPAQueryFactory`가 필요합니다. `JPAQueryFactory`는 JPA 쿼리인 JPQL을 만들기 때문에 `EntityManager`가 필요합니다.

설정 방식은 `JdbcTemplate` 을 설정하는 것과 유사합니다.

참고로 `JPAQueryFactory` 를 스프링 빈으로 등록해서 사용해도 됩니다.

`save()`, `update()`, `findById()` 

기본 기능들은 JPA가 제공하는 기본 기능을 사용합니다.

`findAllOld` 

Querydsl 을 사용해서 동적 쿼리 문제를 해결하였습니다. 

`BooleanBuilder`를 사용해서 원하는 `where` 조건들을 넣습니다. 

이 모든 것을 자바 코드로 작성하기 때문에 동적 쿼리를 매우 편리하게 작성할 수 있습니다.

`findAll`

앞서 `findAllOld`에서 작성한 코드를 깔끔하게 리팩토링했습니다.

```java
List<Item> result = query
.select(item)
.from(item)
.where(likeItemName(itemName), maxPrice(maxPrice))
.fetch();
```

Querydsl에서 `where(A,B)`에 다양한 조건들을 직접 넣을 수 있는데, 이렇게 넣으면 AND 조건으로 처리됩니다. 참고로 `where()` 에 `null`을 입력하면 해당 조건은 무시합니다.

이 코드의 또 다른 장점은 `likeItemName()`, `maxPrice()`를 다른 쿼리를 작성할 때 재사용 할 수 있다는 점입니다. 

쉽게 이야기해서 **쿼리 조건을 부분적으로 모듈화 할 수 있습니다!!!**

**자바 코드로 개발하기 때문에 얻을 수 있는 큰 장점입니다.**

`QuerydslConfig`

```java
package hello.itemservice.config;
import hello.itemservice.repository.ItemRepository;
import hello.itemservice.repository.jpa.JpaItemRepositoryV3;
import hello.itemservice.service.ItemService;
import hello.itemservice.service.ItemServiceV1;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.persistence.EntityManager;

@Configuration
@RequiredArgsConstructor
public class QuerydslConfig {
  
    private final EntityManager em;
  
    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }
  
    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepositoryV3(em);
    }
}
```

`ItemServiceApplication` - 변경

```java
//@Import(SpringDataJpaConfig.class)
@Import(QuerydslConfig.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {}
```

`QuerydslConfig`를 사용하도록 변경합니다.

**예외 변환** 

`Querydsl`에서 별도의 스프링 예외 추상화를 지원하지 않는 대신 JPA에서 학습한 것처럼 `@Repository`에서 스프링 예외 추상화를 처리합니다.

# **정리**

### **Querydsl 장점**

Querydsl 덕분에 동적 쿼리를 매우 깔끔하게 사용할 수 있습니다.

```java
List<Item> result = query
    .select(item)
    .from(item)
    .where(likeItemName(itemName), maxPrice(maxPrice))
    .fetch();
```

쿼리 문장에 오타가 있어도 컴파일 시점에 오류를 막을 수 있습니다.

메서드 추출을 통해서 코드를 재사용할 수 있습니다. 예를 들어서 여기서 만든 `likeItemName(itemName)`, `maxPrice(maxPrice)` 메서드를 다른 쿼리에서도 함께 사용할 수 있습니다!

Querydsl을 사용해서 자바 코드로 쿼리를 작성하는 것이 장점입니다. 

그리고 스프링 데이터 JPA 에서의 약점인 동적 쿼리 문제도 깔끔하게 해결할 수 있습니다.  

Querydsl은 이 외에도 수 많은 편리한 기능을 제공합니다. 

예를 들어서 최적의 쿼리 결과를 만들기 위해서 DTO로 편리하게 조회하는 기능은 실무에서 자주 사용하는 기능입니다. 

JPA를 사용한다면 **스프링 데이터 JPA 와 Querydsl 은 실무의 다양한 문제를 편리하게 해결하기 위해 선택하는 기본 기술**입니다!

# ==== 7. 데이터 접근 기술 - 활용 방안 ====
# 1. 스프링 데이터 JPA 예제와 트레이드 오프

**클래스 의존 관계**

![https://user-images.githubusercontent.com/52024566/202184749-f747f1d7-d5d9-48d7-b7e9-860fa39ea10e.png](https://user-images.githubusercontent.com/52024566/202184749-f747f1d7-d5d9-48d7-b7e9-860fa39ea10e.png)

**런타임 객체 의존 관계**

![https://user-images.githubusercontent.com/52024566/202184757-5136f387-f16f-41b0-91c6-e401394df959.png](https://user-images.githubusercontent.com/52024566/202184757-5136f387-f16f-41b0-91c6-e401394df959.png)

중간에서 `JpaItemRepositoryV2`가 어댑터 역할을 해준 덕분에 `MemberService`가 사용하는 `MemberRepository` 인터페이스를 그대로 유지할 수 있고 클라이언트인 `MemberService` 의 코드를 변경하지 않아도 되는 장점이 있습니다.

**고민**

구조를 맞추기 위해서, 중간에 어댑터가 들어가면서 전체 구조가 너무 복잡해지고 사용하는 클래스도 많아지는 단점이 생깁니다.

실제 이 코드를 구현해야하는 개발자 입장에서 보면 중간에 어댑터도 만들고, 실제 코드까지 만들어야 하는 불편함이 발생합니다.

유지보수 관점에서 `ItemService`를 변경하지 않고 `ItemRepository`의 구현체를 변경할 수 있는 장점이 있습니다. 즉, DI, OCP 원칙을 지킬 수 있다는 좋은 점이 분명히 있지만 반대로 구조가 복잡해지면서 어댑터 코드와 실제 코드까지 함께 유지보수 해야 하는 어려움도 발생합니다.

### **다른 선택** (어댑터 제거)

여기서 완전히 다른 선택을 할 수도 있습니다. `ItemService` 코드를 일부 고쳐서 직접 스프링 데이터 JPA를 사용하는 방법입니다. 

이 방법에서는 DI, OCP 원칙을 포기하는 대신에, 복잡한 어댑터를 제거하고, 구조를 단순하게 가져갈 수 있는 장점이 있습니다.

**클래스 의존 관계**

![https://user-images.githubusercontent.com/52024566/202184759-1c0c9002-a793-4cc5-8835-0d67d62affc6.png](https://user-images.githubusercontent.com/52024566/202184759-1c0c9002-a793-4cc5-8835-0d67d62affc6.png)

**런타임 의존 관계**

![https://user-images.githubusercontent.com/52024566/202184763-1a3d6dc9-0cd0-4099-bfa8-95d00c1368d9.png](https://user-images.githubusercontent.com/52024566/202184763-1a3d6dc9-0cd0-4099-bfa8-95d00c1368d9.png)

`ItemService`에서 스프링 데이터 JPA로 만든 리포지토리를 직접 참조합니다. 이 경우 리포지토리가 변경된다면 `ItemService` 코드를 변경해야 합니다.

![https://user-images.githubusercontent.com/52024566/202184765-5e2b8ab2-f82b-4cbe-a8b0-947ecd9602c0.png](https://user-images.githubusercontent.com/52024566/202184765-5e2b8ab2-f82b-4cbe-a8b0-947ecd9602c0.png)

### **트레이드 오프**

결국 아래 두 방안 중에 한 가지를 선택하면 한 가지를 포기해야 하는 Trade-off 가 발생합니다.

- DI, OCP를 지키기 위해 어댑터를 도입하고, 더 많은 코드를 유지
- 어댑터를 제거하고 구조를 단순하게 가져가지만 DI, OCP를 포기하고, `ItemService` 코드를 직접 변경

결국 여기서 발생하는 트레이드 오프는 **구조의 안정성 vs 단순한 구조와 개발의 편리성** 사이의 선택입니다. 이 둘 중에 하나의 정답만 있는 것은 아닙니다. 어떤 상황에서는 구조의 안정성이 매우 중요하고, 어떤 상황에서는 단순한 것이 더 나은 선택일 수 있지요.

개발을 할 때는 항상 자원이 무한한 것이 아닙니다. 그리고 **어설픈 추상화는 오히려 독**이 되는 경우도 많습니다. 무엇보다 **추상화도 비용**이 듭니다. 

인터페이스도 비용이 듭니다. 여기서 말하는 비용은 유지보수 관점에서의 비용을 의미합니다. 이 추상화 비용을 넘어설 만큼 효과가 있을 때 추상화를 도입하는 것이 실용적입니다.

이런 선택에서 하나의 정답이 있는 것은 아니지만, 프로젝트의 현재 상황에 맞는 더 적절한 선택지가 있을 것입니다. 그리고 현재 상황에 맞는 선택을 하는 개발자가 좋은 개발자인 것이지요!



# 2. 실용적인 구조

### **복잡한 쿼리 분리**

![https://user-images.githubusercontent.com/52024566/202730259-2a567e82-0b02-4827-a368-4b6e21aed491.png](https://user-images.githubusercontent.com/52024566/202730259-2a567e82-0b02-4827-a368-4b6e21aed491.png)

Repository 을 아래처럼 두 가지로 나눌 수 있습니다.

- `ItemRepositoryV2`는 **스프링 데이터 JPA** 의 기능을 제공하는 리포지토리
- `ItemQueryRepositoryV2`는 **Querydsl** 을 사용해서 복잡한 쿼리 기능을 제공하는 리포지토리

이렇게 둘을 분리하면 기본 CRUD와 단순 조회는 스프링 데이터 JPA가 담당하고, 복잡한 조회 쿼리는 Querydsl이 담당합니다. 물론 `ItemService`는 기존 `ItemRepository`를 사용할 수 없기 때문에 코드를 변경해야 합니다.

**ItemRepositoryV2**

```java
import hello.itemservice.domain.Item;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ItemRepositoryV2 extends JpaRepository<Item, Long> {
}
```

`ItemRepositoryV2`는 `JpaRepository`를 인터페이스 상속 받아서 스프링 데이터 JPA의 기능을 제공하는 리 포지토리가 되었습니다.

기본 CRUD는 이 기능을 사용하면 될 것입니다.

여기에 추가로 네이밍 규칙을 따르는 단순한 조회 쿼리들을 추가할 수 있습니다.

`ItemQueryRepositoryV2`

```java
import com.querydsl.core.types.dsl.BooleanExpression;
import com.querydsl.jpa.impl.JPAQueryFactory;
import hello.itemservice.domain.Item;
import hello.itemservice.repository.ItemSearchCond;
import org.springframework.stereotype.Repository;
import org.springframework.util.StringUtils;

import javax.persistence.EntityManager;
import java.util.List;

import static hello.itemservice.domain.QItem.item;

@Repository
public class ItemQueryRepositoryV2 {
  
    private final JPAQueryFactory query;
  
    public ItemQueryRepositoryV2(EntityManager em) {
        this.query = new JPAQueryFactory(em);
    }
  
    public List<Item> findAll(ItemSearchCond cond) {
        return query.select(item)
          .from(item)
          .where(
              maxPrice(cond.getMaxPrice()),
              likeItemName(cond.getItemName()))
          .fetch();
    }
  
    private BooleanExpression likeItemName(String itemName) {
        if (StringUtils.hasText(itemName)) {
            return item.itemName.like("%" + itemName + "%");
        }
        return null;
    }
  
    private BooleanExpression maxPrice(Integer maxPrice) {
        if (maxPrice != null) {
            return item.price.loe(maxPrice);
        }
        return null;
    }
}
```

`ItemQueryRepositoryV2`는 Querydsl 을 사용해서 복잡한 쿼리 문제를 해결할 수 있습니다.

Querydsl 을 사용한 쿼리 문제에 집중되어 있어서, 복잡한 쿼리는 이 부분만 유지보수 하면 되는 장점이 있습니다.

**ItemServiceV2**

```java
@Service
@RequiredArgsConstructor
@Transactional
public class ItemServiceV2 implements ItemService {

    private final ItemRepositoryV2 itemRepositoryV2;
    private final ItemQueryRepositoryV2 itemQueryRepositoryV2;

    @Override
    public Item save(Item item) {
        return itemRepositoryV2.save(item);
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = findById(itemId).orElseThrow();
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }

    @Override
    public Optional<Item> findById(Long id) {
        return itemRepositoryV2.findById(id);
    }

    @Override
    public List<Item> findItems(ItemSearchCond cond) {
        return itemQueryRepositoryV2.findAll(cond);
    }
}
```

기존 `ItemServiceV1` 코드를 남겨두기 위해서 `ItemServiceV2` 을 새로 생성했습니다.

`ItemServiceV2`는 `ItemRepositoryV2`와 `ItemQueryRepositoryV2`를 의존합니다.

`V2Config`

```java
@Configuration
@RequiredArgsConstructor
public class V2Config {
  
    private final EntityManager em;
    private final ItemRepositoryV2 itemRepositoryV2; //SpringDataJPA
  
    @Bean
    public ItemService itemService() {
        return new ItemServiceV2(itemRepositoryV2, itemQueryRepository());
    }
  
    @Bean
    public ItemQueryRepositoryV2 itemQueryRepository() {
        return new ItemQueryRepositoryV2(em);
    }
  
    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepositoryV3(em);
    }
}
```

`ItemServiceV2`를 등록한 부분을 주의합시다. `ItemServiceV1`이 아니라 `ItemServiceV2` 입니다.

`ItemRepository`는 테스트에서 사용하므로 여전히 필요합니다.

**ItemServiceApplication - 변경**

```java
//@Import(QuerydslConfig.class)
@Import(V2Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {}
```

`V2Config`를 사용하도록 변경합니다.

> 참고: 스프링 데이터 JPA가 제공하는 커스텀 리포지토리를 사용해도 비슷하게 문제를 해결할 수는 있습니다.
> 

**이런 식으로 스프링 데이터 JPA 와 QueryDSL 을 모두 사용할 수 있습니다!**


# 3. 다양한 데이터 접근 기술 조합

### **어떤 데이터 접근 기술을 선택하는 것이 좋을까요?**

이 부분은 하나의 정답이 있다기 보다는, 비즈니스 상황과, 현재 프로젝트 구성원의 역량에 따라서 결정하는 것이 맞습니다. 

`JdbcTemplate`이나 `MyBatis` 같은 기술들은 SQL을 직접 작성해야 하는 단점은 있지만 기술이 단순하기 때문에 SQL에 익숙한 개발자라면 금방 적응할 수 있습니다. 

JPA, 스프링 데이터 JPA, Querydsl 같은 기술들은 개발 생산성을 혁신할 수 있지만, 학습 곡선이 높기 때문에, 이런 부분을 감안해야 합니다. 그리고 매우 복잡한 통계 쿼리를 주로 작성하는 경우에는 잘 맞지 않습니다. 

**JPA, 스프링 데이터 JPA, Querydsl을 기본**으로 사용하고, 만약 복잡한 쿼리를 써야 하는데, 해결이 잘 안되면 해당 **부분에는 JdbcTemplate이나 MyBatis를 함께 사용**하는 방법을 추천합니다! 

**실무에서 95% 정도는 JPA, 스프링 데이터 JPA, Querydsl 등으로 해결하고, 나머지 5%는 SQL을 직접 사용**해야 하니 JdbcTemplate이나 MyBatis로 해결합니다. 물론 이 비율은 프로젝트마다 당연히 다릅니다. 

아주 복잡한 통계 쿼리를 자주 작성해야 하면 JdbcTemplate이나 MyBatis의 비중이 높아질 수 있습니다.

### **트랜잭션 매니저 선택**

JPA, 스프링 데이터 JPA, Querydsl은 모두 JPA 기술을 사용하는 것이기 때문에 트랜잭션 매니저로 `JpaTransactionManager`를 선택하면 됩니다. 

해당 기술을 사용하면 스프링 부트는 자동으로 `JpaTransactionManager`를 스프링 빈에 등록합니다. 

그런데 `JdbcTemplate`, `MyBatis`와 같은 기술들은 내부에서 JDBC를 직접 사용하기 때문에 `DataSourceTransactionManager`를 사용합니다. 

따라서 JPA와 JdbcTemplate 두 기술을 함께 사용하면 트랜잭션 매니저가 달라집니다. 결국 트랜잭션을 하나로 묶을 수 없는 문제가 발생할 수 있습니다.

### JpaTransactionManager의 다양한 지원

`JpaTransactionManager`는 놀랍게도 `DataSourceTransactionManager`가 제공하는 기능도 대부분 제공합니다. 

JPA라는 기술도 결국 내부에서는 DataSource와 JDBC 커넥션을 사용하기 때문입니다. 따라서 JdbcTemplate , MyBatis 와 함께 사용할 수 있습니다. 결과적으로 `JpaTransactionManager`를 하나만 스프링 빈에 등록하면, **JPA, JdbcTemplate, MyBatis 모두를 하나의 트랜잭션으로 묶어서 사용할 수 있습니다.**

**주의점** 

이렇게 JPA와 JdbcTemplate을 함께 사용할 경우 JPA의 플러시 타이밍에 주의해야 합니다. JPA는 데이터를 변경하면 변경 사항을 즉시 데이터베이스에 반영하지 않습니다. 기본적으로 트랜잭션이 커밋되는 시점에 변경 사항을 데이터베이스에 반영합니다. 

그래서 하나의 트랜잭션 안에서 JPA를 통해 데이터를 변경한 다음에 JdbcTemplate을 호출하는 경우 JdbcTemplate에서는 JPA가 변경한 데이터를 읽기 못하는 문제가 발생합니다. 

이 문제를 해결하려면 JPA 호출이 끝난 시점에 JPA가 제공하는 플러시라는 기능을 사용해서 JPA의 변경 내역을 데이터베이스에 반영해주어야 합니다. 그래야 그 다음에 호출되는 JdbcTemplate에서 JPA가 반영한 데이터를 사용할 수 있습니다.

# **정리**

위에서 만든 `ItemServiceV2`는 스프링 데이터 JPA를 제공하는 `ItemRepositoryV2`도 참조하고, Querydsl과 관련된 `ItemQueryRepositoryV2`도 직접 참조합니다. 덕분에 `ItemRepositoryV2`를 통해서 스프링 데이터 JPA 기능을 적절히 활용할 수 있고, ItemQueryRepositoryV2 를 통해서 복잡한 쿼리를 Querydsl로 해결할 수 있습니다.

이렇게 하면서 구조의 복잡함 없이 단순하게 개발을 할 수 있습니다. 

본인이 진행하는 프로젝트의 규모가 작고, 속도가 중요하고, 프로토타입 같은 시작 단계라면 이렇게 단순하면서 라이브러리의 지원을 최대한 편리하게 받는 구조가 더 나은 선택일 수 있습니다. 하지만 이 구조는 리포지토리의 구현 기술이 변경되면 수 많은 코드를 변경해야 하는 단점이 있습니다.

이런 선택에서 하나의 정답은 없습니다. 

**이런 트레이드 오프를 알고, 현재 상황에 더 맞는 적절한 선택을 하는 좋은 개발자가 있을 뿐이죠!**

