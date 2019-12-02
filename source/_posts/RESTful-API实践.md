title: RESTful API实践
date: 2017-08-17 21:05:09
tags:
---


RESTful API对于后端开发来说，就如同UI对于前端开发一样，我们希望尽可能的保证API提供的服务是规范，易懂，友好的。所以需要遵守一些规范和实践，但也不要过度纠结一些标准，Github API也有地方和标准不一样，但一样是业界的标准。本篇博客不会罗列REST API的规范，只会写写自己的部分实践。

---

<!-- more -->
### HATEOAS(Hypermedia as the Engine of Application State)
HATEOAS是一项规范，在API的返回中，使用超链接来进行导航，比如
```
{

    "name": "Alice",

    "links": [ {

        "rel": "self",

        "href": "http://localhost:8080/customer/1"

    } ]

}
```
Spring hateoas库提供了很好的支持。它有下面几个常用的概念：
`Resource`表示一个实体类的封装，它封装了一些Link。
`Resources`表示Resource的集合。
`PagedResources`表示一个Resources集合加一些分页的信息。
`ResourceAssembler`表示将一个实体类转换成Resource对象。

比如我们已经有个User的Entity类，要对它实现hateoas，思路如下：
定义UserResource用来存储User信息以及Link的信息，定义UserResourceAssembler用来将User实体类转换成UserResource类，在这个过程中，可以隐藏你不想从数据库暴露的信息，以及添加Link的信息。
```
class UserResource extends ResourceSupport { … }

 class UserResourceAssembler extends ResourceAssemblerSupport<User, UserResource> { … }
```

然后使用的时候
```
PagedResourcesAssembler<User> parAssembler = … // Autowired
UserResourceAssembler userResourceAssembler = … // Autowired

Page<User> users = userRepository.findAll();

// 遍历users，将user转换成userResource，pagedUserResource对象包含了UserResource的集合，一些Link信息，以及分页信息
PagedResources<UserResource> pagedUserResource = parAssembler.toResource(users, userResourceAssembler);
```

---
### Criteria API
当使用JPA进行条件查询的时候，比如按名字查找，`findByName(String name)`就可以轻松实现，但是当查询的条件很多的时候，用这种方式就需要定义很多的方法。这个时候使用Criteria API就非常方便了。
让你的Repository类继承`JpaSpecificationExecutor`，它提供一些方法，接收`Specification`作为参数，Specification表示一些查询的集合，你可以根据你的查询条件，构造一个Specification，也可以构造多个Specification。
Spring也提供了Fluent API的接口，很方便。使用例子如下：
```
public class SpecificationFactory {
    public static Specification like(String attribute, String value) {
        return (root, query, cb) -> cb.like(root.get(attribute), "%" + value + "%");
    }

    public static Specification equal(String attribute, String value) {
        return (root, query, cb) -> cb.equal(root.get(attribute), value);
    }
}
```

```
SearchParameter searchParameter;
Specification specification = Specifications
        .where(equal("name", searchParameter.getName()))
        .and(like("email", searchParameter.getEmail()));

Page<User> users = repository.findAll(specification);
```

但是上面这种方式有个弊端，当查询的参数可以为空的时候，就不好处理了。所以封装了方法来解决这个问题。
```
public class SpecificationHelper<T> {
    private List<Predicate> predicates;
    private Root<T> root;
    private CriteriaBuilder criteriaBuilder;
    private CriteriaQuery criteriaQuery;

    private SpecificationHelper(Root<T> root, CriteriaQuery criteriaQuery, CriteriaBuilder criteriaBuilder) {
        this.predicates = new ArrayList<>();
        this.root = root;
        this.criteriaBuilder = criteriaBuilder;
        this.criteriaQuery = criteriaQuery;
    }

    public static <T> SpecificationHelper<T> create(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {
        return new SpecificationHelper<>(root, query, criteriaBuilder);
    }

    public SpecificationHelper<T> nameEquals(String name) {
        eq("name", name);
        return this;
    }

    public SpecificationHelper<T> emailEquals(String email) {
        eq("email", email);
        return this;
    }

    public SpecificationHelper<T> eq(String tableName, String value) {
        assert tableName != null;
        if (value != null) {
            predicates.add(criteriaBuilder.equal(root.get(tableName), value));
        }
        return this;
    }

    public SpecificationHelper<T> condition(Specification<T> specification) {
        if (specification != null) {
            predicates.add(specification.toPredicate(root, criteriaQuery, criteriaBuilder));
        }
        return this;
    }

    public Predicate build() {
        return criteriaBuilder.and(predicates.stream().filter(p -> p != null).toArray(Predicate[]::new));
    }
}
```
使用这个类也很简单
```
public class UserSpecfication {
    public static Specification<UserEntity> findUsers(String name, String email) {
        return (root, query, cb) -> SpecificationHelper
                .create(root, query, cb)
                .nameEquals(name)
                .emailEquals(email)
                .condition(nameEquals(name))
                .build();
    }

    public static Specification<UserEntity> nameEquals(String name) {
        return (root, query, cb) -> name == null? null: cb.equal(root.get("name"), name);
    }
}
```

---
### 认证
API的认证方式有多种，常见的有`Basic`,`Token`,`JWT`,`OAuth`。

#### Basic
`Basic`即每次访问API都需要提供username和password。优点即代码非常简单，缺点是安全性很低。

```
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests().antMatchers("/**").hasAnyRole("ADMIN")
                .and()
                .httpBasic()
                .and()
                .csrf().disable();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("username").password("password").roles("ADMIN")
                .and()
                .withUser("test").password("test").roles("TEST");
    }
}
```
#### Token
`Token`即客户端先通过username和password调认证的服务，得到一个随机的Token，这个Token会被存到服务端，然后客户端在调其他服务的时候，带上这个Token即可。
优点：

* 编码简单
* 当想撤销Token的时候，只需要从数据库删除即可

缺点：

* 每次请求都会去查数据库
* 当调用API的人很多的时候，Token的存储量大幅增长，可能导致认证这步成为瓶颈。

#### JWT
上面的Token一般是随机生成的，而使用Json Web Token，允许客户端携带一些payload，然后服务端通过secret key来加密生成一个Token返回给客户端。然后之后的请求，服务端会使用这个key验证客户端发过来的Token是否有效。
优点：

* 服务端的存储问题没有了
* 客户端编码简单

缺点：

* JWT的大小比上面的Token要大
* 服务端的key如果泄露了，那么它就能发有效的token给服务端了。
* 不容易实现撤销Token。



---

### 参考文档
[Criteria API](https://jverhoelen.github.io/spring-data-queries-jpa-criteria-api/)
[JWT](https://auth0.com/blog/securing-spring-boot-with-jwts/)
