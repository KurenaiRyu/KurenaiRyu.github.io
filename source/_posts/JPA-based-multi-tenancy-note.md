---
title: 基于 JPA 的多租户实现笔记
date: 2026-07-03 3:14
categories:
  - 编程
tags:
  - JPA
hide: false
comments: true
toc: true
index_img: https://gamesci.cn/wukong/img/blackmyth_wukong_wallpaper_016.7545da0e.jpg
banner_img: https://gamesci.cn/wukong/img/blackmyth_wukong_wallpaper_016.7545da0e.jpg
---
记录一下折腾这个多租户实现的一些情况。至于之前的文章就无视了吧，毕竟只是比较实验的东西。
<!--more-->
虽然前面文章有给出过需求，不过这里我们简化一下，更专注于多租户实现这个问题上。  

可以在[这里](https://github.com/KurenaiRyu/multi-tenancy-with-jpa-demo)看到一个简单的demo
## Requirement
- 一个 server 内有多于一种数据库类型（sybase, pgsql) 同时链接
- 一个请求下，或者说一个 transaction 下希望能够做到不同 datasource 的切换
- 因为是多个数据库所以需要支持 2pc 保证多数据的写入一致性

如果你有跟过 Spring 的事务流程，你可能对这些需求有点绝望。

Spring 本身实现虽然有可拓展部分，例如JTA的具体实现会委托给 UserTransaction，但也有许多核心代码非常的严格限制来自外部的修改/拓展，往后面我会有提到这块重点改动，也是最为头疼的问题。

## 0x00
我们先看需求

支持 2pc 则需要用到 `JtaTransactionManager` 作为事务管理器。

多数据库类型就是多个 `EntityManager` 实例用不同的方言，单纯的动态数据源是不行的，Hibernate 的多租户支持也只是切换 `ConnectionProvider`，也就是我们需要动态 `EntityManager`。

同一个事务下做数据源的切换仍然是动态 `EntityManager` 的问题，但需要动到事务的一些配置类以及 `Repository`相关的配置类。

## 0x01

由于能力以及文章篇幅原因，我将从事务的运行流程作为切入点开始讲。

首先，一个被标注了 @Transactional 的方法会被 `TransactionIntercepter` 拦截到，然后大概的调用顺序是：
```text
org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction
↓
org.springframework.transaction.interceptor.TransactionAspectSupport#createTransactionIfNecessary
↓
org.springframework.transaction.support.AbstractPlatformTransactionManager#getTransaction
→   org.springframework.transaction.support.AbstractPlatformTransactionManager#doGetTransaction
    →   org.springframework.orm.jpa.JpaTransactionManager#doGetTransaction    
    →   org.springframework.transaction.jta.JtaTransactionManager#doGetTransaction   
→   org.springframework.transaction.support.AbstractPlatformTransactionManager#startTransaction
→   org.springframework.transaction.support.AbstractPlatformTransactionManager#doBegin    
    →   org.springframework.orm.jpa.JpaTransactionManager#doBegin
    →   org.springframework.transaction.jta.JtaTransactionManager#doBegin
```

上面的 `doGetTransaction` 与 `doBegin` 就是我们重点要关注的实现方法了，而 `AbstractPlatformTransactionManager` 基本上定义整个基本事务的基本流程，`JpaTransactionManager` 和 `JtaTransactionManager` 则是一般事务与 jta 事务实现，其中 jta 几乎都委托给了 `UserTransaction`。

### JpaTransactionManager
我们先看 `JpaTransactionManager#doGetTransaction`，会发现里面单纯是从 `TransactionSynchronizationManager` 拿到资源做绑定，点进 `TransactionSynchronizationManager` 可以看到它维护了多个 `ThreadLocal`，它就是对整个事务一些上下文做了线程绑定。

接下来是`JpaTransactionManager#doBegin`，这里面则是具体开启事务的一些操作，可以看到里面同样会涉及到 `TransactionSynchronizationManager`，并且在这个方法执行过程中对资源进行了绑定，绑定 EntityManager 用的 Key 就 EntityManagerFactory。

到这里我们就可以发现 Spring 就是这样锁住整个事务中只有一个 `EntityManager` 以及 `DataSource`，`DataSource` 比较好解决，本身就有 `RoutingDataSource` 可以实现，但是 `EntityManager` 我们这样做路由代理就会发生问题，因为里面是存在上下文，`Entity` 缓存等。

