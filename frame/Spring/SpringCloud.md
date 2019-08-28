# SpringCloud

## SpringCloud常用的组件

SpringCloud的常用组件有Spring Cloud Config、Spring Cloud Bus、Eureka、Hystrix、Zuul、Spring Cloud Security、Spring Cloud CLI、Ribbon。

Spring Cloud Config在分布式系统中，为外部配置提供服务端和客户端支持。通过使用Config Server,可以在所有环境中管理应用程序的外部属性。一般使用git管理。

Eureka是一个服务发现组件，服务发现是基于微服务架构的关键原则之一，尝试配置每个客户端比较繁琐，Eureka尝试将服务器配置和部署为高可用性。

Hystrix是SpringCloud的一个断路器方案，服务调用可能会引起级联故障，Hystrix通过回退防止级联故障。

## 参考文献

[外行人都能看懂的SpringCloud，错过了血亏！](https://juejin.im/post/5b83466b6fb9a019b421cecc)
