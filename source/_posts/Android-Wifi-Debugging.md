---
title: Android 无线真机调试
date: 2020-06-11 12:12:50
tags: [Android,Debug,WIFI]
---



### 前言

对于手机前端开发，总会使用到各种设备，日常常用开发机器电脑，手机这些，配合开发调试过程就要用到各种线，对于 Android 手机开发的，就要用到 Micro USB 和 Type C 新老手机对应的手机线。特别是机型适配的时候，桌子上是这样（暂无无图。。。）的凌乱，很早前听说过无线调试（WIFI Debugging），也尝试过很多对应 IDE 无线调试插件，总是有各种各样的问题，于是就继续凌乱中。

最近无意间看官方文档时看到有命令行工具 adb 章节中关于 Wear OS 设备调试的内容，看到了[通过 WLAN 连接到设备](<https://developer.android.google.cn/studio/command-line/adb#wireless>)章节说到无线调试的相关内容，于是，你知道的，我是一个整洁的人，正如一手出来的整洁代码（大脸不惭），于是我就抱着试一试的态度开始这个简短的过程。





### 开始步骤



##### 0. 保证 Android 设备和 IDE 所在电脑主机都连到同一个 WLAN 网络。

注意项：并非所有接入点都可以。可能需要配置防火墙支持 adb 接入点。如果 Wear OS 设备，要关闭与该设备配对的手机上的蓝牙。

##### 1.使用 USB 数据线将设备链接到主机。

（不是说好无线调试呢，怎么连上线了呢，可能手机快没电了（假装说服）。来都来了连一下啦）


##### 2. 设置目标设备以监听端口 5555 上的 TCP/IP 连接。

```shell
adb tcpip 5555
```

##### 3. 拔掉连接的 USB 数据线。
（拔掉，拔掉，通通拔掉）

##### 4. 找到 Android 设备的 IP 地址。

一般在`手机 - 设置 - 关于` 中相应的位置可以找到本机的 ip 地址。


##### 5. 通过 IP 地址连接到设备

```shell
adb connect device_ip_address
```

然后可以通过日常的 adb 命令确认是否连接上设备：

```shell
adb devices
```

现在如果一切顺利，应该可以在 Android Studio 的 Logcat 的窗口看到手机的日志打印信息了。
<!-- more -->
中间如果连接断开了，确认设备和电脑在一个网段内，然后再次执行连接步骤命令 `adb connect`.
如果还是不行，就杀掉重新再来，执行 adb 关闭命令：

```shell
adb kill-server
```

然后重新执行以上步骤。

大致如上步骤了，纯粹记录一下这个操作过程。当然你看到这（内心 OS:这和官网没啥区别啊），是的，和官网没啥区别，手工打印一下:).




> 《题都城南庄》	崔护

> 去年今日此门中，人面桃花相映红。

> 人面不知何处去，桃花依旧笑春风。



### 参考链接：

[崔护](https://zh.wikipedia.org/zh-hans/%E5%B4%94%E6%8A%A4)

[Android 调试桥 (adb)](https://developer.android.google.cn/studio/command-line/adb#wireless)

[调试 Wear OS 应用](<https://developer.android.google.cn/training/wearables/apps/debugging>)





全文完:)