### JtaTransactionManager
这个事务管理器就要看具体的实现了，我是用的 `Atomikos`，基本上来说它也有一个绑定资源的上下文管理器，但在最开始开启事务的时候并没有做资源绑定，资源绑定是发生在 XDatasource 上的，当拿出一个链接时会 enlist，所以它相对于 `JpaTransactionManager` 限制没有那么严。

## 0x02
前面我们了解到的一个阻碍需求的地方是 `TransactionSynchronizationManager` 绑定死一个 `EntityManager`，而用 Jta 则似乎没有问题。

但即使使用 Jta，我们在 `Repository` 调用流程中也会遇到 `EntityManager` 的资源获取问题。

### JpaRepositoryFactoryBean
`Repository` 是通过这个 FactoryBean 创建的，他最终默认会创建一个`SimpleJpaRepository`的代理对象，内部则是把 `EntityManager` 做了一下包装，而这个 `EntityManager` 又是 `SharedEntityManagerCreator` 创建的代理对象，实际为内部私有类 `SharedEntityManagerInvocationHandler`。

也就是说 Repository 每次的实际调用是委托给了 `SharedEntityManagerInvocationHandler`，而他的实现就是用 `TransactionSynchronizationManager` 做资源的获取与绑定，其中因为里面写死了 `EntityMangerFactory`, 也就导致这个上下文资源只有一个 `EntityManager`。  

这个流程实际上也是与直接注入一个 `EntityManager` 类似。

至此，我们就发现被这个 `SharedEntityManagerCreator` 将死了，因为他是静态方法调用创建的代理，代理调用的又是一个私有类，你除非把他这一整套事务核心代码里的 `SharedEntityManagerCreator` 替换掉，不然就只能干瞪眼了。

## 0x03
为了做最小改动，我们的切入点仍然还是 `JpaRepositoryFactoryBean`，在最后的 `afterPropertiesSet()` 做文章:
```kotlin
override fun afterPropertiesSet() {
    setEntityManager(TenantEntityManagerCreator.createTenantEntityManager(null))
    super.afterPropertiesSet()
}
```
可以看到，我写了个 `TenantEntityManagerCreator`，基本照抄 `SharedEntityManagerCreator`，将里面的 `EntityManagerFactory` 改写成了动态的情况，这样就能够对事务上下文绑定多个 `EntityManager` 了。

但于此同时，你会丢失 Spring 的事务调用链，从这里开始流程将强行转换成了你写的实现，换句话说，也就是你丢失了 `SharedEntityManagerCreator#createSharedEntityManager(jakarta.persistence.EntityManagerFactory, java.util.Map<?,?>, boolean, java.lang.Class<?>...)` 这个方法传入的参数，但好在通常情况下是没有传入什么参数的。
- EntityManagerFactory emf  
    这个可以无视，我们本身就希望动态的获取该对象，也知道该对象怎么获取。
- Map<?, ?> properties  
    创建时的一些配置，一般的 Repository 使用情况下是不需要传什么东西的，直接用动态拿到的 EntityManagerFactory 可以获取到，但不排除你有什么骚操作会传入跟 factory 不一样的配置
- boolean synchronizedWithTransaction  
    true 就好了，这个参数主要是用于`EntityManagerFactoryUtils.doGetTransactionalEntityManager(
  this.targetFactory, this.properties, this.synchronizedWithTransaction)`， true 意味者这个 `EntityManager` 纳入当前同步管理，一般 Repository 以及`@PersistenceContext` EntityManager 默认都应该纳入同步，实际就是 `SynchronizationType` 的值。
- Class<?>... entityManagerInterfaces
    这个只是最后生成的代理对象会实现哪些接口，通常不用传。而代码实现会对传入的该参数再加一个 `EntityManagerProxy`

至此我们就解决了这个事务上下文绑定问题了，剩下的其实都比较好办，就是对 `EntityManagerFacotryBean` 做改造，让其能够感知租户的变动从而做动态生成，以及 `Hibernate` 本身的多租户配置需要实现的一些配置类。

## 0x04
现在我们开启一个事务时

假如是 Jpa 事务管理器则会绑定一个 `EntityMnager`，当我们实际调用 Repository 方法时会被代理到我们自定义的代理对象 `TenantEntityManagerInvocationHandler`，假设绑定的 `EntityManager` 不是当前租户的资源，从事务上下文获取 `EntityManager` 时用的是不同的 `factory` 作为key，所以不存在，则新建一个并绑定到上下文中。

假如是 Jta 事务管理器则开始没有绑定 `EntityManager` 的动作，只是开启一个全局事务标志，后续 `Repository` 的调用则跟 Jpa 一样代理到我们自定义实现的类中动态拿到 `EntityManager` 并绑定资源。