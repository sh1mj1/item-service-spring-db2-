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