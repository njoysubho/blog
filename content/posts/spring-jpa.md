---
title: "Spring Jpa"
date: 2022-05-08T10:41:13+02:00
draft: false
categories: ["Platform and reliability"]
---

## Introduction

We started with a simple hello world application and we covered up to how to set up our database schema flyway. We are now ready to write some code which will interact with database. Before we jump on to code let’s look at a bit of history. Java has a nice JDBC api which helps us query database. Making it a base many ORM tools come into existence, Hibernate, Mybatis, Toplink to name a few. ORMs bridged the gap between JDBC and objet orientedness with how we perform database operations and mapped them to some objects. Though we have many different opinions about penalty of using ORMs but in practical we see a huge adotion of this tools and specially Hibernate.

JPA is a specification and is an attempt to uniform the APIs used by many ORMs . If we look at Java based applications today JPA+Hibernate has become the defacto choice for relational databases.

Spring brings more utility that makes life a lot easier for developer.

Now this post is not a Hibernate or JPA tutorial rather it is a simple Spring tutorial on how to use Spring’s support for JPA and Hibernate.

## The dependencies

Like always we have a starter named `spring-boot-starter-jpa` . Below is the dependency

``` xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
```

What does it bring to the application ?

![spring-data-jpa-pom](/spring-data-jpa-pom.png)

We see it brings hibernate core dependencies and JPA dependencies. A small note here JPA which was previously used to be known as Java Persistence API has now renamed to Jakarta Persistence API due to naming rights issues.

What Spring data jpa provides is

- A Repository interface to auto generate most boiler plate query pattern.
- Support for annotation driven transaction mechanism .
- Easy auditing for entities .
- Support for paginations, filter and so on.

So now that we have our dependencies in place, let’s start with code.

Our entity class is simple and looks like below

``` java
@Getter
@Setter
@Entity
@Table(schema = "inv", name = "products")
public class Product{
    @Id
    @GeneratedValue
    private UUID id;
    private String name;
    private Long stock;
    private String manufacturer;
    @CreatedDate
    private OffsetDateTime createdOn;
}
```

(imports are ommitted for brevity)

It’s a simple JPA entity with id field as identifier. Now comes the best part, we ofcourse need some code to store and retrieve products from database. Good news is Spring has got you covered. All you need to do is define a repository as below -

## Repository

``` java
@Repository
public interface ProductRepository extends JpaRepository<Product, UUID>{

}
```

That’s it spring will generate all boiler plate basic query like persists, findAll to name a few . JpaRepository also supports generating query to find by some column of the entity for example id, name, stock, manufacturer,created on. All we need is a method named `findBy<propertyName>`. For a list of supported methods read <https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.query-methods.query-creation>

Let’s try to fetch a product by id .

Below is my `ProductService` which takes the `Product` DTO as input and stores into database.

``` java
@Service
@RequiredArgsConstructor
public class ProductService{
    private final ProductRepository productRepository;

    public Product save(Product productDTO){
        ProductEntity productEnity = new ProductEntity();
        productEnity.setName(productDTO.getName());
        productEnity.setManufacturer(productDTO.getManufacturer());
        productEnity.setStock(productDTO.getStock());
        productEnity.setCreatedOn(OffsetDateTime.now());

        ProductEntity savedEntity = productRepository.save(productEnity);

        return toProductDTO(savedEntity);
    }

    private Product toProductDTO(ProductEntity productEntity){
        return Product.builder()
                .id(productEntity.getId())
                .createdOn(productEntity.getCreatedOn())
                .manufacturer(productEntity.getManufacturer())
                .name(productEntity.getName())
                .stock(productEntity.getStock())
                .build();
    }
}
```

All we need to do after creating our entity is to call `productRepository.save(productEnity)` .

    I have not used any transactions, because JpaRepository itself works in a transaction. Also in this simple example I am not lazily loading any properties from the entity so it is fine to ommit the transaction .

## Logging

We may want to check what SQL hibernate is generating for that we can use below properties

``` yml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
```

In output we will see below logs

``` sql
Hibernate:
    insert
    into
        inv.products
        (created_on, manufacturer, name, stock, id)
    values
        (?, ?, ?, ?, ?)
```

What if we want to see the actual inputs passed in the insert statement. Well there is no direct property but we can enable logs like below

