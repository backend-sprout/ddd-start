리포지토리 조회 기능(JPA 중심)  
==============================     
# 검색을 위한 스펙  
**리포지터리는 애그리거트의 저장소이다.**          
애그리거트를 저장하고 찾고 삭제하는 것이 리포지터리의 기본 기능이다.        
       
```java
public interface OrderRepository {
    Order findById(OrderNo id);
    List<Order> findByOrderer(String orderId, Date fromDate, Date toDate);
    ...
}
```    
검색하는 과정도 다양한 검색조건을 이용하여 다양하게 생성할 수 있다.           
이때, 네이밍 규칙으로 `findBy+개채(필드명)`을 지정하면 `DataJPA`에서 자동으로 쿼리를 만들기도한다.           
     
그런데 이렇게 검색 조건의 조합이 다양할 경우 **스펙**을 이용해서 문제를 풀어야한다.           
**스펙**은 애그리거트가 **특정 조건을 충족하는지 여부를 검사한다.**    

```java
public interface Specification<T> {
    public boolean isSatisfiedBy(T agg);
}
```  
`spring.data.jpa`에서는 이러한 스펙 검증을 위한 `Specification`인터페이스를 제공한다.   
조건 대상이 되는 애그리거트 객체를 파라미터에 넣어 `isSatisfiedBy()`에서 검증하여 true/false를 리턴한다.       
    
```java
public class OrdererSpec implements Specification<Order> {
    private String ordererId;
    public OrderSpec(String ordererId) {
        this.ordererId = ordererId;
    }
    public boolean isSatisfiedBy(Order agg) {
        return agg.getOrdererId().getMemberId().getId().equals(ordererId);
    }
}
```
**리포지터리는 스펙을 전달받아 애그리거트를 걸러내는 용도로 사용**한다.   
만약 리포지터리가 메모리에 모든 애그리거트를 보관하고 있다면 다음과 같은 스팩을 사용할 수 있다.   

```java
pubilc class MemoryOrderRepository implements OrderRepository {
    public List<Order> findAll(Sepcification sepc) {
        List<Order> allOrders = findAll();
        return allOrders.stream()
            .filter(order -> spec.isStaisfiedBy(order)).collect(toList());
    }
}
```
`Specification`는 `SimpleJpaRepository`에 한정되어    
`특정 조건`을 구현하여 이를 주입해주면 조건에 맞추어 쿼리가 실행되고 데이터를 반환한다.       
특정 조건을 충족하는 애그리거트를 찾으려면 이제 원하는 스펙을 생성해서 리포지토리에 전달해주기만 하면 된다.     

```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
List<Order> orders = orderRepository.findAll(ordererSpec);
```

## 스펙 조합 
**스펙의 장점은 `조합`이다.**    
여러 스펙을 조합해서 새로운 조건을 만들어 낼 수 있다.  

```java
public class AndSpec<T> implements Specification<T> {
    private List<Specification<T>> specs;
    
    public AndSpecification(Specification<T> ... sepcs) {
        this.specs = Arrays.asList(specs);
    }
    
    public boolean isSatisfiedBy(T agg) {
        for(Specification<t> spec : specs) {
            if(!spec.isSatisfiedBy(agg)) return false;
        }
        return true;
    }
}
```
```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
Specification<Order> orderDateSpec = new OrderDateSpec(fromDate, toDate);
AndSpec<T> spec = new AndSpec(ordererSpec, orderDateSpec);
List<Order> orders = orderRepository.findAll(spec);
```
`Specification`를 구현한 클래스를 통해 다양한 검색 조건을 스펙을 구현할 수 있다.   

# JPA를 위한 스펙 구현 
리포지터리 코드는 모든 애그리거트를 조회한 다음에 스펙을 이용해서 걸러내는 방식을 사용했다.     
하지만, 이 방식에는 실행 속도 문제가 있다.     
데이터의 갯수만큼 로딩한 다음에 다시 데이터의 갯수만큼 루프를 돌면서 검사를 진행하기 때문이다.    
      
