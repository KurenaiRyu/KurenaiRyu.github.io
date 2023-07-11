---
title: GraalVM With Spring
date: 2023-07-12 0:21:00
categories:
    - Java
tags:
    - GraalVM
comments: true
index_img: /gallery/103834884.jpg
banner_img: /gallery/103834884.jpg

---
最初看见 GraalVM 的时候我就想到 **Make jvm great again!** ww  
实际上我也在之前尝试过几次，但都是不理想或者失败了。这次我看到 Spring Boot 3 对 GraalVM 的支持更加完善了，就又想跑过来试试了，毕竟写 Kotlin 编译成 Native 还是太香了！
同样这次也都还是只做一些折腾的记录。
<!--more-->
> 封面：画师是~~我岳父~~[おにねこ](https://www.pixiv.net/users/3952)，[封面链接](https://www.pixiv.net/artworks/103834884)。
> 就说这位画师非常适合黑色哥特萝莉，201x年初时其实脸跟一些细节都让人感觉比较潦草，但是非常个人风格，背景也很好。现在整体都相较要好很多，细节要更加多。
---
## 前言
GraalVM 与比传统在 JVM 运行是 JIT 编译不同，他是 AOT 编译。由于去掉了 JVM 后一些运行时的动态分析就会失效，如反射，代理，JNI，SPI 等等，而且这里面一些资源也要显式声明，所以需要你需要大量的去告诉编译器这个类实际上运行时是谁，让它去分析这个类、资源的信息，最后才能够在运行时找到。

这其实几乎就是转 GraalVM 的全部工作了，或者说这里面的工作量实在太大了。最初时大多项目都没有提供支持，导致你需要自己等待编译，然后运行报错看提示再去告诉编译器需要分析什么东西。

如今，大多常用的包都有给出一些反射、代理或者资源的配置了，所以我又准备尝试了。
## 环境准备
### Windows
Windows 环境非常恶劣，起初我几乎放弃了。
就算是现在我也不能说我真的就了解怎么搭建环境了。
所以珍惜生命远离 Windows（

Ok，其实现在 GraalVM 的官网上面已经放出了一篇[教程](https://medium.com/graalvm/using-graalvm-and-native-image-on-windows-10-9954dc071311)，只需要安装好 vs 当中的一些组件即可

然后**重点**：找到`x64 Native Tools Command Prompt for VS 2022`这个快捷方式，打开进入cmd，可以看到有个x64的输出，这个环境就算准备好了，2022可以是别的，但是前缀应该就是这些，x86是不支持的，交叉编译不知道。

最后配置一下环境变量 `GRAALVM_HOME` 就好了了，跟 `JAVA_HOME`，这是为了 gradle 的 graalvm 的插件（`org.graalvm.buildtools.native`）做准备。

### 其他
其他大概就不太需要说明了，只要装上 GraalVM 的 sdk 就行了。

## GraalVM 配置
GraalVM 的配置是放在 `resource/META-INF/native-image/` 下面，我不太记得非 spring 项目下是不是默认读取这个目录下的配置了，这块现在[这个](https://github.com/oracle/graalvm-reachability-metadata)仓库有维护一些配置，可以直接拿过来用，主要是 copy 里面的 `reflect-config.json`, `proxy-config.json`，`jni-config.json`，`resource-config.json`

## Spring 配置
依赖什么的这个也没有什么特别说的，直接在[start.spring.io](https://start.spring.io/)上面选好依赖，查看它配置文件怎么写就好了。

## RuntimeHintsRegistrar
然后因为自己手写 json 不太可能，然后 GraalVM 也有一个 agent 虽然可以分析一下运行时情况，但是也不一定能够分析全，所以我们就需要`RuntimeHintsRegistrar`来代码生成。

这个类貌似GraalVM也有提供，但是因为用的 Spring，GraalVM 提供的就没有去研究了。

```kotlin
// hints 和 classLoader 都是实现这个类的`registerHints`方法传入的参数

// 私有类无法访问时
hints.reflection().registerTypeIfPresent(classLoader, "com.github.benmanes.caffeine.cache.PSWMW", MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS)

// 一般情况，后面的 MemberCategory 类型就是要注册的东西，可以是全部方法、字段或者是 public 的方法、字段等等
hints.reflection().registerType(ConstructorDetector::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_METHODS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.DECLARED_CLASSES)

// 拥有内部类的时候
AtomicLongFieldUpdater::class.java.declaredClasses.forEach {
    hints.reflection().registerTypeIfPresent(classLoader, it.name, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_METHODS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.DECLARED_CLASSES)
}
// 一些基础类型的注册
hints.jni().registerType(java.lang.Boolean::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(java.lang.Integer::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(java.lang.Long::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(java.lang.Double::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(java.lang.String::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(java.util.Arrays::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(Array::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(IntArray::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(LongArray::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(DoubleArray::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
hints.jni().registerType(BooleanArray::class.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)

// jni内部类的注册
TdApi::class.nestedClasses.forEach { clazz ->
    if (clazz.isSubclassOf(TdApi.Function::class).not()) {
        hints.jni().registerTypeIfPresent(classLoader, "${clazz.name}[]")
    }
//            clazz.java.declaredMethods.forEach {
//                hints.jni().registerMethod(it, ExecutableMode.INVOKE)
//            }
//            clazz.java.fields.forEach {
//                hints.jni().registerField(it)
//            }
    hints.reflection().registerType(clazz.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
    hints.jni().registerType(clazz.java, MemberCategory.DECLARED_FIELDS, MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, MemberCategory.INVOKE_DECLARED_METHODS)
}

// 资源的注册
hints.resources().registerPattern("META-INF/tdlightjni/*")
```

写完这个类后，找个`@Configuration`的类或者程序入口类添加`@ImportRuntimeHints(YourHintsRegirar::class)`，这样他就会编译的时候在`build/generated/aotResources`下生成对应的配置文件，可以打开检查一下生成有没有问题。

## 编译
Ok，万事俱备了，就剩下编译了。

> Windows编译必须要在刚刚说到的`x64 Native Tools Command Prompt for VS 2022`当中执行命令。

执行`gradle nativeCompile`等待编译完成。

编译完成会在`build/native/nativeCompile`目录下找到可执行文件，执行即可。

## 结语
这东西我 5600X 几乎需要花 3min+ 编译，当你不太确定或者一些库用了大量反射或者大量个人库的时候，你可能需要给他写大量的配置文件，重复 N 次编译，所以如果可以，还是在配置比较高的情况下玩，不然一天很快过去的www