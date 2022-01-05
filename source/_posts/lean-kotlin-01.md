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
## 有别于Java中一些特殊的方法
~~我不知道这东西叫啥。~~  
Kotlin要说我感到最爽的东西，大概除了协程就是这个了。
```kotlin
object MarkdownUtil {
    private val formatChar = "_*[]()~`>#+-=|{}.!".toCharArray()
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
## 协程
讲Kotlin爽快点那肯定少不了协程了，但其实他也不是真正的协程，只是做了一层封装，背后实际大概是个线程池。但你不可否认这比起大多的异步编程框架还是爽上不少的，毕竟可以直接写类似同步代码的代码达到异步编程的效果实在太棒了。但说实话我在写自己项目时，也有一些不解的地方，或者说我还不太了解它的运行轨迹。  
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