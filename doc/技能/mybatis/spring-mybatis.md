# 一、MyBatis-Spring-Boot-Starter简介
        MyBatis-Spring-Boot-Starter类似一个中间件，链接Spring Boot和MyBatis，构建基于Spring Boot的MyBatis应用程序。
        MyBatis-Spring-Boot-Starter是个集成包，对MyBatis、MyBatis-Spring和SpringBoot都有依赖。

# 二、MyBatis-Spring-Boot-Starter功能
1. 自动发现的DataSource；
2. 利用SQLSessionFactoryBean创建并注册SQLSessionFactory；
3. 创建并注册SQLSessionTemplate；
4. 自动扫描Mapper，并注册到Spring上下文环境方便程序的注入使用；

# 三、MyBatis-Spring-Boot-Starter扫描
        默认情况下，MyBatis-Spring-Boot-Starter会查找以@Mapper注解标记的映射器。
        你需要给每个MyBatis映射器标识上@Mapper注解，但是这样会非常的麻烦，这时可以使用@MapperScan注解来扫描包。
        