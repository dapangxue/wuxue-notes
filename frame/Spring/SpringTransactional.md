# Spring事务方式总结

一般在项目中对数据库的操作都会用到事务，事务主要是用来处理复杂的多个操作的关系。

在Spring中提供的事务的处理方式有：

+ 编程式事务
+ 使用XML配置的声明式事务
+ 使用注解配置的声明式事务

Spring提供了多种事务管理器，最顶层的类就是`PlatformTransactionManager`,也就是说其他的事务管理器必须直接或间接继承自`PlatformTransactionManager`，比如MyBatis常用的事务管理器`DataSourceTransactionManager`。

## 编程式事务

编程式事务一般很少使用，它需要在一个操作上引入的额外代码太多，对代码的侵入性太大，使得开发人员很难专注于业务。

Spring为编程式事务提供了模板类`TransactionTemplate`，然后提供了对应的执行事务的接口，`TransactionCallback`（如果有返回结果）和`TransactionCallbackWithoutResult`（不需要返回结果)的接口。

## 声明式事务

声明式事务是一种约定型事务：
+ 如果没有出现异常，那么事务可以提交。
+ 如果出现了开发者允许提交事务的异常，Spring也会让事务管理器提交事务。
+ 如果出现了不被开发者配置信息所允许的异常，Spring会让事务管理器回滚事务。

### 1.基于XML配置的声明式事务

Spring声明式事务是基于AOP实现的，Spring将事务管理增强逻辑动态织入业务方法的相应切点中。

现在常用的基于XML的声明式事务都是基于aop/tx命名空间的配置，需要在XML配置中添加tx的命名空间，然后通过aop指定对应的事务方法的策略。

### 2.基于注解配置声明式事务

基于注解的声明式事务首先需要配置特定的事务管理器，对于常用的数据库持久层框架Mybatis，可以采用XML配置事务管理器方式打开注解事务，也可以采用Java配置的方式实现Spring数据库事务（采用EnableTransactionManagement注解）。

使用到事务的时候在对应的方法或者类上加上@Transactional注解，可以指定是否采用数据库默认的隔离级别。