실제 구현에서는 쿼리의 where 절에 조건을 붙여서 필요한 데이터를 걸러야 한다.        
이는 스펙 구현도 메모리에서 걸러내는 방식에서 쿼리의 where를 사용하는 방식으로 바뀌어야 한다는 것을 뜻한다.         
JPA는 다양한 검색 조건을 조합하기 위해 CriteriaBuilder와 Prdicate를 지원하므로 이를 이용해서 검색조건을 구하자  
 
## JPA 스펙 구현   
JPA를 사용하는 리포지터리를 위한 스펙의 인터페이스는 아래와 같이 정의할 수 있다.     

```java
public interface Specification<T> {
    public boolean isSatisfiedBy(T agg);
}
```  
```java
public class OrderSpec implements Specification<Order> {
    private String ordererId;
    
    public OrdererSpec(String ordererId) {
        this.ordererId = ordererId;
    }
    
    @Override
    public Prdicate toPredicate(Root<Order> root, CriteriaBuilder cb) {
        return cb.equal(root.get(Order_.orderer)
                          .get(Orderer_.memberId_.get(MemberId_.id), ordererId);
    }
}
```
`OrderSpec`의 `toPredicate()`는 Order의 `orderer.memberId.id`프로퍼티가       
생성자로 전달받은 `ordererId`와 같은지 비교하는 Predicate 를 생성해서 리턴하다.            
   
```java
Specification<Order> ordererSpec = new OrdererSpec("madvirus");
List<Order> orders = orderRepository.findAll(ordererSpec);
```
응용 서비스는 원하는 스펙을 생성하고 리포지터리에 전달해서 필요한 애그리거트를 검색하면 된다.         
               
```java
public class OrdererSpecs {
    public static Specification<Order> orderer(String ordererId) {
        return (root, cb) -> cb.equals(
            root.get(Order_.orderer).get(Orderer_.memberId).get(MemberId_.id), ordererId);
    }
    
    public static Specification<Order> between(Date from, Date to) {
        return (root, cb) -> cb.between(root.get(Order_.orderDate), form, to);
    }
}
```
Specification 구현 클래스를 개별적으로 만들지 않고 별도 클래스에 스펙 생성 기능을 모아도 된다.   
스펙 생성이 필요한 코드는 스펙 생성 기능을 제공하는 클래스를 이용해서 조금 더 간결하게 스펙을 생성할 수 있다. 

```java
Specification<Order> betweenSpec = OrderSpecs.between(fromTime, toTime);
```  

### JPA 정적 메타 모델 
```java
@StaticMetamodel(Order.class)
public abstract class Order_ {
    public static volatile SingularAttribute<Order, OrderNo> number;
    ...
}
```

정적 메타 모델은 `@StaticMetamodel`를 통해서 관련 모델을 지정한다.   
메타 모델 클래스의 이름은 관례적으로 맨 뒤에 `_`를 붙인다. 

정적 메타 모델 클래스는 대상 모델의 각 프로퍼티와 동일한 이름을 갖는 정적 필드를 정의한다.    
이 정적 필드는 프로퍼티에 대한 메타 모델로서 프로퍼티 타입에 따라 여러 타입을 사용해서 메타 모델을 정의한다.   

정적 메타 모델을 사용하는 대신 문자열로 프로퍼티를 지정할 수도 있다.   

```java
root.get("orderer").get("memberId").get("id")
```
하지만 문자열은 오타 가능성이 있고 실행하기 전까지 오타가 있다는 것을 놓치기 싫다.    
이런 이유로 Criteria를 사용할 때에는    
정적 메타 모델 클래스를 사용하는 것이 코드 안정성이나 생산성 측면에서 유리하다.       
   
적적 메타 모델 클래스를 직접 작성할 수 있지만 하이버네이트와 같은 JPA 프로바이더는       
정적 메타 모델을 생성하는 도구를 제공하고 있으므로 이들 도구를 사용하면 편리하다.      
     
## AND/OR 스펙조합을 위한 구현          
JPA를 위한 AND/OR 스펙은 다음과 같이 구현할 수 있다.   

