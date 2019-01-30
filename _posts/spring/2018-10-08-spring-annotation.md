---
layout: post
title: "spring注解驱动及其源码分析"
date: 2019-01-12 15:25:32
categories: spring
---

# 一、组件注册

​	Spring注解驱动上下文环境类为`AnnotationConfigApplicationContext`，避免使用`application.xml`进行配置。`AnnotationConfigApplicationContext(Class<?>... annotatedClasses)`传入配置类即可，相比XML配置更加便捷 。

## @Configuration注解

`Configuration`配置类，替代`application.xml`加载配置内容

```java
//配置类，作用相当于配置文件
@Configuration
public class MainConfig {

    @Bean
    public Person person() {
        return new Person("小牛", 19);
    }	
}
```

## @ComponentScan注解

`@ComponentScan`包扫描类，加载指定包下的内容，为可重复注解，如果jdk小于1.7则可使用`@ComponentScans`配置多个`@ComponentScan`

```java
@Configuration
//配置包扫描，去除Controller的加载
@ComponentScan(basePackages="com.kun.componentscan",
    excludeFilters= {@Filter(type=FilterType.ANNOTATION,classes=Controller.class)})
public class MainConfig {

}
```

```java
@Configuration
//配置包扫描，只加载Controlle，注意要设置useDefaultFilters=false
@ComponentScan(basePackages="com.kun.componentscan",useDefaultFilters=false,
    includeFilters= {@Filter(type=FilterType.ANNOTATION,classes=Controller.class)})
public class MainConfig {

}
```

FilterType的种类

- FilterType.ANNOTATION		根据注解类型过滤

- FilterType.ASSIGNABLE_TYPE	更具指定类型加载类及其子类

- FilterType.ASPECTJ			使用AspectJ表达式过滤

- FilterType.REGEX				使用正则表达式过滤

- FilterType.CUSTOM			自定义规则

  ```java
  public class MyTypeFilter implements TypeFilter{
    /**
     * metadataReader：读取当前正在扫描类的信息
     * metadataReaderFactory：可以获取到其他任何类的信息
     */
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        //获得当前扫描类的注解信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
  
        //获取当前扫描类的资源信息（类路径等）
        Resource resource = metadataReader.getResource();
        //获得当前扫描类的类信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
        //获取类名
        String className = classMetadata.getClassName();
        return true;
    }
  }
  ```

## @Scope注解

`@Scope`生命周期配置类

> @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)	//原型，每次从容器获取均创建
> @Scope(ConfigurableBeanFactory.SCOPE_SINGLETON) 	//单实例，每次从容器获取的都是同一个

## @Lazy注解

`@Lazy`懒加载，只对单实例Bean有效。容器启动时并不创建对象，在第一次获取时才创建对象

```java
@Bean
@Lazy
public Person person() {
    return new Person("小牛", 19);
}
```

## @Condition注解

`@Condition`按照条件进行动态创建，可标注在方法和类上。ConditionContext可以得到BeanFactory、getEnvironment、Registry等重要信息。

```java
public class LinuxSystemCondition implements Condition{
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String osName = context.getEnvironment().getProperty("os.name");
        return !osName.contains("Windows");
    }
}

public class WindowsSystemCondition implements Condition{
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String osName = context.getEnvironment().getProperty("os.name");
        return osName.contains("Windows");
    }
}

@Configuration
public class MainConfig {

    @Bean
    @Conditional(WindowsSystemCondition.class)
    public Person personWindows() {
        return new Person("Bill Gates", 60);
    }

    @Bean
    @Conditional(LinuxSystemCondition.class)
    public Person personLinux() {
        return new Person("linus",48);
    }
}
```

## @Import注解

`@Import`容器会自动注册这个组件，id为全类名。`@Improt`的value[]可配置以下类型：

- 普通类，在容器中注册一个普通类型
- ImportSelector的实现类，实现String[] selectImports(AnnotationMetadata importingClassMetadata)，返回需要类的全类名

- ImportBeanDefinitionRegistrar，void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)，使用registry注册类信息

```java
@Configuration
@Import(Red.class)
public class MainConfig {
}

public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar{
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        boolean registerRed = registry.containsBeanDefinition("com.kun.model.Red");
        boolean registerBlue = registry.containsBeanDefinition("com.kun.model.Blue");
        if(registerRed && registerBlue) {
            registry.registerBeanDefinition("yellow", new RootBeanDefinition(Yellow.class));
        }
    }
}

public class MyImportSelector implements ImportSelector{
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.kun.model.Blue","com.kun.model.Pink"};
    }
}
```

## FactoryBean创建实例

```java
public class ColorFactoryBean implements FactoryBean<Color>{
    @Override
    public Color getObject() throws Exception {
        return new Color();
    }
    @Override
    public Class<?> getObjectType() {
        return Color.class;
    }
}

public class MainTest {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(MainConfig.class);
        //拿到ColorFactoryBean,FACTORY_BEAN_PREFIX为前缀‘&’
        Object bean = applicationContext.getBean(BeanFactory.FACTORY_BEAN_PREFIX+"color");
    }
}
```

