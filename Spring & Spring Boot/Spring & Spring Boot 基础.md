## IoC 和 DI

### IoC(Inversion of Control)

**控制反转**，是一种**设计原则**，核心思想是将对象之间依赖关系的控制权交给容器去管理，而不是我们自己 new 对象，各个组件之间能**解耦**。这样我们能从复杂的依赖关系中解放出来，专注于业务逻辑实现。

### DI(Dependency Injection)

**依赖注入**，是 IoC 的一种**实现方式**。

### IoC 比较常见的两种实现方式：

- **依赖查找**（Dependency Lookup）：即**根据某种标识符从特定的容器或上下文获取依赖**，比如 Spring 通过 beanName 从容器中获取组件。除了 Spring，很多开源框架通常会有一个用于存放各种组件的全局配置类或上下文，等运行时再从中获取相关组件（如 MyBatis），这种方式也可以理解为一种依赖查找。
- **依赖注入**（Dependency Injection）：即根据配置，**由框架在特定的时机主动地将依赖通过构造器或方法注入到对象中**，Spring 就是用来干这个的。

Spring 就是目前 Java 中最主流的 IoC 框架，它既支持依赖查找，也支持依赖注入。

## 在 Spring 中配置 Bean

在 Spring 中，通过配置告诉 IoC 容器**如何创建 Bean、如何注入依赖、如何管理 Bean 的生命周期**。

早期的 Spring 应用大量使用 applicationContext.xml 等 XML 文件来定义 Bean 的类名、作用域、依赖关系，配置 AOP、事务、数据源等。这种方式清晰但是冗长，不好维护。

后来 Spring 逐渐支持基于注解的配置：

- @Coponent/@Repository/@Service/@Controller：声明 Bean
- @Autowired/@Qualifier：注入依赖
- @Configuration/@Bean：以 Java 类形式配置 Bean
- @Value/@PropertySource：注入配置属性

大大简化了配置流程，更加直观、灵活。

Spring Boot 在注解配置上进一步增强：

- 自动配置（@EnableAutoConfiguration）
- 属性配置（application.yml/application.properties）
- 提供丰富的 Starter 依赖，减少手动配置工作量