---
title: Kotlin使用记录01
date: 2021-12-17 09:18:07
categories:
- 编程
tags:
- Kotlin
- 协程
hide: false
comments: true
index_img: /gallery/逆光剑フラガラック.jpg
banner_img: /gallery/逆光剑フラガラック.jpg
---
Kotlin是一个完全兼容java的jvm语言，但不是说什么地方都跟java一样，只是说可以完全调用java。而也有于此，导致Kotlin一些使用方面需要注意一些东西，例如标准库不一定是跟JDK用法相同。  
不过我也没有打算完全记录，可能更多的还是做个TODO列表~~防健忘~~
<!--more-->
> 封面  
> 逆光剑，Fate/hollow ataraxia中バゼット使用的武器之一。  
> 在Fate/unlimited codes中与Lancer的Gae Bolg同时发动会有奇效：由于皆为因果逆转宝具——注定命中对方的心脏；后发但会先贯穿对方。两方会由于逆转因果失败而同时受到宝具伤害后倒地。

## Kotlin的标准库
Kotlin的标准库是有别于Java的，注意调用时点进去看是什么包下面的。~~其实好像也就想到了list跟map~~
### List, Map等
List和Map是有些非常方便的方法构建的，`listOf("xxx", "xxx")`, `mapOf("xxx" to "xxx", "xxx" to "xxx")`，这些用起来非常爽快。但需要注意这里返回的类型是Kotlin下的List跟Map，它所提供的方法很有限，都是不可变的方法。但具体返回的对象则不一定是不可变的，具体点进方法内看就知道了。  
那么我们需要一个可变的List或是Map时就需要让方法返回MutableList或是MutableMap的对象，可以用`mutableOfList()`或`mutableOfMap()`，当然你也可以用is去看是不是某类型的实例，大多其实还是返回的JDK里面的list跟map。
## 扩展函数
Kotlin要说我感到最爽的东西，大概除了协程就是这个了。
```kotlin
object MarkdownUtil {
    private val formatChar = "_*[]()~`>#+-=|{}.!".toCharArray()
    
    //扩展函数，可以直接就等于一个方法，也可以调用该类内部方法，this指代的就是当前调用这个方法的String对象
    fun String.format2Markdown(): String = MarkdownUtil.format(this)
    
    private fun format(target: String): String {
        var result = target
        for (c in formatChar) {
            result = result.replace(c.toString(), "\\$c")
        }
        return result
    }
}
```
上面的例子是我写Telegram Bot时写的一个方法，用来转换掉markdown的特殊字符，调用时只需要`"Some String".format2Markdown()`，这就像String类扩展了一个方法一样，当你有许多这样的方法的时候，也许你就可以像链式调用一样写起来特别爽快了。

这样对于一些非自己项目内的代码进行非常简单的扩展也会大大增加写代码的效率，而不需要额外调用一个工具类对该变量做处理（不需要管其他工具类，只要点出来就好了）等等。

## Lambda
Kotlin的Lambda整体感觉是要强于java很多的，java很多情况下的推断都不太行导致很多地方没法使用，另外一个是Kotlin对Lambda的支持要能够放入更多的地方，例如你可以直接将一串Lambda赋值于一个变量而不需要特别的写一个函数接口，拿到的一个类型就是Unit。

## Type safety
类型安全也是Kotlin的一个区别于java的特色，虽然java其实可以用Optional作为代替，但是Kotlin也是可以使用的。  

但Kotlin自身的类型安全是你声明时是否声明一个空变量，而后编译的时候就给你检测报错，并且强制要求你做一次判空，但Kotlin对判空进行了简化，用`value?.doSomeThing()`的方式先做判空，若是空则不会调用后面的方法。

但实际上强制你做类型声明的时候生命是否为空也是很烦人的，因为后面你到哪里都要做一次判空，虽然Kotlin能够推断当前的变量是否为空而后不需要判空，但他无法对变量内部的成员变量做判断，就算你判空过一次他也无法后面不需要判空，虽然可以赋值到当前的变量当中但是仍然是比较麻烦的。

那么这个时候其实可以考虑用Java写POJO类，Kotlin对Java类都是不做强制要求判空的。

或者就需要煞费苦心分别分开不同的类比如，对接其他API的时候实际上返回的是一个聚合了很多东西的类，不同情况下不同的字段是有可能空或者非空的，如果都用一个类就会造成几乎所有字段都是可能空的尴尬，那么分开几个类型来接受就能够确切的知道当前的哪些字段是否空了，
这样也许就比较符合我们想要的`type-safe`了。

## 推断
Kotlin的推断能力在上面也有提到，再结合IDE就会边得异常爽快，例如上面提到的，如果这个变量进行过判空操作，后面就会自动的认为这个变量是非空的而不需要再次强制要求判空。  

又比如java当中非常繁琐的一种操作是`if a instanceOf B`，`b = (B)a`，然后用这个b变量操作，而Kotlin直接`is`一次判定为真后自动推断你这个变量就是某类型，而后可以直接用该变量调用该类型的方法。  

而这些在IDE上是都有提示的，最典型的就是`val a = "123"`在IDE当中是可以显示a的类型而不需要再手动定义类型了，当然你也可以显式声明类型。

## 协程
这个留在最后讲其实主要我也不是特别熟，但不得不说这也是Kotlin的一大爽点吧。但其实他也不是真正的协程，只是做了一层封装，背后实际大概是个线程池。但你不可否认这比起大多的异步编程框架还是爽上不少的，毕竟可以直接写类似同步代码的代码达到异步编程的效果实在太棒了。但说实话我在写自己项目时，也有一些不解的地方，或者说我还不太了解它的运行轨迹。  
现在了解到的就是，实际上他是做了一个标记，类似占位符，提交这段代码到线程执行，然后马上看执行结果，若是完成则返回结果，否则就是一个占位符（我没有看过源码，也不打保票就是这样，毕竟我自己写代码的时候还是有一些问题的）。

### 一些问题
- 在运行一些长时间保持运行的项目当中，容易造成内部卡死。  
    e.g.
    ```kotlin
    scope.launch {
        while (isActive) {
            try {
                feedAndSend(rss, group)
                delay(1000 * 60)
            } catch (e: Exception) {
                log.error(e) { "执行订阅出错。" }
            }
        }
        log.warn { "Coroutine was inactive." }
    }
    ```
- 另外还有类似接收到服务器发送的信息进行处理，运行久了就会无报错信息卡住不动，但最初的是时候我使用Dispatch.IO，之后我自己构建线程池指定队列长度以及线程配置后就还没发生过问题。

未完待续...