```java
public class AndSpecification<T> implements Specification<T> {
    private List<Specification<T>> specs;
    
    public AndSpecification(Specification<T> ... specs) {
        this.specs = Arrays.asList(specs);
    }
    
    @Override
    public Predicate toPredicate(Root<T> root, CriterialBuilder cb) {
        Predicated[] predicates = specs.stream()
            .map(spec -> spec.toPredicate(root, cb))
            .toArray(size -> new Predicate[size]);
            return cb.and(predicates);
    }
}
```
```java
... OR 스펙은 생략 
```
`toPredicate()`는 생성자로 전달받은 `Specification` 목록으로 바꾸고       
`CriteriaBuilder`의 `and()`를 사용해서 새로운 Predicate를 생성한다.       
     
두 스펙을 쉽게 생성하기 위해 아래와 같은 클래스를 구현할 수 있다.       

```java
public class Specs {
    public static<T> Specification<T> and(Specification<T> ... specs) {
        return new AndSpecification<>(specs);
    }
    
    pubilc static<T> Specification<T> or(Specification<T> ... specs) {
        return new OrSpecification<>(specs);
    }
}
```
이제 스펙 조합이 필요하면 다음과 같은 방법으로 스펙을 조합하면 된다.   

```java
Specification<Order> specs = Specs.and(
    OrderSpecs.orderer("madvirus"), OrderSpecs.between(fromTime, toTime));
```

## 스펙을 사용하는 JPA 리포지터리 구현    
이제 남은 작업은 스펙을 사용하도록 리포지터리를 구현하는 것이다.     
먼저 리포지터리 인터페이스는 스펙을 사용하는 메서드를 제공해야한다.    

```java
public interface OrderRepository {
    public LIst<Order> findAll(Specification<Order> spec);
    ...
}
```
  
이를 상속받은 JPA 리포지토리를 같이 구현할 수 있다.    

```java
@Repository  
public class JpaOrderRepository implments OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<Order> findAll(Specification<Order> spec) {
       CriteriaBuilder cb = entityManager.getCriteriaBuilder();
       CriteriaQuery<Order> criteriaQuery = cb.createQuery(Order.class);  
       Root<Order> root = criteriaQuery.from(Order.class);
       Predicate predicate = spec.toPredicate(root, cb);
       criteriaQuery.where(predicate);
       criteriaQeury.orderBy(
           cb.desc(root.get(Order_.number).get(OrderNo_.number)));
       TypedQuery<Order> query = entityManager.createQuery(criteriaQuery);
       return query.getResultList();
    }
}
```
          
# 정렬 구현   
JPA의 `CriteriaQuery=orderBy()`를 이용해서 정렬 순서를 지정한다.           
`CriteriaBuilder#asc()` 메서드와 `clesc()` 메서드로 정렬할 대상을 지정한다.               
`JPQL`을 사용하는 경우에는 JPQL의 `order by` 절을 사용하면 된다.          
  
```java  
TypedQuery<Order> query = entityManager.createQuery(
    "select o from Order o " +
        "where o.orderer.memberId.id = :ordererId " + 
        "order by o.number.number desc", Order.class);
```
정렬 순서가 고정된 경우에는 `CriteriaQuery#orderBy()`나      
JPQL의 orderby 절을 이용해서 정렬 순서를 지정하면 되지만,       
정렬 순서를 응용 서비스에 결정하는 경우에는 정렬 순서를 리포지터리에 전달할 수 있어야한다.       
           
JPA Criteria는 Order 타입을 이용해서 정렬 순서를 지정한다.         
그런데 JPA의 Order는 CriteriaBuilder를 이용해야 생성할 수 있다.           
정렬 순서를 지정하는 코드는 리포지터리를 사용하는 응용 서비스에 위치하게 되는데       
응용 서비스는 JPA Order가 아닌 다른 타입을 이용해서 리포지터리에 정렬 순서를 전달하고        
JPA 리포지터리는 이를 다시 JPA Order로 변환하는 작업을 해야한다.    

정렬 순서를 지정하는 가장 쉬운 방법은 다음과 같이 문자열을 사용하는 것이다.        
```java
List<Order> orders = orderRepository.findAll(someSpec, "number.number desc");
```
JPA 리포지터리 구현 클래스는 문자열을 파싱해서      
JPA Criteia의 Order로 변환하거나 JPQL의 order by 절로 변환하면된다.       

