## SpringCache配置多CacheManager

###  背景

> ​	Spring为了减少数据的执行次数（重点在数据库查询方面）, 在其内部使用aspectJ技术，为执行操作的结果集做了一层缓存的抽象。这极大的提升了应用程序的性能。由于其切面注入的特性，所以不会对我们的程序造成任何的影响。对于一些实时性要求不那么高的业务数据，我们可以在Service上进行一些缓存的操作。这样就可以减少访问数据库的频率。



### 默认缓存的使用顺序

> ​	在Spring内部，缓存的实现，依赖`org.springframework.cache.Cache`与`org.springframework.cache.CacheManager`共同协作，它们只是定义了一种规范接口，实际的存储规则，需要用户自己定义，当没有提供用户自定义Bean对象，SpringBoot会自动执行以下的检测顺序：

1. [Generic](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-generic)

2. [JCache (JSR-107)](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-jcache) (EhCache 3, Hazelcast, Infinispan, and others)

3. [EhCache 2.x](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-ehcache2)

4. [Hazelcast](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-hazelcast)

5. [Infinispan](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-infinispan)

6. [Couchbase](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-couchbase)

7. [Redis](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-redis)

8. [Caffeine](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-caffeine)

9. [Simple](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#boot-features-caching-provider-simple)

   **具体的用法请参考：[Spring Caching](<https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html>)**



**Cache**

cache接口的作用是Spring提供的一种抽象操作规范，里面包含的`crud操作`

**CacheManager**

cacheManager接口的作用是用来获取Cache，类似一种对象工厂，所有的Cache，必须依赖与CacheManager来获取。



在Spring内部提供了三个默认的实现：

- SimpleCacheManager  简单的集合实现
- ConcurrentMapCacheManager 内部使用ConcurrentHashMap作为缓存容器
- NoOpCacheManager 一个空的实现，不会保存任何记录。



当然其他的依赖需要我们引入三方的Jar，以及一些自定义的配置。



下面看下，执行切面的具体实现：

```java
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {

   @Override
   @Nullable
   public Object invoke(final MethodInvocation invocation) throws Throwable {
      Method method = invocation.getMethod();

      CacheOperationInvoker aopAllianceInvoker = () -> {
         try {
            return invocation.proceed();
         }
         catch (Throwable ex) {
            throw new CacheOperationInvoker.ThrowableWrapper(ex);
         }
      };

      try {
         return execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
      }
      catch (CacheOperationInvoker.ThrowableWrapper th) {
         throw th.getOriginal();
      }
   }

}
```

使用`AOP`的方式，将拦截添加缓存注解的方法，然后会调用父类的方法去执行：

`execute`方法中，会去寻找缓存的操作源`CacheOperationSource`，操作源中包含一个`CacheOperation`的集合。`CacheOperation`代表着`@Cachable`上注解信息的的实体类。然后我们可以用过`cacheManager`属性去寻找对应的`cacheManager`实现，获取`Cache`来完成缓存操作。



### 实现多CacheManager

​	前面分析了，缓存的大致实现过程，我们就可以使用`cacheManager`去有选择性的选择我们需要使用的缓存实现。

​	但是这里有个注意的点，我们需要给CacheManager一个默认的实现，这是由于Spring容器初始化机制造成的。`CacheAspectSupport`抽象实现了`SmartInitializingSingleton`接口，这个接口在Spring容器中，是单例对象的回调接口，当所有的单例对象实例化完毕，就会调用此方法。

​	我们观察下`CacheAspectSupport`的逻辑：

```java
public void afterSingletonsInstantiated() {
   if (getCacheResolver() == null) {
      // Lazily initialize cache resolver via default cache manager...
      Assert.state(this.beanFactory != null, "CacheResolver or BeanFactory must be set on cache aspect");
      try {
         setCacheManager(this.beanFactory.getBean(CacheManager.class));
      }
      catch (NoUniqueBeanDefinitionException ex) {
         throw new IllegalStateException("No CacheResolver specified, and no unique bean of type " +
               "CacheManager found. Mark one as primary or declare a specific CacheManager to use.");
      }
      catch (NoSuchBeanDefinitionException ex) {
         throw new IllegalStateException("No CacheResolver specified, and no bean of type CacheManager found. " +
               "Register a CacheManager bean or remove the @EnableCaching annotation from your configuration.");
      }
   }
   this.initialized = true;
}
```

​	` setCacheManager(this.beanFactory.getBean(CacheManager.class));`，默认是从容器中拿到`CacheManager`对象，这就会出现一个问题，当配置多个`CacheManager`实例bean，并暴露给容器后，会出现Bean装载的错误，因为`getBean`，默认只会返回一个对象，当出现了两个Bean实例就会报错，找不到Bean对象。

​	所以，当对于自定义的多个Bean，我们需要指定一个打上`@Primary`注解，这样就可以解决冲突。



#### 具体实现

​	接下来，贴出具体示例：

我们选择ehcache与redis作为多种缓存源。

**以下示例全部基于`2.1.4.RELEASE`版本，不同版本的代码差异较大**

1. pom文件

   ```xml
   <dependency>
               <groupId>net.sf.ehcache</groupId>
               <artifactId>ehcache</artifactId>
           </dependency>
   
           <dependency>
               <groupId>com.google.guava</groupId>
               <artifactId>guava</artifactId>
               <version>27.1-jre</version>
           </dependency>
     <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-data-redis</artifactId>
           </dependency>
   <dependency>
               <groupId>org.apache.commons</groupId>
               <artifactId>commons-pool2</artifactId>
           </dependency>
     <dependency>
               <groupId>com.h2database</groupId>
               <artifactId>h2</artifactId>
               <scope>runtime</scope>
           </dependency>
        <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-data-jpa</artifactId>
           </dependency>
   ```

   我选择使用内置的H2+JPA作为数据持久层。因为这样比较方便。。不用连接MYSQL。

2. application.yml配置

   ```yaml
   spring:
     application:
       name: authority-management
     datasource:
       driver-class-name: org.h2.Driver
       url: jdbc:h2:~/test
       password:
       username: sa
     h2:
       console:
         enabled: true
     jpa:
       generate-ddl: true
       hibernate:
         ddl-auto: create-drop
       show-sql: true
     redis:
       host: XXXX
       port: 6379
       lettuce:
         pool:
           max-active: 8
           max-wait: 200
           max-idle: 8
           min-idle: 2
       timeout: 2000
     cache:
       ehcache:
         config: classpath:ehcache.xml
       type: EHCACHE
   ```

3.  编写配置类

   ```java
   @Configuration
   @EnableCaching
   @EnableConfigurationProperties(CacheProperties.class)
   public class CacheManagerConfiguration {
       private final CacheProperties cacheProperties;
       public CacheManagerConfiguration(CacheProperties cacheProperties)    {
           this.cacheProperties = cacheProperties;
       }
       public interface CacheManagerNames {
           String REDIS_CACHE_MANAGER = "redisCacheManager";
           String EHCACHE_CACHE_MANAGER = "ehCacheManager";
       }
   
       @Bean(name = CacheManagerNames.REDIS_CACHE_MANAGER)
       public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
           Map<String, RedisCacheConfiguration> expires = ImmutableMap.<String, RedisCacheConfiguration>builder()
                   .put("15", RedisCacheConfiguration.defaultCacheConfig().entryTtl(
                           Duration.ofMillis(15)
                   ))
                   .put("30", RedisCacheConfiguration.defaultCacheConfig().entryTtl(
                           Duration.ofMillis(30)
                   ))
                   .put("60", RedisCacheConfiguration.defaultCacheConfig().entryTtl(
                           Duration.ofMillis(60)
                   ))
                   .put("120", RedisCacheConfiguration.defaultCacheConfig().entryTtl(
                           Duration.ofMillis(120)
                   ))
                   .build();
   
           RedisCacheManager redisCacheManager = RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(factory)
                   .withInitialCacheConfigurations(expires)
                   .build();
           return redisCacheManager;
       }
   
       @Bean(name = CacheManagerNames.EHCACHE_CACHE_MANAGER)
       @Primary
       public EhCacheCacheManager ehCacheManager() {
           Resource resource = this.cacheProperties.getEhcache().getConfig();
           resource = this.cacheProperties.resolveConfigLocation(resource);
           EhCacheCacheManager ehCacheManager = new EhCacheCacheManager(
                   EhCacheManagerUtils.buildCacheManager(resource)
           );
           ehCacheManager.afterPropertiesSet();
           return ehCacheManager;
       }
   }
   
   ```

   以上就是具体配置信息，需要主要的是别忘记`@Primary`

   **Service实现**

   ```java
      @Override
       @Cacheable(key = "#userId", cacheNames = CacheManagerConfiguration.CacheNames.CACHE_15MINS,
               cacheManager = CacheManagerConfiguration.CacheManagerNames.EHCACHE_CACHE_MANAGER)
       public User findUserAccordingToId(Long userId) {
           return userRepository.findById(userId).orElse(User.builder().build());
       }
   
       @Override
       @Cacheable(key = "#username", cacheNames = CacheManagerConfiguration.CacheNames.CACHE_15MINS,
               cacheManager = CacheManagerConfiguration.CacheManagerNames.REDIS_CACHE_MANAGER)
       public User findUserAccordingToUserName(String username) {
           return userRepository.findUserByUsername(username);
       }
   ```

   现在，就可以使用cacheManager属性来选择缓存源，用户可以灵活配置。

   除了在方法上，使用注解外，我们还可以直接指定到类上，下面的示例，表示该类的全部方法都使用`Encache`作为缓存源。

   ```java
   @CacheConfig(cacheManager = CacheManagerConfiguration.CacheManagerNames.EHCACHE_CACHE_MANAGER)
   ```

4. 测试类+日志信息

   ```java
    userService.findUserAccordingToId(save.getId());
   userService.findUserAccordingToId(save.getId());
   userService.findUserAccordingToUserName(save.getUsername());
   userService.findUserAccordingToUserName(save.getUsername());
   ```

   

   ```verilog
   Hibernate: select user0_.id as id1_11_0_, user0_.email as email2_11_0_, user0_.image_url as image_ur3_11_0_, user0_.introduction as introduc4_11_0_, user0_.last_login_at as last_log5_11_0_, user0_.level as level6_11_0_, user0_.login_ip as login_ip7_11_0_, user0_.nickname as nickname8_11_0_, user0_.password as password9_11_0_, user0_.retry_login_count as retry_l10_11_0_, user0_.sex as sex11_11_0_, user0_.status as status12_11_0_, user0_.telephone as telepho13_11_0_, user0_.username as usernam14_11_0_ from user user0_ where user0_.id=?
   2019-05-12 20:43:06.486  INFO 12020 --- [           main] io.lettuce.core.EpollProvider            : Starting without optional epoll library
   2019-05-12 20:43:06.488  INFO 12020 --- [           main] io.lettuce.core.KqueueProvider           : Starting without optional kqueue library
   2019-05-12 20:43:06.807  INFO 12020 --- [           main] o.h.h.i.QueryTranslatorFactoryInitiator  : HHH000397: Using ASTQueryTranslatorFactory
   Hibernate: select user0_.id as id1_11_, user0_.email as email2_11_, user0_.image_url as image_ur3_11_, user0_.introduction as introduc4_11_, user0_.last_login_at as last_log5_11_, user0_.level as level6_11_, user0_.login_ip as login_ip7_11_, user0_.nickname as nickname8_11_, user0_.password as password9_11_, user0_.retry_login_count as retry_l10_11_, user0_.sex as sex11_11_, user0_.status as status12_11_, user0_.telephone as telepho13_11_, user0_.username as usernam14_11_ from user user0_ where user0_.username=?
   
   Process finished with exit code -1
   ```

   ​	可以看到，两次查询，都只是用了一次select语句，还出现了一次`redis`连接操作，证明上述的配置是生效的。

#### 注意：

1. 我在此使用的`lettuce`作为redis客户端，使用连接池时，它依赖`commons-pool2`
2. 测试时，当redis作为缓存时，发现有时候还是会去数据库查询两次，怀疑是配置的超时时间问题。









