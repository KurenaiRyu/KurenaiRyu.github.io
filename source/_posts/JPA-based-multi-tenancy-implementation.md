---
title: 基于 JPA 的多租户实现
date: 2025-06-04 16:39:44
categories:
  - 编程
tags:
  - JPA
hide: false
comments: true
toc: true
index_img: https://gamesci.cn/wukong/img/blackmyth_wukong_wallpaper_037.1798fb5a.jpg
banner_img: https://gamesci.cn/wukong/img/blackmyth_wukong_wallpaper_037.1798fb5a.jpg
---
在工作开发中我遇到了需要链接多个DB项目的情况，业务上以不同的医院进行请求查询，而一个cluster能够包含几个医院，一个cluster则会划分到一个DB（或schema）中，基本上可以认为不同cluster为不同的db connection，至此该项目就像是多租户上划分多个db的情况了。
<!--more-->
## Requirement
- Thread-Safe
  基于servlet一个请求绑定一个线程
- 根据请求切换至对应数据库
- 链接多于一种类型数据库
  由于业务还需要逐步从A数据库到B数据库，这期间一定会有链接两种数据库的情况
- 监听/定时刷新激活的数据库配置
  即哪些cluster已转移至B数据库，哪些没有

## Thinking
由于传统Servlet为一个请求绑定住一个线程，除非自己切换不然都能够安全通过ThreadLocl来作为上下文确定当前应该切换至哪个cluster。
但需要注意，每个请求过来绑定的线程不一定会在请求结束后销毁，也许会被重复利用在下一个请求当中，所以需要加入一个OnePerRequestFilter在请求结束后清空ThreadLocal上下文已保证新请求绑定已有线程不被污染。
Btw，Reactive就要用Reactive的线程上下文了。

激活的数据库配置初步是定为配置文件放在minio中共各个服务读取，以达成统一配置减少线上部署人员的工作，定时刷则用timer定时跑一个task重复刷新配置文件，这里需要给到相关变量读写锁的控制，另外在初始化timer的时候要用双检锁保证exactly-once initialize(瞎掰的词，源自exactly-once delivery)。

而链接不同数据库，在同一种数据库下是可以直接用[AbstractRoutingDataSource](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jdbc/datasource/lookup/AbstractRoutingDataSource.html)做动态切换的，但你需要在事务开启前做这个事情，另外还需要注意，`spring.jpa.open-in-view`需要关闭它以避免请求过来的时候就过早开启一个session导致各种问题的发生，参考[这里](https://stackoverflow.com/questions/30549489/what-is-this-spring-jpa-open-in-view-true-property-in-spring-boot)。
但需求是链接两种数据库，那么你还需要对Entity Manager做Routing来确保两种不同的数据库方言能够正运行。
所以下面就开始了部分源码的阅读以及调试时实际情况是如何的了。

## Learning & Debug
实际上，如果不希望冒着这样的风险做改动，最简单的做法就是将配置复制N分，遇到什么cluster就用什么repository，但是我们有7个cluster x 2种数据库，感觉还是太蠢了。
那么动态生成repository呢？我想应该是可以的，但他需要一个entitymanger，那你是不是又要自己动态生成一个entitymanager呢？然后一系列的东西就会指引到TransactionMnager上。

经过断点，`DataSource#getConnection`以及`EntityManagerFactory#createEntityManager`，然后你就会发现在一个Transaction内他们是只会调用一次的，而调用的地方就是`JpaTransactionManager#doBegin`，也就是进入被`@Transactional`注解的方法前，而在前面两个方法创建后，`EntityManager`会放入这次Transaction的上下文对象`JpaTransactionObject`(该对象在doGetTransaction方法生成)的`EntityManagerHolder`中，并且绑定到`TransactionSynchronizationManager`，`DataSource`则只会绑定至`TransactionSynchronizationManager`。
也就是说，spring事务开启后，处于事务期间你是无法切换`EntityMnager`以及`DataSource`的，除非你自定义一个`TransactionMnager`，但是如果我们自定义事务管理器可以拿两个`EntityManagerFactory`，但是`Repository`的配置中只会认一个`EntityManagerFactory`，所以最终你还是必须有一个Routing的`EntityManagerFactory`。

之后根据源码可以了解到实际上事务管理器没有用到太多`EntityManagerFactory`的接口，我们只需要重写`AbstractEntityManagerFactoryBean`下的`createNativeEntityManagerFactory`, `getNativeEntityManagerFactory`，这些方法都根据当前上下文的cluster以及迁移的db类型配置路由到对应的FactoryBean上就好了，一般直接拿`nativeEntityManagerFactory`就好了，`destroy`方法直接调用一边维护的所有FactoryBean就好了。

为什么是FactoryBean？因为这是特殊的类型，实际注入的是getObject拿到的对象，而`AbstractEntityManagerFactoryBean`的对象则会维护在`nativeEntityManagerFactory`变量上。

这样我们也就可以用一个repository实现需求了。

但是且慢，我们知道事务注解下的情况，那非事务下呢？
实际上这个也是最开始我有做断点了解到的，非事务下是会有一个shared相关的实例（具体忘了是啥了），他会绑定到Primary的`DataSource`,  `EntityManagerFactory`，所以这也是为什么思考方向会朝着一个单独的一个配置做Routing去。
Btw，务必保证有一个主要的一套配置在，不然JPA Auto Configuration不会生效。

至此我们就能够实现之前提出要求了。

## Implement
只重点说一下Jpa的配置，首先需要两套`DataSource`，`EntityManagerFactory`，分别为A数据库类型和B数据库类型，而`DataSource`是一个Routing到同类型不同数据库的数据源，再加入一个主要数据源Routing不同数据库类型到实际的Routing数据源，一个主要`EntityManagerFactory`并自定义实现的Routing功能就好了。融合一个大的主要数据源还有个原因是为了可以用`JdbcTemplate`

## Conclusion
Demo可以在[这里](https://github.com/KurenaiRyu/multi-db-demo)看到，并且这是非常实验性的，还未跑过大量数据以及业务去验证其正确性，例如多个不同传播类型的事务互相调用互相嵌套的情况。另外`AbstractEntityManagerFactoryBean`也不是很能够确定是否这样重写几个接口就没有问题，只是目前发现`TransactionManager`没有过多使用其他接口。

