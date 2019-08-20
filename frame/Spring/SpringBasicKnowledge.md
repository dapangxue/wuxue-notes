# Spring的基础知识

## 基本概念

### 基本组件

Spring常见的组件有Core Container、AOP、Web、Data Access。

Core Container是Spring的核心，可以说Spring的功能是基于Core Container来构建的，内部装载了用户需要的各种Bean。控制反转的概念也由此而来，实例化对象的构建不再由用户自己构建，而是有Spring容器自己构建。

Data Access层包含了JDBC、ORM、Transaction模块。主要用于控制数据访问。

### Spring IoC概念

SpringIoC是一个容器，容器中的Java资源叫做Bean，容器就是用来管理这些Bean和它们之间的关系的。抽象的描述就是不需要再自己实例化资源，只需要描述需要什么资源，Spring IoC会从容器中找到所需要的资源。

### Spring AOP

SpringAOP的作用就是将代码横切，给特定的切点（Point）插入特定的Advice，和OOP纵向的继承不同，它是横向的切分方法。

Spring AOP使用了两种代理机制，一种是基于JDK的动态代理，另外一种是基于CGLib的动态代理。

动态代理就是开发者在运行期间创建接口的代理实例。

基于JDK的动态代理主要是用到`java.lang.reflect`包下面的两个类，Proxy和InvocationHandler，通过实现InvocationHandler定义横切逻辑，并通过反射机制调用目标目标类的代码，将横切逻辑和业务逻辑编织在一起。

基于CGLib的的动态代理采用底层字节码的技术，为一个类创建子类，在子类中采用方法拦截的技术拦截父类的方法，并且织入横切逻辑。但是CGLib的动态创建子类的方式受到了Java继承规则的约束，final/private/default标识的方法无法进行代理。