```java
@Repository 
pubilc class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;  
    ...
    
    @Override
    public List<Order> findAll(Specification<Order> spec, String ... orders) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Order> criteriaQuery = cb.createQuery(Order.class);  
        Root<Order> root = criteriaQuery.from(Order.class);
        Predicate predicate = spec.toPredicate(root, cb);
        criteriaQuery.where(predicate);
        if (orders.length > 0 ) {
            criteriaQuery.orderBy(JpaQueryUtils.toJpaOrders(root, cb, orders));
        }
        TypedQuery<Order> query = entityManager.createQuery(criteriaQuery);
        return query.getResultList();
    }
}
```
`JpaQueryUtils.toJpaOrders()`는 변환을 위해 구현한 보조 클래스로 구현 코드는 아래와 같다.   
  
```java
생략..  
```   
`JpaQueryUtils.toJpaOrders()` 메서드를 사용하면 문자열 배열로부터     
다음과 같이 JPA Order 객체를 생성하므로 다양한 정렬 순서를 지정할 수 있다.     
  
* name desc -> cb.desc(root.get( "name" )); 
* customer.name asc -> cb.asc(root.get( "customer" ).get( "name" ));
 
JPQL을 사용하는 리포지터리를 위해 JpaQueryUtils에 메서드를 추가로 구현한다.      
    
```java
public class JpaQueryUtils {
    ...(생략)
    
    public static String toJPQLOrderby(String alias, String ... orders) {
        if(orders == null || orders.length == 0) return "";
        String orderParts = Arrays.stream(orders)
            .map(order -> alias + "." + order)
            .collect(joining(","));
        return "order by " = orderParts;    
    }
}
```
JPQL을 사용하는 리포지터리 코드는 
다음과 같이 `toJPQLOrderBy()` 메서드를 사용해서 알맞은 `order by` 절을 생성할 수 있다.          

```java
TypedQuery<Order> query = entityManager.createQuery(
    "select o from Order o where o.orderer.memberId.id = :ordererId " +  
        JpaQueryUtils.toJPQLOrderby("o", "number.number desc"), Order.class);
)
```
...생략하겠습니다.... 

# 페이징과 개수 구하기 구현  
JPA 쿼리는 `setDirstResult()`, `setMaxResult()`를 제공하고 있는데    
이 두 메서드를 이용해서 페이징을 구현할 수 있다.  


```java
@Override
pubilc List<Order> findByOrdererId(String ordererId, int startRow, int fetchSize) {
    TypedQuery<Order> query = entityManager.createQuery(
        "select o from Order o " +
            "where o.orderer.memberId.id = :ordererId " +
            "order by o.number.number desc"), Order.class);
    query.setParameter("ordererId", ordererId);
    query.setFirstResult(startRow);
    query.setMaxResult(fetchSize);
    return query.getResultList();
}
```

* setFirstResult(startRow) : 읽어올 첫 번째 행 번호를 지정합니다.       
* setMaxResult(fetchSize) : 읽어올 행 개수를 지정한다.     

시작행 값과 결과 개수를 파라미터로 전달하면 4페이지에 해당하는 데이터 결과를 구할 수 있게된다.   
```java
List<Order> orders = findbyOrdererId("madvirus", 45, 15);
```   
  
페이징과 함께 사용되는 기능이 전체 개수를 구하는 기능이다.     
전체 개수를 구하는 기능은 JPQL을 이용해서 간단하게 구현할 수 있다.      

```java
@Repository
public class JpaOrderRepository implements OrderRepository {
    ...
    @Override
    public Long countsAll() {
        TypeQuery<Long> query = entityManager.createQuery(
            "select count(o) from Order o", Long.class);
        )
        return query.getSingleResult();
    }
}
```
Specification을 이용해서 특정 조건을 충족하는 애그리거트의 개수를 구하고 싶다면 아래와 같은 코드를 사용한다.   