---

**总结：**

给容器中注册组件的方法

- 包扫描+组件标注注解（@Controller/@Service/@Repository/@Component）[自己写的类]

- @Bean[导入的第三方包里面的组件]

- @Import(要导入到容器中的组件)；容器中就会自动注册这个组件，id默认是全类名

- @Bean配合Spring提供的 FactoryBean（工厂Bean）

  ​	1）、默认获取到的是工厂bean调用getObject创建的对象

  ​	2）、要获取工厂Bean本身，我们需要给id前面加一个&

# 二、生命周期

## @Bean设置初始化和销毁方法

单例对象时：在对象创建之后调用initMethod方法、在容器销毁时调用destroyMethod

非单例对象时：在对象创建之后调用initMethod方法，不会调用destroyMethod

```java
public class Car {
    public Car() {
        System.out.println("Car.Car()");
    }
    public void init() {
        System.out.println("Car.init()");
    }
    public void destory() {
        System.out.println("Car.destory()");
    }
}

@Configuration
public class MainConfig {
    @Bean(initMethod="init",destroyMethod="destory")
    public Car car() {
        return new Car();
    }
}
```

## InitializingBean和DisposableBean

作用和在`@Bean`中设置initMethod、destroyMethod一样

```java
public class Bus implements InitializingBean, DisposableBean {
    @Override
    public void destroy() throws Exception {
        System.out.println("Bus.destroy()");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("Bus.afterPropertiesSet()");
    }
}

@Configuration
public class MainConfig {
    @Bean
    public Bus bus() {
        return new Bus();
    }
}
```

## JSR-250规范实现初始化和销毁方法

`@PostConstruct`、`@PreDestroy`实现设置初始化和销毁

```java
public class Dog {
    @PostConstruct
    public void init() {
        System.out.println("Dog.init()");
    }
    
    @PreDestroy
    public void destory() {
        System.out.println("Dog.destory()");
    }
}

@Configuration
public class MainConfig {	
    @Bean
    public Dog dog() {
        return new Dog();
    }
}
```

## BeanPostProcesser后置处理器控制初始化

```java
@Component
public class MyBeanPostProcesser implements BeanPostProcessor {
    //在Bean创建之后执行init方法之前调用
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(beanName.equals("car")) {
            System.out.println("car before initialization do something");
        }
        return bean;
    }
    //在Bean创建之后执行init方法之后调用
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if(beanName.equals("car")) {
            System.out.println("car after initialization do something");
        }
        return bean;
    }
}
```

# 三、属性赋值

方式一：类比@Value("值")，将值直接注入

方式二：类比@Value("#{20-2}")，使用SpEl计算

方式三：配合`@PropertySource`注解导入properties文件，类比@Value("${employee.name}")方式取值

@Value实现属性

# 四、自动装配

## `@Autowired`自动装配：

- 默认按照类型优先的方式去容器中找对应的组件
- 如果找到多个类型相同的组件，再使用属性名称取匹配（使用`@Qualifier`可以指定查找的对象ID名称）
- 自动装配如果找不到对应的Bean会报错（可以使用@Autowired(required=false)取消必须匹配）
- 再首选Bean上指定`@Primary`使之成为首选对象，如果此时找到多个类型相同的组件，使用首选Bean注入，`@Primary`优先级在`@Qualifier`之下
- 标注在有参构造器上，如果此类只有一个有参构造器会使用自动装配实例化此类

## `@Resource`和`@Inject`自动装配：

@Resource（JSR250）：
 * 可以和@Autowired一样实现自动装配功能，默认是按照组件名称进行装配的（可以使用name属性限定），匹配不到不会报错
 * 不能支持`@Primary`、`@Qualifier`组合

@Inject（JSR330）：

* 需要导入javax.inject的包，和Autowired的功能一样，必须匹配到Bean
* 不能支持`@Primary`、`@Qualifier`组合

## @Bean方法参数：

- 在@Bean标注的方法中，创建对象时入参是通过Spring注入的

```java
//此person由Spring注入
@Bean
public Car car(Person person) {
    Car car = new Car();
    car.setPerson(person);
    return car;
}
```

## 使用Aware接口

- 自定义组件如果要使用Spring底层的一些组件（ApplicationContext，BeanFactory等等）自定义组件可以实现对应的Aware接口，在创建对象时，Spring会传入相关组件。

```txt
Aware
 |
 +- ApplicationContextAware
 |
 +- ApplicationEventPublisherAware
 |
 +- BeanClassLoaderAware
 |
 +- BeanFactoryAware
 |
 +- BeanNameAware
 |
 +- EmbeddedValueResolverAware
 |
 +- EnvironmentAware
 |
 +- ImportAware
 |
 +- LoadTimeWeaverAware
 |
 +- MessageSourceAware
 |
 +- NotificationPublisherAware
 |
 +- ResourceLoaderAware
 |
 +- ServletConfigAware
 |
 +- ServletContextAware
```

