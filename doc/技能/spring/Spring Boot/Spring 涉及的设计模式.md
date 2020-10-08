# 一、简单工厂
## 1. 实现方式
        BeanFactory，Spring中的BeanFactory就是简单工厂模式的体现，根据传入一个唯一的标识来获得Bean对象，但是否是在传入参数后创建还是传入参数前创建这个要根据具体情况来定。
## 2. 实质
        一个工厂类根据传入的参数，动态决定应该创建哪一个产品类。
## 3. 实现原理
        bean容器的启动阶段：
  - 读取bean的xml配置文件，将bean元素分别转换成一个BeanDefinition对象；
  - 然后通过BeanDefinitionRegistry将这些bean注册到beanFactory中，保存在它的一个ConcurrentHashMap中；  
        容器中bean的实例化阶段：  
        实例化阶段主要是通过反射或者cglib进行实例化，在这个阶段Spring给我们暴露了很多的扩展点：
  - 各种的Aware接口，比如BeanFactoryAware，对于实现了这些Aware接口的bean，在实例化bean时Spring会帮我们注入对应的BeanFactory的实例。
  - BeanPostProcessor接口，实现了BeanPostProcessor接口的bean，在实例化bean后Spring会帮我们调用接口中的方法；
  - InitializingBean接口，实现了InitializingBean接口的bean，在实例化bean时Spring会帮我们调用接口中的方法；
  - DisposableBean接口，实现了DisposableBean接口的bean，在该bean死亡时Spring会帮我们调用接口中的方法；
## 4. 设计意义
  - 松耦合  
        可以将原先硬编码的依赖，通过Spring的beanFactory这个工厂来注入依赖，也就是说原来只有依赖方和被依赖方，现在我们引入了第三方，也就是BeanFactory，右它来解决bean之间的依赖问题，达到了松耦合的效果。
  - bean的额外处理  
        通过Spring接口的暴露，在实例化bean的截断我们可以进行一些额外的处理，这些额外的处理只需要让bean实现对应的接口即可，那么spring就会在bean的生命周期调用我们实现的接口来处理该bean。

# 二、工厂方法
## 1. 实现方式  
        FactoryBean接口；
## 2. 实现原理
        实现了FactoryBean接口的Bean是一类叫做Factory的bean，其特点是：Spring会在使用getBean()调用获得该bean时，会自动调用该bean的getObject()方法，所以返回的不是factory这个bean，而是这个bean.getObject()方法的返回值。

# 三、单例模式
        Spring依赖注入bean实例默认是单例的。
        Spring的依赖注入都是发生在AbstractBeanFactory的getBean()里。getBean()的doGetBean()方法调用getSingleton()进行bean的创建。
```
public Object getSingleton(String beanName){
    //参数true设置标识允许早期依赖
    return getSingleton(beanName,true);
}
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //检查缓存中是否存在实例
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        //如果为空，则锁定全局变量并进行处理。
        synchronized (this.singletonObjects) {
            //如果此bean正在加载，则不处理
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                //当某些方法需要提前初始化的时候则会调用addSingleFactory 方法将对应的ObjectFactory初始化策略存储在singletonFactories
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    //调用预先设定的getObject方法
                    singletonObject = singletonFactory.getObject();
                    //记录在缓存中，earlysingletonObjects和singletonFactories互斥
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```
![blockchain](/resource/images/spring%20创建实例过程.jpg)  


# 四、适配器模式
## 1. 实现方式
        SpringMVC中的适配器HandlerAdapter；
## 2. 实现原理
        HandlerAdapter根据Handler规则执行不同的Handler；
## 3. 实现过程
        DispatcherServlet根据HandlerMapping返回的Handler，向HandlerAdapter发起请求，处理Handler；
        HandlerAdapter根据规则找到对应的Handler并让其执行，执行完毕后Handler会向HandlerAdapter返回一个ModelAndView，最后由HandlerAdapter向DispatcherServlet返回一个ModelAndView。
## 4. 实现意义
        HandlerAdapter使得Handler的扩展变得容易，只需要增加一个新的Handler和一个对应的HandlerAdapter即可；
        因此Spring定义了一个适配接口，使得每一种Controller有一种对应的适配器实现类，让适配器代替Controller执行相应的方法，这样在扩展Controller时，只需要增加一个适配器类就完成SpringMVC的扩展了。

# 五、装饰器模式
## 1. 实现方式
        Spring中用到的包装器模式在类名上有两种体现，一种是类名中含有Wrapper，另一种类名中含有Decorator。
## 2. 实质
        动态的给一个对象添加一些额外的职责；

# 六、代理模式
        代理分动态代理和静态代理；
        动态代理在内存中直接构建，不需要手动编写代理类；
        静态代理需要手动编写代理类，代理类引用被代理对象；
        AOP底层，就是动态代理模式的实现；切面在应用运行的时刻被植入，一般情况下，在植入切面时，AOP容器会为目标对象动态的创建一个代理对象。