``` yml
logging:
  level:
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

Our application will churn out logs -

    2022-05-08 12:09:36.682 TRACE 40492 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [TIMESTAMP] - [2022-05-08T12:09:36.651572+02:00]
    2022-05-08 12:09:36.683 TRACE 40492 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [VARCHAR] - [test-mfg1]
    2022-05-08 12:09:36.683 TRACE 40492 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [3] as [VARCHAR] - [test-product5]
    2022-05-08 12:09:36.683 TRACE 40492 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [4] as [BIGINT] - [100]
    2022-05-08 12:09:36.683 TRACE 40492 --- [nio-8080-exec-1] o.h.type.descriptor.sql.BasicBinder      : binding parameter [5] as [OTHER] - [dd7d89fa-c201-4338-871f-a00fed464bd5]

Let’s try to get all products -

We again going to take help from Spring JPA repositories and our code will look something like -

``` java
    public List<Product> getAllProducts(){
       return productRepository.findAll()
                .stream()
                .map(this::toProductDTO)
                .collect(Collectors.toList());
    }
```

## Pagination

Well the above works but there’s a basic issue, there may be a huge number of products, in that case we need pagination support. Spring repositories are here to rescue us again. We need to modify our `ProductRepository` class a little -

``` java
@Repository
public interface ProductRepository extends PagingAndSortingRepository<ProductEntity, UUID>{

}
```

Our method signatures change this time and most of the method takes an `Pageable` type . The modified method looks like

``` java
    public Page<Product> getAllProducts(Pageable pageRequest){
       return productRepository.findAll(pageRequest)
                .map(this::toProductDTO);
    }
```

Notice how the return type changes from `List<Product>` to `Page<Product>` . A Page type contains information like total page count and total items.

We can also verify in our application log that select queries are not using `limit` and `offset` instead of doing a select all .

``` sql
Hibernate:
    select
        productent0_.id as id1_0_,
        productent0_.created_on as created_2_0_,
        productent0_.manufacturer as manufact3_0_,
        productent0_.name as name4_0_,
        productent0_.stock as stock5_0_
    from
        inv.products productent0_ limit ? offset ?
```

------------------------------------------------------------------------

**Note**

Response from our controller also needs to be changed to accomodate the new pagination information.

------------------------------------------------------------------------

## Auditing

If we look into our `save` method in ProductService, we are setting the value of `createdOn` field to be current date time. Although in this demo context there is nothing wrong in doing that , there is a better way to populate this fields.

Spring data jpa provides auditing fetaure via `AuditingEntityListener`. This provide a bunch of annotations that populate a field before or after an event.

Let’s try to populate our `createdOn` field.

- We first need to add `@EntityListeners(AuditingEntityListener.class)` to our `ProductEntity` class.
- We need to provide a bean of type `DateTimeProvider` which will be responsible for providing a current time. Because we are using `OffsetDatetime` we create a bean like below which gives an `OffsetDatetime`.

``` java
   @Bean
    public DateTimeProvider dateTimeProvider(){
        return ()-> Optional.of(OffsetDateTime.now());
    }
```

- Finally, we add the following annotation `@EnableJpaAuditing(dateTimeProviderRef = "dateTimeProvider")` .

Just like timestamp we can also add an `auditorAwareRef` which returns an `AuditorAware<T>` . Let’s add a new column to our `ProductEntity`

``` java
    @CreatedBy
    private String createdBy;
```

and we add a bean like below

``` java
    @Bean
    public AuditorAware<String> auditorAwareRef(){
        return () -> Optional.of("test-user");
    }
```

if we now create a new product we will see `test-user` has been set as createdBy in database.

------------------------------------------------------------------------

***Note***

Adding a constant `test-user` is only for example purpose. Getting the real user name may involve getting it from ThreadLocal, SecurityContext, Auth Header or from anything else suitable for your context.

------------------------------------------------------------------------

We also have other annotation `LastModifiedBy` and `LastModifiedOn` to capture modification auditing.

## Some More features

- `@Query` - Sometime even the provided repository methods will not be enough for our usecase. May be we need a more complex query, in that case we can add a method and use `@Query` annotation to specify our sql query. If we set `native=true` we can provide a native SQL query not a JPQL one.

- `Specification` - Our repository can also inherit from JpaSpecificationExecutor which provides method that takes an `Specification` type . We can make use of JPA `Criteria` query to build more nuanced and complex queries.

Spring data jpa is a big module and not everything can be covered in a single post. However we now can create an entity and know how to persist it and query it. In future posts we will see more features from spring-data-jpa.

That’s it from this post .