## @Profile环境感知

@Profile可以根据当前环境，动态的激活和切换一系列组件的功能

- 默认@Profile会激活值为default的配置
- 在JVM参数添加`-Dspring.profiles.active=值`会激活对应的配置
- 使用java代码也可激活配置

```java
/**
 * Java代码激活
 * AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
 * applicationContext.getEnvironment().setActiveProfiles("Test");
 * applicationContext.register(MainConfig3.class);
 * applicationContext.refresh();
 */
@Configuration
public class MainConfig3 {
    //默认激活
    @Profile("default")
    @Bean(initMethod="init",destroyMethod="close")
    public DataSource dataSourceDev() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUsername("root");
        dataSource.setPassword("123");
        dataSource.setUrl("jdbc:mysql://localhost:3306/dev");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }
    
    @Profile("test")
    @Bean(initMethod="init",destroyMethod="close")
    public DataSource dataSourceTest() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUsername("root");
        dataSource.setPassword("123");
        dataSource.setUrl("jdbc:mysql://localhost:3306/test");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }

    @Profile("produ")
    @Bean(initMethod="init",destroyMethod="close")
    public DataSource dataSourceProdu() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUsername("root");
        dataSource.setPassword("123");
        dataSource.setUrl("jdbc:mysql://localhost:3306/produ");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }
}
```

# 五、AOP

AOP【动态代理】：指在程序运行期间动态的将某段代码切入到指定方法指定位置进行运行的编程方式

## AOP的使用方法

- 导入aop模块【Spring AOP：(spring-aspects)】
- 定义一个业务逻辑类
- 定义一个切面类（并在切面类上加注解@Aspect）
  - 前置通知（@Before）	在目标方法运行之前运行
  - 后置通知（@After）	在目标方法运行结束之后运行（无论方法正常结束还是异常结束）
  - 返回通知（@AfterReturning）	在目标方法正常返回之后运行
  - 异常通知（@AfterThrowing）	在目标方法出现异常以后运行
  - 环绕通知（@Around）		动态代理，手动推进目标方法运行（joinPoint.procced()）
- 给切面类的目标方法标注何时何地运行（通知注解）
- 将切面类和业务逻辑类（目标方法所在类）都加入到容器中
- 给配置类中加 @EnableAspectJAutoProxy

```java
//业务逻辑类
public class MathCalculator {
    public int div(int i, int j) {
        return i / j;
    }
}

//切面类，@Aspect标志自己是配置类
@Aspect
public class LogAspects {

    //定义切面，本类引用直接写方法名，引用外部类的需要加上类全限定名
    @Pointcut("execution(* com.kun.aop.MathCalculator.*(..))")
    public void pointCut() {}
    
    @Before("pointCut()")
    public void logStart(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        System.out.println("方法名"+methodName);
        System.out.println("参数"+Arrays.toString(args));
    }
    
    @After("pointCut()")
    public void logEnd(JoinPoint joinPoint) {
        System.out.println(joinPoint.getSignature().getName()+"调用结束");
    }
    
    @AfterReturning(pointcut="pointCut()",returning="result")
    public void logReturn(JoinPoint joinPoint, Object result) {
        System.out.println(joinPoint.getSignature().getName()+"正常返回，运行结果："+result);
    }

    @AfterThrowing(pointcut="pointCut()",throwing="exception")
    public void logException(JoinPoint joinPoint,Exception exception){
        System.out.println(joinPoint.getSignature().getName()+"异常，异常信息："+exception);
    }
}

//配置类
@Configuration
@EnableAspectJAutoProxy		//开启AOP自动配置
public class MainConfig4 {
    //业务逻辑类加入容器中
    @Bean
    public MathCalculator calculator(){
        return new MathCalculator();
    }
    //切面类加入到容器中
    @Bean
    public LogAspects logAspects(){
        return new LogAspects();
    }
}
```

## AOP源码分析

**1、EnableAspectJAutoProxy实际上导入了AspectJAutoProxyRegistrar.class**

```java
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
    boolean exposeProxy() default false;
}
```

