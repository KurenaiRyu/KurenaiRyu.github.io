---
title: 用Gradle构建lib分离的jar包
date: 2022-06-23 21:52:00
categories:
    - Java
tags:
    - Java
    - Kotlin
    - Gradle
comments: true
index_img: /gallery/sv-5th-anni.jpg
banner_img: /gallery/sv-5th-anni.jpg

---
如果平时比较多使用docker部署，那么为了节省部署的时间，一般会把一些不太变化的东西放在比较前面写作为一层，分层处理后可以利用编译的缓存快速构建出镜像。

因此，如果一个jar包是分离lib的话就可以做灵活的分层处理了。
<!--more-->
> 封面：Shadowverse 5周年贺图，角色是~~我女儿~~雪华。BTW，这个角色的cv是刚出道的时候非常呆萌非常贪吃的[もちょ](https://www.bilibili.com/video/BV194411M7mt)。
---
## 0x01
最近由于接触了一下Quarkus，发现其打包jar的方式非常棒，分离了lib，最终推送docker镜像时利用缓存只需要1M不到的数据量，虽然我以前是知道maven有类似的插件的，但是由于最近我完全入坑gradle了，也尝试过找类似的插件但都未果，此次因为尝到了甜头，以及我不想要绑定quarkus框架所以又去找了一下解决方案。

## 0x02
首先Jar包内是有一个`META-INF/MANIFEST.MF`这样的文件，里面我只挑能够达成jar包跟lib分离目的的参数：
- Main-Class  
    这是指定程序入口的参数，一般就是main方法所在的class，Kotlin的class需要加上Kt的后缀才正确。
    这个一般打成jar运行是基础配置，不然无法运行。
- Class-Path  
    这个就是需要加载的lib的配置了，需要对每个lib包都做声明，空格分割。

下一步是对Gradle的jar任务做修改，让其排除所有的*.jar文件，并自定义一个删除以及拷贝lib的任务让其依赖。这需要了解一下Gradle如何自定义一个task以及jar任务如何修改。

## 0x03
了解完上面说的两点，就可以直接上代码了：
```kotlin
tasks.register<Delete>("clearLib") { //清除lib
    delete("$buildDir/libs/lib")
}

tasks.register<Copy>("copyLib") { //拷贝lib
    from(configurations.runtimeClasspath) //从运行时目录
    into("$buildDir/libs/lib")  //到打包目录
}

tasks.jar {
    dependsOn("clearLib") //依赖清除和拷贝lib任务
    dependsOn("copyLib")
    exclude("**/*.jar") //打包时排除jar文件（不打包成fat jar）
    manifest {
        attributes["Manifest-Version"] = "1.0"
        attributes["Multi-Release"] = "true"
        attributes["Main-Class"] = "moe.kurenai.bot.BgmApplicationKt" //main方法所在的class，我这个例子是用的Kotlin所以带有Kt后缀
        attributes["Class-Path"] = configurations.runtimeClasspath.get().files.map { "lib/${it.name}" }.joinToString(" ")  //构建出 lib/包名 的字符串并用空格分隔
    }
}
```

最终效果
```
Manifest-Version: 1.0
Multi-Release: true
Main-Class: moe.kurenai.bot.BgmApplicationKt
Class-Path: lib/bangumi-sdk-0.0.1-SNAPSHOT.jar lib/td-light-sdk-0.0.1-SN
 APSHOT.jar lib/kotlinx-coroutines-core-jvm-1.6.1.jar lib/kotlin-stdlib-
 jdk8-1.6.21.jar lib/simple-cache-1.2.0-SNAPSHOT.jar lib/redisson-3.17.1
 .jar lib/reflections-0.10.2.jar lib/log4j-core-2.17.1.jar lib/log4j-api
 -2.17.1.jar lib/disruptor-3.4.4.jar lib/jackson-module-kotlin-2.13.1.ja
 r lib/jackson-dataformat-yaml-2.13.1.jar lib/kotlin-stdlib-jdk7-1.6.21.
 jar lib/kotlin-reflect-1.6.21.jar lib/kotlin-stdlib-1.6.21.jar lib/jack
 son-datatype-jdk8-2.13.1.jar lib/jackson-datatype-jsr310-2.13.1.jar lib
 /jackson-databind-2.13.1.jar lib/lettuce-core-6.1.6.RELEASE.jar lib/rea
 ctor-core-3.4.17.jar lib/jackson-core-2.13.1.jar lib/jackson-annotation
 s-2.13.1.jar lib/commons-lang3-3.12.0.jar lib/commons-pool2-2.10.0.jar 
 lib/commons-codec-1.3.jar lib/kryo-5.3.0.jar lib/netty-resolver-dns-4.1
 .74.Final.jar lib/netty-handler-4.1.74.Final.jar lib/netty-codec-dns-4.
 1.74.Final.jar lib/netty-codec-4.1.74.Final.jar lib/netty-transport-4.1
 .74.Final.jar lib/netty-buffer-4.1.74.Final.jar lib/netty-resolver-4.1.
 74.Final.jar lib/netty-common-4.1.74.Final.jar lib/cache-api-1.1.1.jar 
 lib/rxjava-3.0.12.jar lib/reactive-streams-1.0.3.jar lib/jboss-marshall
 ing-river-2.0.11.Final.jar lib/jboss-marshalling-2.0.11.Final.jar lib/s
 lf4j-api-1.7.36.jar lib/byte-buddy-1.11.0.jar lib/jodd-bean-5.1.6.jar l
 ib/javassist-3.28.0-GA.jar lib/jsr305-3.0.2.jar lib/kotlin-stdlib-commo
 n-1.6.21.jar lib/annotations-13.0.jar lib/reflectasm-1.11.9.jar lib/obj
 enesis-3.2.jar lib/minlog-1.3.1.jar lib/snakeyaml-1.28.jar lib/netty-tc
 native-classes-2.0.48.Final.jar lib/jodd-core-5.1.6.jar
```

## 0x03
上面只是比较简单的示例，你还可以对你不经常变动的包分到另一个目录当中去（例如`bangumi-sdk-0.0.1-SNAPSHOT.jar`这个包是我自己写的sdk，会经常变动），这样更加能够利用好缓存构建docker镜像，实际上quarkus是分了4个文件夹。