---
title: 基于 JPA 的多租户实现后续
date: 2025-09-21 9:22:00
categories:
  - 编程
tags:
  - JPA
hide: false
comments: true
index_img: https://gamesci.cn/wukong/img/blackmyth_wukong_wallpaper_016.7545da0e.jpg
banner_img: https://gamesci.cn/wukong/img/blackmyth_wukong_wallpaper_016.7545da0e.jpg
---
经过一段时间后，上次构思的实现发现了一些问题
<!--more-->
## 数据库链接的默认值
由于数据库初始化的时候必须给到一个初始配置（X 类型，Y cluster），这个没有太大问题，但是切换数据库的时候需要严谨的判断，判断目标数据库是否已配置，
判断目标数据库是否在在active list中，避免拿到一个default的数据库链接。

## 数据ID的生成
ID如果是用table generator并且预取N>1的id会导致切换数据库后拿的是上一个数据库的id插入，解决办法要么是预取size设置为1，要么改造默认的生成器以适配数据库切换。  

改在默认生成器会比较麻烦，原本的实现写了比较多的情况以及方法的抽象。

---
待续