**2、AspectJAutoProxyRegistrar是ImportBeanDefinitionRegistrar的子类，实际注册了ID为internalAutoProxyCreator的AnnotationAwareAspectJAutoProxyCreator的类信息**

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
//注册了ID为org.springframework.aop.config.internalAutoProxyCreator的AnnotationAwareAspectJAutoProxyCreator.class组件信息
         AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        //修改AnnotationAwareAspectJAutoProxyCreator的信息
        AnnotationAttributes enableAspectJAutoProxy =

        AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy != null) {
            if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }
            if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
            }
        }
    }
}
```

**3、AnnotationAwareAspectJAutoProxyCreator创建以及被代理类创建流程分析、**

1. 传入配置类，创建ioc容器

2. 注册配置类，调用refresh()刷新容器

3. registerBeanPostProcessors(beanFactory)，注册bean的后置处理器来方便拦截bean的创建

   1. 先获取ioc容器已经定义了的需要创建对象的所有BeanPostProcessor
   2. 给容器中加别的BeanPostProcessor
   3. 优先注册实现了PriorityOrdered接口的BeanPostProcessor
   4. 再给容器中注册实现了Ordered接口的BeanPostProcessor
      1. 创建Bean的实例
      2. populateBean：给bean的各种属性赋值
      3. initializeBean：初始化bean
         1. invokeAwareMethods()：处理Aware接口的方法回调
         2. applyBeanPostProcessorsBeforeInitialization()：应用后置处理器的postProcessBeforeInitialization()
         3. invokeInitMethods()：执行自定义的初始化方法
         4. applyBeanPostProcessorsAfterInitialization()：执行后置处理器的postProcessAfterInitialization()
      4. BeanPostProcessor(AnnotationAwareAspectJAutoProxyCreator)创建成功-->	aspectJAdvisorsBuilde
      5. 注册没实现优先级接口的BeanPostProcessor
      6. 把BeanPostProcessor注册到BeanFactory中

4. finishBeanFactoryInitialization(beanFactory)完成BeanFactory初始化工作，创建剩下的单实例bean，与AOP相关部分

   遍历获取容器中所有的Bean，依次创建对象，在创建Bean之前调用applyBeanPostProcessorsBeforeInstantiation()

```java
//识别保存@Aspects注解类，保存相关注解信息和切面、切入点表达式
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    Object cacheKey = getCacheKey(beanClass, beanName);
    if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }

    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }
    return null;
}
```

​	2）运用applyBeanPostProcessorsAfterInitialization()，将对象进行动态代理

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (!this.earlyProxyReferences.contains(cacheKey)) {
            //包装生成动态代理类，
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
/**
wrapIfNecessary(bean, beanName, cacheKey);
    1、获取当前bean的所有增强器（通知方法）  Object[]  specificInterceptors
        1、找到候选的所有的增强器（找哪些通知方法是需要切入当前bean方法的）
        2、获取到能在bean使用的增强器
        3、给增强器排序
    2、保存当前bean在advisedBeans中
    3、如果当前bean需要增强，创建当前bean的代理对象
        1、获取所有增强器
        2、保存到proxyFactory
        3、创建代理对象：Spring自动决定
**/
```

**4、目标方法的执行**

1. CglibAopProxy.intercept();拦截目标方法的执行

2. 根据ProxyFactory对象获取将要执行的目标方法拦截器链

   ```java
   //获取目标方法的过滤器链
   List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
   Object retVal;
   //如果链为空，直接执行目标方法
   if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
       Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
       retVal = methodProxy.invoke(target, argsToUse);
   }
   else {
       retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
   }
   ```

3. 如果有拦截器链，把需要执行的目标对象，目标方法，拦截器链等信息传入创建一个 CglibMethodInvocation 对象，并调用 Object retVal =  mi.proceed()

4. 拦截器链的触发过程

   1. 如果没有拦截器执行执行目标方法，或者拦截器的索引和拦截器数组-1大小一样（指定到了最后一个拦截器）执行目标方法
   2. 链式获取每一个拦截器，拦截器执行invoke方法，每一个拦截器等待下一个拦截器执行完成返回以后再来执行；拦截器链的机制，保证通知方法与目标方法的执行顺序

# 六、声明式事务

## 声明式事务使用方式

- 配置数据源和事务管理器
- @EnableTransactionManagement 开启基于注解的事务管理功能
- 在service上需要事务的方法上注明@Transactional 表示当前方法是一个事务方法

```java
@EnableTransactionManagement
@ComponentScan("com.kun.transcation")
@Configuration
public class MainConfig {

    //配置事务管理器
    @Bean
    public DataSourceTransactionManager transactionManagement() {
        return new DataSourceTransactionManager(dataSource());
    }

    //配置数据源
    @Bean(initMethod="init",destroyMethod="close")
    public DataSource dataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setUrl("jdbc:mysql://localhost:3306/mapper");
        dataSource.setUsername("root");
        dataSource.setPassword("123456");
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        return dataSource;
    }

    //配置JdbcTemplate
    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());
    }
}

@Repository
public class TestDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    public void insert() {
        jdbcTemplate.update("insert into person(name) values(?)","kun");
    }
}

@Service
public class TestService {

    @Autowired
    private TestDao testDao;

    //声明事务
    @Transactional
    public void insert() {
        testDao.insert();
        int i = 1/0;
    }
}
```

## 声明式事务源码分析

- `@EnableTransactionManagement`注解导入了`TransactionManagementConfigurationSelector.class`

```java
//TransactionManagementConfigurationSelector实现了ImportSelector接口
//使用默认配置会导入AutoProxyRegistrar和ProxyTransactionManagementConfiguration
protected String[] selectImports(AdviceMode adviceMode) {
    switch (adviceMode) {
        case PROXY:
            return new String[] {AutoProxyRegistrar.class.getName(),
                                 ProxyTransactionManagementConfiguration.class.getName()};
        case ASPECTJ:
            return new String[] {determineTransactionAspectClass()};
        default:
            return null;
    }
}
```

