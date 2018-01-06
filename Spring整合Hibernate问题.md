### 1. java.lang.ClassNotFoundException: org.apache.juli.logging.LogFactory
- 原因：缺少jar包
- 解决方法：在调用的时候需要加入tomcat-juli.jar这个包，此包位于tomcat根目录的bin下。

### 2. A servletContext is required to configure default servlet handing
- 测试类
```java 
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes= {RootConfig.class, WebConfig.class}) 
public class TestHibernate {
}
```

- RootConfig配置类
```java
@Configuration
@EnableWebMvc	// enable spring mvc
@ComponentScan(basePackages = "edu.demon.web")	// enbale component scan
public class WebConfig extends WebMvcConfigurerAdapter{
```
- 原因：因为带有@Configuration注解的配置类中有@EnableWebMvc，这使得即使在测试类只需要简单的Spring上下文，但因为这个注解，会引入Web应用上下文。
- 解决方法：
    - 在测试Hibernate时，把WebConfig上的@EnableWebMvc去掉，或者在测试类的中@ContextConfiguration(classes= {RootConfig.class})//, WebConfig.class}) 不要引入WebConfig类；
    - 在测试类上加@WebAppConfiguration。

### 3. org.hibernate.HibernateException: Could not obtain transaction-synchronized Session for current thread
- 原因：
    - SessionFactory的getCurrentSession并不能保证在没有当前Session的情况下会自动创建一个新的，这取决于CurrentSessionContext的实现，而在Spring提供的CurrentSessionContext的实现SpringSessionContext中，会在开始事务之前通过AOP的方式为当前线程创建Session，此时调用getCurrentSession()将得到正确结果。当Hibernate与Spring集成时，将使用该SessionContext，故此时调用getCurrentSession()的效果完全依赖于SpringSessionContext的实现。
    - 在不使用Spring的情况下使用Hibernate，需要在Hibernate配置类中设置current_session_context_class为thread，也即使用了ThreadLocalSessionContext，那么我们在调用getCurrentSession()时，如果当前线程没有Session存在，则会创建一个绑定到当前线程。
    - 在Spring中使用Hibernate，如果我们配置了TransactionManager，那么我们就不应该调用SessionFactory的openSession()来获得Sessioin，因为这样获得的Session并没有被事务管理。[openSession和getCurrentSession区别](https://www.cnblogs.com/chyu/p/4817291.html)
    - 由于本工程中，一开始没有在RootConfig类中配置事务管理，且dao的实现方法中没有申明事务的边界，因此造成了这个错误。

- 解决办法：在配置文件中添加事务
- #### XML方式中
    a. 在spring 配置文件中加入
```java 
<tx:annotation-driven transaction-manager="transactionManager"/>
```
    b. 在处理业务逻辑的类上采用注解
```java
@Service
            public class CustomerServiceImpl implements CustomerService {  
            @Transactional
            public void saveCustomer(Customer customer) {
            customerDaoImpl.saveCustomer(customer);
            }
            ...
            }
```
    c. 在 hibernate 的配置文件中
```java
<property name="current_session_context_class">thread</property>
```
    
- #### Java config方式中
    a. 在config类中
```java 
@Configuration
@ComponentScan(basePackages= {"edu.demon"}, excludeFilters= {@Filter(type=FilterType.ANNOTATION, value = EnableWebMvc.class)})
@EnableTransactionManagement (proxyTargetClass=true) //启用注解事务管理，使用CGLib代理
public class RootConfig {
                @Bean(name="transactionManager")
	            public HibernateTransactionManager hibernateTransactionManager() {
		            HibernateTransactionManager hibernateTransactionManager = new HibernateTransactionManager();
		            hibernateTransactionManager.setSessionFactory(sessionFactory().getObject());
		            return hibernateTransactionManager;
	            }
            }
```
    b. 在处理业务逻辑的类上采用注解
```java
@Service
public class CustomerServiceImpl implements CustomerService {  
    @Transactional
    public void saveCustomer(Customer customer) {
        customerDaoImpl.saveCustomer(customer);
    }
```

### 4. Could not open Hibernate Session for transaction; nested exception is java.lang.NoClassDefFoundError: org/hibernate/engine/transaction/spi/TransactionContext
- 我用Maven引入的是5.0.2的hibernate core的jar包，但是在配置中使用的Hibernate4的TransactionManager。

- 解决办法

    - 把将换成hibernate-core-5.0.2.Final.jar换成hibernate-core-4.3.4.Final.jar

    - 在配置文件中使用Hibernate5的TransactionManager.


### 5. error code [1364]; org.hibernate.exception.GenericJDBCException: could not execute statement
- 需要检查的点
    
    - MySQL中表中自增长字段是否设置为Auto_Increnement;
    
    - 工程中的POJO中的字段上，
```java
        @Id
	@GeneratedValue(strategy=GenerationType.IDENTITY)
	@Column(name="id")
	public Integer getId() {
		return id;
        }
```

- 
    - 方言
    - 连接数据库的URL后，加编码类型
>   - jdbc:mysql://localhost/SpringDemo?characterEncoding=utf8