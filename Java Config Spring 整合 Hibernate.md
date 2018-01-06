## Java Config Spring 整合 Hibernate

1. 在Eclipse中创建Maven工程，archetype选择webapp 1.0.
2. 在src/main/java中创建config，dao，service和web包。
3. 创建配置类，代替XML配置
    - 在config中，创建继承了AbstractAnnotationConfigDispatcherServletInitializer的类JavaCfgWebAppInitializer，其会自动配置DispatcherServlet和Spring应用上下文。
    > Servlet3.0 会会在类路径中查找实现了javax.servlet.ServletConyaionerInializer接口的类，用来配置Servlet容器。Spring提供了这个接口的实现SpringServletContainerInitializer，这个类会查找实现了WebApplicationInitializer的类，并把配置任务交给它们完成。而Spring3.2提供的AbstractAnnotationConfigDispatcherServletInitializer就是WebApplicationInitializer的实现类。
    
    因此当工程部署到Servlet中，容器会自动发现JavaCfgWebAppInitializer，并用它来配置Servlet上下文。
```java
public class SpitterWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
// 指定DispactcherServlet创建Spring应用上下文的配置类，加载控制器，视图解析器及处理器映射
  @Override
  protected Class<?>[] getRootConfigClasses() {
    return new Class<?>[] { RootConfig.class };
  }
// 指定ContextLoaderListener创建的应用上下文配置类，加载驱动应用后端的中间层和数据层组件
  @Override
  protected Class<?>[] getServletConfigClasses() {
    return new Class<?>[] { WebConfig.class };
  }
// 把DispactcherServlet映射到“／”
  @Override
  protected String[] getServletMappings() {
    return new String[] { "/" };
  }
  
  @Override
  protected void customizeRegistration(Dynamic registration) {
    registration.setMultipartConfig(
        new MultipartConfigElement("/Users/juliang/Documents", 2097152, 4194304, 0));
  }
}
```
```java
@Configuration
@ComponentScan(basePackages= {"edu.demon"}, 
excludeFilters= {@Filter(type=FilterType.ANNOTATION, value = EnableWebMvc.class)})
@EnableTransactionManagement(proxyTargetClass=true) //开启事务管理的注解
public class RootConfig {
	
	
	@Bean(name="dataSource")
	public BasicDataSource dataSource() {
		BasicDataSource ds = new BasicDataSource();
		ds.setDriverClassName("com.mysql.jdbc.Driver");
		ds.setUrl("jdbc:mysql://localhost/SpringDemo?characterEncoding=utf8");
		ds.setUsername("root");
		ds.setPassword("root");
//		ds.setInitialSize(5);
		return ds;
	}

	@Bean(name="sessionFactory")
	public LocalSessionFactoryBean sessionFactory() {
		LocalSessionFactoryBean sfb = new LocalSessionFactoryBean();
		
		sfb.setDataSource(dataSource());
		sfb.setPackagesToScan(new String[] {"edu.demon.DAO", "edu.demon.service"});
		Properties pros = new Properties();
		pros.setProperty("hibernate.dialect", "org.hibernate.dialect.MySQLDialect");
		pros.setProperty("hibernate.show_sql", "true");
		pros.setProperty("hibernate.hbm2ddl.auto", "update");
		pros.setProperty("hibernate.format_sql", "false");
		sfb.setHibernateProperties(pros);
		return sfb;
	}
	
	@Bean(name="transactionManager")
	public HibernateTransactionManager hibernateTransactionManager() {
		HibernateTransactionManager hibernateTransactionManager = new HibernateTransactionManager();
		hibernateTransactionManager.setSessionFactory(sessionFactory().getObject());
		return hibernateTransactionManager;
	}
	
	/**
	 * 捕获任何平台相关的异常，以Spring统一的非检查型异常的形式重新抛出
	 * @return
	 */
	@Bean
	public BeanPostProcessor persisteenceTranslation() {
		return new PersistenceExceptionTranslationPostProcessor();
	}
}
```
```java
@Configuration
@EnableWebMvc	// enable spring mvc
@ComponentScan(basePackages = "edu.demon.web")	// enbale component scan
public class WebConfig extends WebMvcConfigurerAdapter{
	// config jsp view resolver
	@Bean
	public ViewResolver viewResolver() {
		InternalResourceViewResolver resolver = new InternalResourceViewResolver();
		resolver.setPrefix("/WEB-INF/views/");
		resolver.setSuffix(".jsp");
		resolver.setExposeContextBeansAsAttributes(true);	
//		resolver.setViewClass(JstlView.class);
		return resolver;
	}
	// config handle of static resource 
	@Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		configurer.enable();
	}
}
```

