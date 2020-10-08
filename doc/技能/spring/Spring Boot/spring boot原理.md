# 一、Spring Boot介绍
        Spring Boot实现了auto-configuration自动配置（另外三大神器：actuator监控、cli命令行接口、starter依赖），降低了项目构建的复杂度。它主要是为了解决使用Spring框架需要大量的配置太麻烦的问题。

# 二、Spring Boot启动类入库
```
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

# 三、@SpringBootApplication注解
```
@Target(ElementType.TYPE) // 注解的适用范围，其中TYPE用于描述类、接口（包括包注解类型）或enum声明
@Retention(RetentionPolicy.RUNTIME) // 注解的生命周期，保留到class文件中（三个生命周期）
@Documented // 表明这个注解应该被javadoc记录
@Inherited // 子类可以继承该注解
@SpringBootConfiguration // 继承了Configuration，表示当前是注解类
@EnableAutoConfiguration // 开启springboot的注解功能，springboot的四大神器之一，其借助@import的帮助
@ComponentScan(excludeFilters = { // 扫描路径设置（具体使用待确认）
@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
   ...
}　　
```
## 1. @SpringBootConfiguration  
        标注当前类是Java Config配置类，会被扫描并加载到IOC容器；
## 2. @ComponentScan
        扫描默认包或指定包下面的符合条件的组件或者bean定义并加载到Ioc容器中，如@Component、@Controller等；
        我们可以通过basePackages等属性来细粒度的定制@ComponentScan自动扫描的范围，如果不指定，则默认Spring框架实现从声明@ComponentScan所在类的package进行扫描。
## 3. @EnableAutoConfiguration  
        从classpath中搜索所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableAutoConfiguration对应的配置项通过反射实例化为对应的标注了@Configuration的Java Config的IoC容器配置类，加载到IOC容器。
        在Spring框架中提供了各种以@Enable开头的注解，例如@EnableScheduling、@EnableCaching、@EnableMBeanExport等。@EnableAutoConfiguration的理念和做事方式就是：借助@Import的支持，收集和注册特定场景相关的bean定义。
        @EnableScheduling通过@Import将Spring调度框架相关的bean定义都加载到IOC容器。
        @EnableMBeanExport通过@Import将JMX相关的bean定义加载到IoC容器。
        @EnableAutoConfiguration借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IOC容器。
        @EnableAutoConfiguration定义如下：
```
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
...
}
```
### 3.1 @AutoConfigurationPackages
        注册当前主程序类的同级以及子集的包中的符合条件（如@Configuration）的Bean的定义；
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
 
}
```
### 3.2 @Import(AutoConfigurationImportSelector.class)
        扫描各个组件jar META-INF目录下的spring.factories文件，将下面的包名.类名中的工厂类全部加载到IOC容器中。