- `AutoProxyRegistrar`实现了`ImportBeanDefinitionRegistrar`接口导入了ID为`internalAutoProxyCreator`的`InfrastructureAdvisorAutoProxyCreator`类实现了AOP功能【使用postProcessAfterInitialization()包装类实现动态代理】

- `ProxyTransactionManagementConfiguration`

```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {
    //实现切入点
    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
        BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
        advisor.setTransactionAttributeSource(transactionAttributeSource());
        advisor.setAdvice(transactionInterceptor());
        if (this.enableTx != null) {
            advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
        }
        return advisor;
    }
    
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource();
    }

    //拦截器链
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionInterceptor transactionInterceptor() {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionAttributeSource(transactionAttributeSource());
        if (this.txManager != null) {
            interceptor.setTransactionManager(this.txManager);
        }
        return interceptor;
    }
}
```

# 七、扩展原理

## Bean后置处理器

**BeanPostProcessor**接口在容器内对象创建完成之后属性赋值前后调用

```java
public interface BeanPostProcessor {

    //对象创建完成，在afterPropertiesSet或自定义init方法执行之前
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    //对象创建完成，在afterPropertiesSet或自定义init方法执行之后
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

}
```

**InstantiationAwareBeanPostProcessor**接口在容器内对象创建前后调用

```java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

    //对象实例化之前调用
    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }

    //对象实例化之后调用
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }

    //对象实例化之后属性赋值之前，@Autowired、@Resource等就是根据这个回调来实现最终注入依赖的
    @Nullable
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
            throws BeansException {
        return null;
    }
}
```

**DestructionAwareBeanPostProcessor**接口在对象销毁之前调用

```java
public interface DestructionAwareBeanPostProcessor extends BeanPostProcessor {

    //对象销毁之前调用
    void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;

    //判断是否需要处理这个对象的销毁
    default boolean requiresDestruction(Object bean) {
        return true;
    }
}
```

**MergedBeanDefinitionPostProcessor**混合Bean定义之后的处理器

```java
public interface MergedBeanDefinitionPostProcessor extends BeanPostProcessor {

    //合并bean定义后进行的处理
    void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);

    //通知指定名称的bean定义已被重置，这个后处理器应该清除受影响bean的任何元数据
    default void resetBeanDefinition(String beanName) {}
}
```

**执行时序**

1. InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation(beanClass, beanName)在创建对象之前调用，如果有返回实例则，不会去走下面创建对象的逻辑但会调用postProcessAfterInitialization()
2. BeanPostProcessor.postProcessAfterInitialization(result, beanName)对象创建之后调用
3. SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors(beanClass, beanName)如果需要的话，会在实例化对象之前执行
4. MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition(mbd, beanType, beanName)在对象实例化完毕 初始化之前执行
5. InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)在bean创建完毕初始化之前执行
6. InstantiationAwareBeanPostProcessor.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName)在bean的property属性注入完毕向bean中设置属性之前执行
7. BeanPostProcessor.postProcessBeforeInitialization(result, beanName)在bean初始化（自定义init或者是实现了InitializingBean.afterPropertiesSet())之前执行
8. BeanPostProcessor.postProcessAfterInitialization(result, beanName)在bean初始化（自定义init或者是实现了InitializingBean.afterPropertiesSet())之后执行
9. 其中DestructionAwareBeanPostProcessor方法的postProcessBeforeDestruction(Object bean, String beanName)会在销毁对象前执行

## BeanFactory后置处理器

**BeanFactoryPostProcessor**在BeanFactory标准初始化之后调用，来定制和修改BeanFactory的内容,此时所有的bean定义已经保存加载到beanFactory，但是bean的实例还未创建

```java
public interface BeanFactoryPostProcessor {
    //已有Bean的定义
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

**BeanDefinitionRegistryPostProcessor**，在`BeanFactoryPostProcessor`之后调用，用于注册Bean信息

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
   
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

## 监听器

**监听器的使用**

方式一：实现ApplicationListener接口

```java
//Event的类型有
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    void onApplicationEvent(E event);
}
```

方式二：使用`@EventListener`注解

```java
@Service
public class EventService {