```java
@Repository
public class JpaOrderRepository implements OrderRepository {
    ...
    @Override
    public Long counts(Specification<Order> spec) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Order> criteriaQuery = cb.createQuery(Order.class);  
        Root<Order> root = criteriaQuery.from(Order.class);
        Predicate predicate = spec.toPredicate(root, cb);
        criteriaQuery.select(db.count(root)).where(spec.toPredicate(root, cb));
        TypedQuery<Order> query = entityManager.createQuery(criteriaQuery);
        return query.getResultList();
    }
}
```
  
# 조회 전영 기능 구현  
리포지터리는 애그리거트의 저장소를 표현하는 것으로 다음 용도로 리포지토리를 사용하는 것은 접합하지 않다.   
   
* 여러 애그리거트를 조합해서 한 화면에 보여주는 데이터 제공      
* 각종 통계 데이터 제공    
   
여러 애그리거트를 조합하여 활용하는 것은 즉시로딩/지연로딩/연관 매핑들로 인해 골치가 아플 것이다.     
애그리거트간에 직접 연관을 맺으면 ID로 참조할 때의 장점을 활용할 수 없게 된다.      

각종 통계 데이터 역시 다양한 테이블을 조인하거나 DBMS 전용 기능을 사용해야 구할 수 있는데,      
이는 `JPQ`이나 `Criteria`로 처리하기 힘들다.    

## 동적 인스턴스 생성 
JPA는 쿼리 결과에서 임의의 객체를 동적으로 생성할 수 있는 기능을 제공하고 있다.   


```java
@Override
public List<OrderView> selectByOrderer(String ordererId) {
    String selectQuery = 
        "select new com.myshop.order.application.dto.OrderView(o, m, p)" +
        "from Order o join o.orderLine o1, Member m, Product p " +
        "where o.orderer.memberId.id = :ordererId " +
        "and o.orderer.memberId = m.id " + 
        "and index(o1) = 0 " + 
        "and o1.productId = p.id " + 
        "order by o.number.number desc";
    TypedQuery<OrderView> query = 
        em.createQuery(selectQuery, OrderVlew.class);
    query.setParameter("ordererId", ordererId);
    return query.getResultList();
}
```
이 코드에서 JPQL의 select 절을 보면 new 키워드가 있다.       
new 키워드 뒤에 생성할 인스턴스의 완전한 클래스 이름을 지정하고         
괄호 안에 생성자에 인자로 전달할 값을 지정한다.       
   
이 코드의 경우 OrderView 생성자에 인자로 각각 Order, Member, Product를 전달하고      
생성자는 전달받은 객체로부터 필요한 값을 추출한다.    

```java
public class OrderView {
    private String number;
    private long totalAmounts;
    ...
    private String productName;
}
public class OrderView(Order order, Member member, Product product) {
    this.number = order.getNumber().getNumber();
    this.totalAmoutns = order.getToatalAmounts().getValue();
    ...
    this.productName = product.getName();
}
...
```
조회 전용 모델을 만드는 이유는 표현 영역을 통해 사용자에게 데이터를 보여주기 위함이다.        
많은 웹 프레임워크는 새로 추가한 밸류 타입을 알맞은 형식으로 출력하지 못하므로           
위 코드처럼 값을 기본 타입으로 변환하면 편리하다.        
   
물론, 사용하는 웹 프레임워크에 익숙하다면 Money와 같은 밸류 타입을       
원하는 형식으로 출력하도록 프레임워크를 확장해서      
조회 전용 모델에서 밸류 타입 의미가 사라지지 않도록 할 수 있다.      
   
모델의 개별 프로퍼티를 생성자에 전달할 수도 있다.   
주문 목록을 보여줄 목적으로 OrderView를 사용한다면 생성자로 필요한 값만 받아도 될 것이다.   
즉, 다음과 같이 필요한 값만 전달하도록 JPQL과 생성자를 수정할 수도 있다.     
   
```java
select new com.mysqhop.order.application.dto.orderView(
    o.number.number, o.totalAmounts, o.orderDate, m.id.id, m.name, p.name)
...(생략)
```
동적 인스턴스의 장점은 JPQL을 그대로 사용하므로 객체 기준으로 쿼리를 작성하면서도          
동시에 지연/즉시 로딩과 같은 고민 없이 원하는 모습으로 데이터를 조회할 수 있다는 점이다.    
   
