# 一、什么是Spring MVC
        Spring MVC是一个基于Java的实现了MVC设计模式的请求驱动类型的轻量级Web框架，通过把Model、View、Controller分离，将web层进行职责解耦，把复杂的web应用分成逻辑清晰的几部分，简化开发、减少出错，方便组内开发人员的配合。

# 二、Spring MVC流程
![blockchain](/resource/images/spring%20mvc请求流程.jpg "Spring MVC流程")  
  - 用户发送请求至前端控制器DispatcherServlet；
  - DispatcherServlet收到请求后，调用HandlerMapping处理器映射器，请求获取Handler；
  - 处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器一并返回给DispatchServlet；
  - DispatcherServlet调用HandlerAdapter处理器适配器；
  - HandlerAdapter经过适配调用具体处理器；
  - Handler执行完成返回ModelAndView；
  - HandlerAdapter将Handler执行结果ModelAndView返回给DispatcherServlet；
  - DispatcherServlet将ModelAndView传给ViewResolver视图解析器进行解析；
  - ViewResolver解析后返回具体view；
  - DispatcherServlet对view进行渲染视图；
  - DispatcherServlet响应用户。

# 三Spring MVC的主要组件
## 1. 前端控制器（DispatcherServlet）
        接收请求、响应结果，相当于转发器，有了DispatcherServlet就减少了其他组件之间的耦合度。
## 2. 处理器映射器（HandlerMapping）
        根据请求的URL来查找Handler
## 3. 处理器适配器（HandlerAdapter）
        在编写Handler的时候要按照HandlerAdapter要求的规则去编写，这样适配器HandlerAdapter才可以正确的去执行Handler；
## 4. 处理器（Handler）
## 5. 视图解析器（ViewResolver）
        进行视图的解析，根据视图逻辑名解析成真正的视图。
## 6. 视图View
        View是一个接口，它的实现类支持不同的视图类型（jsp、freemarker、pdf等）；