    @EventListener
    public void playEvent(ApplicationEvent event) {
        System.out.println(event);
    }
}
```

# 八、源码分析

Spring容器的refresh()【创建刷新过程】

1. prepareRefresh()【刷新前的预处理】

   1. initPropertySources()【初始化一些属性设置，交给子类自定义个性化的属性设置方法】
   2. getEnvironment().validateRequiredProperties()【检验属性的合法等】
   3. this.earlyApplicationEvents = new LinkedHashSet<>()【保存容器中的一些早期的事件的集合】

2. ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()【获得BeanFactory】

   1. refreshBeanFactory()【创建了一个this.beanFactory = new DefaultListableBeanFactory()并设置id】
   2. getBeanFactory()【返回刚才GenericApplicationContext创建的BeanFactory对象】
   3. 将创建的BeanFactory【DefaultListableBeanFactory】返回

3. prepareBeanFactory(beanFactory)【BeanFactory的预准备工作】

   1. 设置BeanFactory的类加载器、支持表达式解析器等
   2. 添加部分BeanPostProcessor【ApplicationContextAwareProcessor和ApplicationListenerDetector】
   3. 设置忽略的自动装配的接口EnvironmentAware、EmbeddedValueResolverAware、ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware和ApplicationContextAware，即这些接口实现类无法通过Autowired方式注入
   4. 注册可以解析的自动装配，我们能直接在任何组件中自动注入【BeanFactory、ResourceLoader、ApplicationEventPublisher和ApplicationContext】
   5. 添加编译时的AspectJ相关组件
   6. 给BeanFactory中注册一些能用的组件【environment、systemProperties和systemEnvironment】

4. postProcessBeanFactory(beanFactory)【子类通过重写这个方法来在BeanFactory创建并预准备完成以后做进一步的设置】

5. invokeBeanFactoryPostProcessors(beanFactory)【调用BeanFactoryPostProcessor的方法】

   1. 执行BeanDefinitionRegistryPostProcessor的实现类
      1. 获取所有的BeanDefinitionRegistryPostProcessor
      2. 先执行实现了PriorityOrdered优先级接口的BeanDefinitionRegistryPostProcessor
         调用postProcessor.postProcessBeanDefinitionRegistry(registry)
      3. 再执行实现了Ordered顺序接口的BeanDefinitionRegistryPostProcessor
         调用postProcessor.postProcessBeanDefinitionRegistry(registry)
      4. 最后执行没有实现任何优先级或者是顺序接口的BeanDefinitionRegistryPostProcessor
         调用postProcessor.postProcessBeanDefinitionRegistry(registry)
   2. 执行BeanFactoryPostProcessor的实现类
      1. 获取所有的BeanFactoryPostProcessor
      2. 先执行实现了PriorityOrdered优先级接口的BeanFactoryPostProcessor
         调用postProcessor.postProcessBeanFactory(ConfigurableListableBeanFactory)
      3. 再执行实现了Ordered顺序接口的BeanFactoryPostProcessor
         调用postProcessor.postProcessBeanFactory(ConfigurableListableBeanFactory)
      4. 最后执行没有实现任何优先级或者是顺序接口的BeanFactoryPostProcessor
         调用postProcessor.postProcessBeanFactory(ConfigurableListableBeanFactory)

6. registerBeanPostProcessors(beanFactory)【注册BeanPostProcessor并不执行】

   1. 获取所有的 BeanPostProcessor

   2. 先注册PriorityOrdered优先级接口的BeanPostProcessor，把每一个BeanPostProcessor添加到BeanFactory

      sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
      registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

   3. 再注册Ordered顺序接口的BeanPostProcessor，把每一个BeanPostProcessor添加到BeanFactory

      sortPostProcessors(orderedPostProcessors, beanFactory);
      registerBeanPostProcessors(beanFactory, orderedPostProcessors);

   4. 然后注册没有实现任何优先级或者是顺序接口的BeanPostProcessorr

      registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

   5. 最终注册MergedBeanDefinitionPostProcessor

      sortPostProcessors(internalPostProcessors, beanFactory);
      registerBeanPostProcessors(beanFactory, internalPostProcessors);

   6. 注册一个ApplicationListenerDetector【用于检查在Bean创建完成后检查是否是ApplicationListener】

7. initMessageSource()【初始化MessageSource组件（做国际化功能；消息绑定，消息解析）】

   1. 获取BeanFactory
   2. 容器中是否有id为messageSource的，类型是MessageSource的组件如果有赋值给messageSource，如果没有自己创建一个DelegatingMessageSource
   3. 把创建好的MessageSource注册在容器中，以后获取国际化配置文件的值的时候，可以自动注入MessageSource

8. initApplicationEventMulticaster()

   1. 获取BeanFactory
   2. 从BeanFactory中获取applicationEventMulticaster的ApplicationEventMulticaster
   3. 如果上一步没有配置；创建一个SimpleApplicationEventMulticaster
   4. 将创建的ApplicationEventMulticaster添加到BeanFactory中，以后其他组件直接自动注入

9. onRefresh()【子类重写这个方法，在容器刷新的时候可以自定义逻辑】

10. registerListeners()

    1. 从容器中拿到所有的ApplicationListener
    2. 将每个监听器添加到事件派发器中
    3. 派发之前步骤产生的事件【this.earlyApplicationEvents保存的事件】

11. finishBeanFactoryInitialization(beanFactory)【初始化所有剩下的单实例bean，beanProcessor等的早已实例化】

    beanFactory.preInstantiateSingletons()【获取容器中的所有Bean，依次进行初始化和创建对象】

    1. 取Bean的定义信息【RootBeanDefinition】

    2. 如果Bean不是抽象的，是单实例的，是懒加载

       1. 判断是否是FactoryBean通过FactoryBean的规则获得bean

       2. 不是工厂Bean。利用getBean(beanName)创建对象

          1. 先获取缓存中保存的单实例Bean。如果能获取到说明这个Bean之前被创建过，返回

          2. 缓存中获取不到，开始Bean的创建对象流程

          3. 标记当前bean已经被创建【防止并发环境产生错误】

          4. 获取Bean的定义信息

          5. 获取当前Bean依赖的其他Bean，如果有使用getBean(beanName)递归调用把依赖的Bean先创建出来

          6. 启动单实例Bean的创建流程

             1. resolveBeforeInstantiation(beanName, mbdToUse)让BeanPostProcessor先拦截返回代理信息对象【InstantiationAwareBeanPostProcessor提前执行】
                先触发：postProcessBeforeInstantiation()；
                如果有返回值：触发postProcessAfterInitialization()

             2. 如果前面的InstantiationAwareBeanPostProcessor没有返回代理对象调用下面步骤

             3. Object beanInstance = doCreateBean(beanName, mbdToUse, args)【创建Bean】

                1. Object beanInstance = doCreateBean(beanName, mbdToUse, args)【利用工厂方法或者对象的构造器创建出Bean实例】

                2. pplyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName)【调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition(mbd, beanType, beanName)】

                3. populateBean(beanName, mbd, instanceWrapper)【Bean属性赋值】

                   1. 拿到InstantiationAwareBeanPostProcessor后置处理器调用

                      postProcessAfterInstantiation()

                      postProcessPropertyValues()

                   2. 应用Bean属性的值applyPropertyValues(beanName, mbd, bw, pvs)

                4. initializeBean(beanName, exposedObject, mbd)【Bean初始化】

                   1. invokeAwareMethods(beanName, bean)【执行Aware接口方法】
                   2. applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName)【init方法执行之前调用】
                   3. invokeInitMethods(beanName, wrappedBean, mbd)【执行初始化方法】
                   4. applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName)【执行后置处理器初始化之后】

                5. 注册Bean的销毁方法

12. finishRefresh()

    1. initLifecycleProcessor()初始化和生命周期有关的后置处理器【默认从容器中找是否有LifecycleProcessor的组件，如果没有new DefaultLifecycleProcessor()加入到容器用于处理生命周期回调】
    2. getLifecycleProcessor().onRefresh()【拿到前面定义的生命周期处理器回调onRefresh()】
    3. publishEvent(new ContextRefreshedEvent(this));发布容器刷新完成事件

# 九、Spring MVC纯注解

## Servlet3.0相关

前提：Servlet3.0需要Tomcat7以上版本

- 实现`ServletContainerInitializer`接口完成容器启动配置功能，相当于web.xml

```java
/**
* 实现ServletContainerInitializer的接口需要在src目录下创建
* META-INF/services/javax.servlet.ServletContainerInitializer文件
* 文件内容是实现其接口的实现类
*/
@HandlesTypes(value= {IService.class})
public class MyServletContainerInitializer implements ServletContainerInitializer{