## 하이버네이트 @Subselect 사용   
하이버네이트는 JPA 확장 기능으로 `@Subselect`를 제공한다.       
`@Subselect`는 쿼리 결과를 `@Entity`로 매핑할 수 있는 유용한 기능이다.      

```java
@Entity
@Immutable
@Subselect("select o.order_number as number, " +
           "o.orderer_id, o.orderer_name, o.total_amoutns, " +
           "o.receiver_name, o.state, o.order_date, " + 
           "p.product_id, p.name as product_name " +
           "from purchase_order o inner join order)line ol " +
               "on o.order_number = ol.order_number " +
               "cross join product p " +
               "where ol.line_idx = 0 and ol.product_id = p.product_id")
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
    @Id
    private String number;
    private String ordererId;
    // 생략 
}
```
`@Immutable`, `@Subselect`, `@Synchronize`는 하이버네이트 전용 어노테이션인데          
이 태그를 사용하면 테이블이 아닌 쿼리 결과를 `@Entity`로 매핑할 수 있다.       
    
`@Subselect`는 조회 쿼리를 값으로 갖는다.        
하이버네이트는 이 select 쿼리 결과를 매핑할 테이블처럼 사용한다.(서브쿼리)       
`@Subselect`를 사용하면 쿼리 실행 결과를 매핑할 테이블처럼 사용한다.           
뷰를 수정할 수 없듯이 `@Subselect`로 조회한 `@Entity` 역시 수정할 수 없다.   

`@Subselect`를 이용한 `@Entity`의 매핑 필드를 수정하면    
하이버네이트는 변경 내역을 반영하는 update 쿼리를 실행할 것이다.        
그런데, 매핑한 테이블이 없으므로 에러가 발생한다.        
    
이런 문제를 방지하기 위해 `@Immutable`을 사용한다.       
`@Immutable`을 사용하면 하이버네이트는      
해당 엔티티의 매핑 필드/프로퍼티가 변경되어도 DB에 반영하지 않고 무시한다.     

```java
Order order = orderRepository.findById(orderNumber);
order.changeShippingInfo(newInfo); // 상태 변경 

// 변경 내역이 DB에 반영되지 않았는데 purchase_order 테이블에서 조회  
List<OrderSummary> summaries = orderSummaryRepository.findByOrdererId(userId);
```
위 코든느 Order의 상태를 변경한 뒤에 OrderSummary를 조회하고 있다.     
특별한 이유가 없으면 하이버네이트는 변경사항을 트랜잭션을 커밋하는 시점에 DB에 반영하므로,      
Order의 변경 내역을 아직 purchase_order 테이블에 반영하지 않은 상태에서           
purchase_order 테이블을 사용하는 OrderSummary를 조회하게 된다.           
즉, OrderSummary에는 최신 값이 아니 이전 값이 담기게 된다.     
  
이런 문제를 해소하기 위한 용도로 사용하는 것이 `@Synchronize`이다.        
`@Synchronize`는 해당 엔티티와 관련된 테이블 목록을 명시한다.        
하이버네이트는 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 플러시를 먼저한다.     
OrderSummary의 `@Synchronize`는 `purchase_order` 테이블을 지정하고 있으므로 OrderSummary로 로딩하기 전에      
`purchase_order` 테이블에 변경이 발생하면 관련 내역을 먼저 플러시한다.     
따라서 OrderSummary를 로딩하는 시점에서는 변경 내역이 반영된다.   

`@Subselect`를 사용해도 일반 `@Entity`와 같기 때문에   
`EntityManager.find()`, `JPQL`, `Criteria`를 사용해서 조회할 수 있다는 것이  
`@Subselect`의 장점이다.   

```java
OrderSummary summary = entityManager.find(OrderSummary.class, orderNumber);
TypedQuery<OrderSummary> query = em.createQuery("select os from OrderSummary " + 
    "os where os.ordererId = :ordererId " + 
    "order by os.orderDate desc", OrderSummary.class);
query.setParameter("ordererId", ordererId);
List<OrderSummary> result = query.getResultList();
```

`@Subselect`는 이름처럼 `@Subselect`의 값으로 지정한 쿼리를 from 절의 서브쿼리로 사용한다.    
