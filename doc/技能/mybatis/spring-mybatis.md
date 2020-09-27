# 一、MyBatis-Spring间接
        MyBatis-Spring会帮助你将MyBatis代码无缝的整合到Spring中，它允许MyBatis参与到Spring的事务管理中，创建映射器Mapper和SqlSession并注入到bean中，以及将MyBatis的异常转换为Spring的DataAccessException。最终可以做到应用代码不依赖与MyBatis、Spring、或MyBatis-Spring。

# 二、SqlSessionFactoryBean
        在基础的MyBatis用法中，是通过SqlSessionFactoryBuilder来创建SQLSessionFactory，而在MyBatis-Spring中，则使用SqlSessionFactoryBean来创建。

# 三、SQLSessionTemplate
        SqlSessionTemplate是MyBatis-Spring的核心，作为SqlSession的一个实现，这意味着可以使用它无缝代替你代码中已经在使用的SqlSession。
        SqlSessionTemplate是线程安全的，可以被多个Dao或映射器所共享使用。


# 四、MyBatis-Spring-Boot-Starter简介
        MyBatis-Spring-Boot-Starter类似一个中间件，链接Spring Boot和MyBatis，构建基于Spring Boot的MyBatis应用程序。
        MyBatis-Spring-Boot-Starter是个集成包，对MyBatis、MyBatis-Spring和SpringBoot都有依赖。

# 五、MyBatis-Spring-Boot-Starter功能
1. 自动发现的DataSource；
2. 利用SQLSessionFactoryBean创建并注册SQLSessionFactory；
3. 创建并注册SQLSessionTemplate；
4. 自动扫描Mapper，并注册到Spring上下文环境方便程序的注入使用；

# 六、MyBatis-Spring-Boot-Starter扫描
        默认情况下，MyBatis-Spring-Boot-Starter会查找以@Mapper注解标记的映射器。
        你需要给每个MyBatis映射器标识上@Mapper注解，但是这样会非常的麻烦，这时可以使用@MapperScan注解来扫描包。