    //Set<Class<?>> c传入的是@HandlesTypes配置的类及其子类
    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
        //ServletContext可以添加Servlet、Fileter、Listener、initParam等
    }
}
```

- 异步请求

  ![servlet3.0-before-request](/img/spring/servlet3.0-before-request.png)

  - 在Servlet3.0之前，Servlet采用Thread-Pre-Request的方式处理请求，即每一次Http请求都是由某一个线程从头到尾负责处理。如果一个请求需要进行IO操作，那么其所对应的线程将同步地等待IO操作完成，此时线程并不能及时的释放回线程池以供后续使用，在并发量越来越大的情况下这将带来严重的性能问题。

  ![servlet-asnyc](/img/spring/servlet-asnyc.png)

  - Servlet3.0之后但是了异步处理机制，目的是解决大量低延迟、少量高延迟大并发情况下带来的性能瓶颈


```java
@WebServlet(value="/async",asyncSupported=true)
public class HelloAsyncServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        //1、支持异步处理asyncSupported=true
        //2、开启异步模式
        System.out.println("主线程开始。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
        AsyncContext startAsync = req.startAsync();

        //3、业务逻辑进行异步处理;开始异步处理
        startAsync.start(new Runnable() {
            @Override
            public void run() {
                try {
                    System.out.println("副线程开始。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
                    sayHello();
                    //获取到异步上下文
                    AsyncContext asyncContext = req.getAsyncContext();
                    //4、获取响应
                    ServletResponse response = asyncContext.getResponse();
                    response.getWriter().write("hello async...");
                    System.out.println("副线程结束。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
                } catch (Exception e) {
                }
            }
        });		
        System.out.println("主线程结束。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
    }

    public void sayHello() throws Exception{
        System.out.println(Thread.currentThread()+" processing...");
        Thread.sleep(3000);
    }
}
```

## Spring MVC纯注解

**整合原理**

- 根据Servlet3.0规范web容器在启动的时候，会扫描每个jar包下的META-INF/services/javax.servlet.ServletContainerInitializer文件，而且Spring Web包下的此文件内容为org.springframework.web.SpringServletContainerInitializer。
- SpringServletContainerInitializer会加载WebApplicationInitializer子类及其实现类

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

    @Override
    public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext) throws ServletException {

        List<WebApplicationInitializer> initializers = new LinkedList<>();

        if (webAppInitializerClasses != null) {
            for (Class<?> waiClass : webAppInitializerClasses) {
                //如果不是接口和抽象类会被实例化
                if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
                    try {
                        initializers.add((WebApplicationInitializer)ReflectionUtils.accessibleConstructor(waiClass).newInstance());
                    }
                    catch (Throwable ex) {
                        throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
                    }
                }
            }
        }

        if (initializers.isEmpty()) {
            servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
            return;
        }

        servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
        AnnotationAwareOrderComparator.sort(initializers);
        for (WebApplicationInitializer initializer : initializers) {
            initializer.onStartup(servletContext);
        }
    }
}
```

- WebApplicationInitializer子类有

```txt
WebApplicationInitializer
 |
 +- AbstractReactiveWebInitializer
 |
 +- AbstractContextLoaderInitializer【Spring父容器初始化器】
     |
     +- AbstractDispatcherServletInitializer【创建了dispatcherServlet】
         +
         |
         + AbstractAnnotationConfigDispatcherServletInitializer【注解初始化器】
```

- 整合方法：以注解方式来启动SpringMVC，继承AbstractAnnotationConfigDispatcherServletInitializer
  实现抽象方法指定DispatcherServlet的配置信息

**实现示例**

```java
public class MyWebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer{

    //获取根容器的配置类
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] {RootConfig.class};
    }

    //获取web容器的配置类
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] {ServletConfig.class};
    }

    /**
     *  获取DispatcherServlet的映射信息
     * /：拦截所有请求（包括静态资源（xx.js,xx.png）），但是不包括*.jsp； 
     * /*：拦截所有请求；连*.jsp页面都拦截；jsp页面是tomcat的jsp引擎解析的；
     */
    @Override
    protected String[] getServletMappings() {
        return new String[] {"/"};
    }
    
    //配置过滤器
    Override
    protected Filter[] getServletFilters() {
           return new Filter[] {
                   new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}

@Configuration
@ComponentScan(basePackages= {"com.kun"},excludeFilters= {@Filter(type=FilterType.ANNOTATION,classes={Controller.class})})
public class RootConfig {

}

@Configuration
@ComponentScan(basePackages= {"com.kun"},useDefaultFilters=false,
includeFilters= {@Filter(type=FilterType.ANNOTATION,classes=Controller.class)})
public class ServletConfig {

}
```

**定制SpringMVC功能**

```java
@Configuration
@ComponentScan(basePackages= {"com.kun"},useDefaultFilters=false,
includeFilters= {@Filter(type=FilterType.ANNOTATION,classes=Controller.class)})
@EnableWebMvc	//定制spring mvc配置
public class ServletConfig implements WebMvcConfigurer{

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/jsp", ".jsp");
    }
}
```

**异步请求**

- 控制器返回Callable
- Spring异步处理，将Callable 提交到 TaskExecutor 使用一个隔离的线程进行执行
- DispatcherServlet和所有的Filter退出web容器的线程，但是response 保持打开状态
- Callable返回结果，SpringMVC将请求重新派发给容器，恢复之前的处理
- 根据Callable返回的结果。SpringMVC继续进行视图渲染流程等（从收请求-视图渲染）

```java
@Controller
public class AsyncController {
    @ResponseBody
    @RequestMapping("/async")
    public Callable<String> async(){
        System.out.println("主线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());

        Callable<String> callable = new Callable<String>() {
            @Override
            public String call() throws Exception {
                System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
                Thread.sleep(2000);
                System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
                return "Callable<String> async()";
            }
        };

        System.out.println("主线程结束..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
        return callable;
    }
}
```

运行结果

```txt
preHandle.../springmvc-annotation/async
主线程开始...Thread[http-bio-8081-exec-3,5,main]==>1513932494700
主线程结束...Thread[http-bio-8081-exec-3,5,main]==>1513932494700
副线程开始...Thread[MvcAsync1,5,main]==>1513932494707
副线程开始...Thread[MvcAsync1,5,main]==>1513932496708
preHandle.../springmvc-annotation/async
postHandle...
afterCompletion...
```

运行结果解释

拦截请求->主线程执行->主线程执行结束->DispatcherServlet和所有的Filter退出web容器的线程->子线程开始->子线程结束->重新模拟请求->前置拦截器->返回子线程执行完成结果->后置拦截器->视图处理完成拦截器

注意：Spring MVC异步处理还可以使用返回DeferredResult进行实现，拦截异步请求处理过程可以使用AsyncHandlerInterceptor进行拦截
