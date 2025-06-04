---
title: Spring JPA 笔记 01
date: 2024-8-3 23:22:00
categories:
    - 编程
tags:
    - JPA
comments: true
index_img: /gallery/86588827_p1.jpg
banner_img: /gallery/86588827_p1.jpg

---
记录一下最近工作用jpa的一些心得，或者说就是坑（笑）
<!--more-->
> Banner Illustration: [荻pote](https://www.pixiv.net/users/2131660)  
> 非常通透的画面感，以及大多都是初/高中生。图里面一般都带有一定的叙事。但其实很多细节感觉还是不太好的，例如本图的衣服皱褶。但是个人风格非常明显，我还是很喜欢的。  
> https://www.pixiv.net/artworks/86588827
---
## 关联查询
众所周知，JPA Fetch Type 有 `Eager` 和 `Lazy`，而 `Eager` 只能有一个，那么其他就要在读取的时候（例如 `getter`）才会查库。这在某些量比较大的关联的表是不友好的，甚至有时候并不想查出来，这也困扰着我现在做的项目。

经过一番搜索，我看到大家都推荐用 `Entity Graph` 来解决这个问题，可是新的问题出现了，官方的 `Entity Graph` 是不支持动态传入的，所以我又找到了 [spring-data-jpa-entity-graph](https://github.com/Cosium/spring-data-jpa-entity-graph) 这个项目，允许运行时传入 `Entity Graph`，所以这个问题也暂时告一段落了。

但是好景不长，由于项目本身是接手的以前的团队，另外实体类的关联规划也都早就写好了，不太可能改，而原想团队写这些实体类的时候几乎就是滥用的程度，想要查什么或者看起来就是能够关联上的就直接写进去，这让我们在开发报表类的需求时极其头疼，因为我们发现 `JPA`/`Hibernate` 就算你不去动那些懒加载的关联项，也会因为某种原因被触发，而且就我们项目来说几乎是 `Repository` 方法调用拿到结果赋值给一个对象的时候就发生了，再往里面跟踪我就实在是不太能够理解代码了，而且看起来几乎没有办法阻止。

所以我们就开始考虑直接用 `Entity Graph` 对那些懒加载（假如他会，因为不是所有懒加载都会去查询）的关联项都直接查询出来，毕竟比起懒加载一条一条查，一条 `sql` 查出来通常是更快的。  
可是新的问题又出现了，因为其关联的对象巨大的，层层加码，导致一条 `sql` 关联的表实在是太多了（而且可能是重复关联）。  

几经周折，我们发现对于一些巨大的表来说，拆分查询，也就是查出来后把所有的 `id` 收集到后再去查库，会有更好的性能表现。但对于更加复杂的（我们报表开发几乎是基于存储过程再改写成 `java` 的）sql 就只能够用 native query（也就是直接写 sql）了。
> PS: 用 id 查询的时候有一个坑，就是巨大的 id 量也会导致其查询非常缓慢，所以有必要切分成几次查询，每次查2k左右

## 对象映射/空元素
空元素出现实际上代表着该条目是有数据的，但是在 `JPA`/`Hibernate` 在进行对象映射的时候，由于该 `id` 为null，亦或者是这是一个组合 `id`，其中有一个字段是null的时候，就会发生。

虽然常用数据库可以写入条件限制这个情况的发生，但不幸的是我们现在用的数据库 `Sybase` 其实是没有主键的说法的，所以我们只是用 `index` 来做限制以达到类似主键的限制的目的。但是其不对该字段为空做限制，而且原先业务数据库所存在的数据就是这样，要么对数据库里的数据进行清洗，要么就是在代码层面兼容。
> PS: 这里面仍然有一个问题，实际上我们数据库是空白串，char固定长度类型，但是仍然在get值时返回给到null，这块还没有确定是不是单纯数据库驱动的问题。

由于这些数据都是已经在用的了，做修改是非常大的风险， 而且也不单纯只是我们项目在用，除非客户本身已经考虑数据清洗/迁移了，所以数据清洗是不做考虑的。

所以我门最终是以修改 `JPA`/`Hibernate` 的数据映射达成的，就是对特定字段添加 mapper 注解调用自定义映射逻辑，把他改成空串。

重点在 `nullSafeGet` 方法
```java
public class NullToEmptyCharType implements UserType {


    @Override
    public int[] sqlTypes() {
        return new int[]{Types.VARCHAR};
    }

    @Override
    public Class<?> returnedClass() {
        return String.class;
    }

    @Override
    public boolean equals(Object x, Object y) throws HibernateException {
        if (x == null && y == null) {
            return true;
        } else if (x != null && y != null) {
            return x == y || x.equals(y);
        } else {
            return false;
        }
    }

    @Override
    public int hashCode(Object x) throws HibernateException {
        return Objects.hash(x);
    }

    @Override
    public Object nullSafeGet(ResultSet rs, String[] names, SharedSessionContractImplementor session, Object owner) throws HibernateException, SQLException {
        String value = rs.getString(names[0]);
        // 就是这一步拿出的value就算数据库是空白串，这里的值也是null
        if (value == null) {
            return CommonUtil.EMPTY_STRING;
        }
        return value;
    }

    @Override
    public void nullSafeSet(PreparedStatement st, Object value, int index, SharedSessionContractImplementor session) throws HibernateException, SQLException {
        if (value == null) {
            st.setNull(index, Types.VARCHAR);
        } else {
            String phone = value.toString();
            st.setString(index, phone);
        }
    }

    @Override
    public Object deepCopy(Object value) throws HibernateException {
        return value;
    }

    /**
     * Whether the type is variable
     */
    @Override
    public boolean isMutable() {
        return false;
    }

    /**
     * This method is called when the type is written to the second-level cache
     */
    @Override
    public Serializable disassemble(Object value) throws HibernateException {
        return (Serializable) value;
    }

    /**
     * This method is called when data is fetched from the level 2 cache
     */
    @Override
    public Object assemble(Serializable cached, Object owner) throws HibernateException {
        return cached;
    }

    @Override
    public Object replace(Object original, Object target, Object owner) throws HibernateException {
        return original;
    }
}
```

## Native Query with Stream
由于上面说到过我们项目会写 `Dynamic SQL`，也就是会调用 `nativeQuery` 方法做查询，就碰上了奇怪的问题，这个我暂时没有搞明白是怎么回事。  
简单说就是最后调用 `getResultStream` 获取结果的话，会导致异常的发生，说这个 `Stream` 已经执行过 `terminal` 操作，也就是执行了那些会触发 `Stream` 执行的方法。

Fix 也很简单，单纯就是不要用 `Stream` 接收他这个结果集，只用 List 操作。或者是用一个新的 List 接收再去做操作。