# 七、观察者模式
## 1. 实现方式
        Spring的事件驱动模型使用的是观察者模式，Spring中Observer模式常用的地方是Listener的实现；
## 2. 具体实现
        事件机制的实现需要三个部分：事件源、事件、事件监听器；
        ApplicationEvent抽象类（事件）继承自jdk的EventObject，所有的事件都需要继承ApplicationEvent，并且通过构造器参数source得到事件源。
```
public abstract class ApplicationEvent extends EventObject {
    privatestaticfinallong serialVersionUID = 7099057708183571937L;
    privatefinallong timestamp;
    public ApplicationEvent(Object source) {
    super(source);
    this.timestamp = System.currentTimeMillis();
    }
    public final long getTimestamp() {
        returnthis.timestamp;
    }
}
```
        该类的实现类ApplicationContextEvent表示ApplicationContext的容器事件。

        ApplicationListener接口（监听器）继承自jdk的EventListener，所有的监听器都要实现这个接口。这个接口只有一个onApplicationEvent()方法，该方法接受一个ApplicationEvent或其子类对象作为参数，在方法体重，可以不同不过对Event的判断来进行相应的处理。

        ApplicationContext接口（事件源）是Spring中的全局容器，翻译过来就是“应用上下文”，实现了ApplicationEventPublisher接口，负责读取bean的配置文档、管理bean的加载、维护bean之间的依赖关系，可以说是负责bean的整个生命周期，再通俗一点就是我们平时所说的IOC容器。
```
public interface ApplicationEventPublisher {
        void publishEvent(ApplicationEvent event);
}

public void publishEvent(ApplicationEvent event) {
    Assert.notNull(event, "Event must not be null");
    if (logger.isTraceEnabled()) {
         logger.trace("Publishing event in " + getDisplayName() + ": " + event);
    }
    getApplicationEventMulticaster().multicastEvent(event);
    if (this.parent != null) {
    this.parent.publishEvent(event);
    }
}
```
# 八、策略模式
## 1. 实现方式
        Spring框架的资源访问Resource接口，该接口提供了更强的资源访问能力，Spring框架本身大量使用了Resource接口来访问底层资源。
        Resource接口是具体资源访问策略的抽象，也是所有资源访问类实现的接口，实现了getInputStream()、exists()等接口；
        Resource接口本身没有提供访问任何底层资源的显示逻辑，针对不同的底层资源，Spring将会提供不同的Resource实现类，不同的实现类负责不同的资源访问逻辑，如URLResource、ClassPathResource、FileSystemResource、InputStreamResource、ByteArrayResource等。
# 九、模版方法模式
        经典模版方法定义：父类定义了股价（调用哪些方法及顺序），某些特定方法由子类实现；
        最大的好处：代码复用，减少重复代码，除了子类要实现的特定方法，其他方法及方法调用顺序都在父类中预先写好了。
        Spring模版方法模式实质是模版方法模式和回调模式的结合，是Template Method不需要继承的另一种实现方式。
        JDBC的抽象和对Hibernate的基础，都采用了这种处理方式。
```
public abstract class JdbcTemplate {
     publicfinal Object execute（String sql）{
        Connection con=null;
        Statement stmt=null;
        try{
            con=getConnection（）;
            stmt=con.createStatement（）;
            Object retValue=executeWithStatement（stmt,sql）;
            return retValue;
        }catch（SQLException e）{
             ...
        }finally{
            closeStatement（stmt）;
            releaseConnection（con）;
        }
    }
    protectedabstract Object executeWithStatement（Statement   stmt, String sql）;
}
```
        采用模版方法模式是为了以一种统一而集中的方式来处理资源的获取和释放。
        采用回调方式的是因为JdbcTemplate是抽象类，不能独立使用，我们每次进行数据访问时都要给出一个相应的子类实现，这样不方便，所以就引入了回调。
```
publicinterface StatementCallback{
    Object doWithStatement（Statement stmt）;
}
```
```
publicclass JdbcTemplate {
    publicfinal Object execute（StatementCallback callback）{
        Connection con=null;
        Statement stmt=null;
        try{
            con=getConnection（）;
            stmt=con.createStatement（）;
            Object retValue=callback.doWithStatement（stmt）;
            return retValue;
        }catch（SQLException e）{
            ...
        }finally{
            closeStatement（stmt）;
            releaseConnection（con）;
        }
    }

    ...//其它方法定义
}
```
        Jdbc使用方法如下：
```
    JdbcTemplate jdbcTemplate=...;
    final String sql=...;
    StatementCallback callback = new StatementCallback() {
        public Object = doWithStatement(Statement stmt) {
        return ...;
    }
}
jdbcTemplate.execute(callback);
```
        因为JDBCTemplate这个类的方法太多，使用继承太麻烦。所以我们就把变化的东西抽出来作为一个参数传入JdbcTemplate的方法中，我们把变化的东西抽象成一个回调对象，在这个回调对象中定义一个操纵JdbcTemplate中变量的方法，我们去实现这个方法，就把变化的东西集中在这里了。