4. 在MySQL中创建数据表
```sql
create database SpringDemo
use SpringDemo
create table students(
    id int not null auto_increment primary key,
    name varchar(16) not null,
    sex varchar(4) not null,
    age int unsigned not null,
    telephone varchar(13) not null default "110"
)
```
5. 创建dao包中的类
```java
@Entity
@Table(name="students")
public class Student {
	private Student() {};//声明private的默认构造器，为了不让人使用默认构造器实例化Student
	private Integer id;
	private String name;
	private String sex;
	private Integer age;
	private String telephone;
	public Student(Integer age, String name, String sex, String telephone) {
		this.age = age;
		this.name = name;
		this.sex = sex;
		this.telephone = telephone;
	}
	public Student(Integer id, Integer age, String name, String sex, String telephone) {
		this.id = id;
		this.age = age;
		this.name = name;
		this.sex = sex;
		this.telephone = telephone;
	}
	@Id
	@GeneratedValue(strategy=GenerationType.IDENTITY)
	@Column(name="id")
	public Integer getId() {
		return id;
	}
	public void setId(Integer id) {
		this.id = id;
	}
	@Column(name="name")
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	@Column(name="sex")
	public String getSex() {
		return sex;
	}
	public void setSex(String sex) {
		this.sex = sex;
	}
	@Column(name="age")
	public Integer getAge() {
		return age;
	}
	public void setAge(Integer age) {
		this.age = age;
	}
	@Column(name="telephone")
	public String getTelephone() {
		return telephone;
	}
	public void setTelephone(String telephone) {
		this.telephone = telephone;
	}
}
```
```java
public interface StudentDao {
	  Student findByName(String name);

	  List<Student> findAll();
	  
	  Student findOne(Integer id);

	  Serializable save(Student student);
}
```
```java
public class StudentHibernateDaoIml implements StudentDao {
	@Autowired
	private SessionFactory sessionFactory;
	
	private Session currentSession() {
		return sessionFactory.getCurrentSession();
	}
	
	public Student findByName(String name) {
		return (Student) currentSession().createCriteria(Student.class).add(Restrictions.eq("name", name)).list().get(0);
	}

	public List<Student> findAll() {
		return (List<Student>) currentSession().createCriteria(Student.class).list();
	}

	public Student findOne(Integer id) {
		return (Student)currentSession().get(Student.class, id);
	}

	public Serializable save(Student student) {
		Serializable id = currentSession().save(student);
		return id;
	}
}
```
6. 创建dao包中的类
    - 创建service包中的类
```java
public interface StudentService {
	public void save(Student student);
}
```
```java
@Service
public class StudentHibernateServiceImp implements StudentService{
	@Autowired
	private StudentDao studentDao;	
	@Transactional
	public void save(Student student) {
		studentDao.save(student);
	}
}
```
7. 测试Spring和Hibernate整合
- 在src/test/java中，创建测试类
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes= {RootConfig.class})//, WebConfig.class}) 
public class TestHibernate {
	private ApplicationContext context = null;
	private StudentService service = null;
	{
		context = new AnnotationConfigApplicationContext(RootConfig.class);//, WebConfig.class);
		service = context.getBean(StudentService.class);
	}
	@Test
	public void testSave() {
		Student student = new Student(20, "NewFriend", "male", "16666666666");
		service.save(student);
		System.out.println(student.getName());
	}
}
```