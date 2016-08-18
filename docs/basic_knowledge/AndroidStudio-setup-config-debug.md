# 参考资料

**官方网站**
- https://github.com/googlesamples/android-ndk, Android NDK官方参考代码
- http://tools.android.com/tech-docs， Android Studio技术文档
- http://code.google.com/p/android/issues/list， Android问题列表
- http://android-developers.blogspot.com/， Android官方博客

# Android Studio 安装

## 安装JDK

Android Studio不建议使用openjdk（因为其UI性能较差），因此从Oracle官网下载JDK，下载网址：
- [JDK8 官方下载页面](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
笔者选择的是jdk-8uXXX-linux-x64.tar.gz。

**下载后解压即可，不必配置`JAVA_HOME`等环境变量**

## 安装AS

去developmen.android官方下载最新版Android Studio，并解压，假定目录：`~/software/android-studio`
修改该目录下的bin/studio.sh文件，在开头添加`STUDIO_JDK`变量手动指定JDK。

```shell
#!/bin/sh
#
# ---------------------------------------------------------------------
# Android Studio startup script.
# ---------------------------------------------------------------------
#
STUDIO_JDK="$HOME/software/jdk1.8.0_91/"
```

说明：这样系统可以存在多个JDK版本而互不干扰。如笔者机器上还安装了`openjdk-7-jdk`，用于编译Android系统代码。

## 配置Android Studio主题和插件

### 插件
笔者使用黑色系，选择黑色系，在settings菜单中，选择plugins，安装如下插件：
- Material Theme UI，黑色主题插件，配置完成后新增菜单 tools | Material Theme，选择darker应用。
- PlantUML，UML绘图插件，使用特殊的标记语法绘制各种类型的UML类图、时序图，并可以生成svg/eps/jpg/png等图片。[语法参考](http://plantuml.com/sequence.html)。

PlantUML绘制时序图效果参考：[VirtualAppDoc](https://github.com/prife/VirtualAppDoc) 项目。

## 使用Android Studio编译NDK

**官方文档**
- [Experimental Plugin User Guide](http://tools.android.com/tech-docs/new-build-system/gradle-experimental)，出自上面**Android Studio技术文档**网站。

**网友博客**
- [使用AndroidStudio进行NDK开发](https://dailyios.com/article/Using_Android_Studio_for_NDK_development.html)

# Android Studio调试技巧

## 调试有源码程序

### 调试应用

首先在合适的地方设置断点，Android Studio中支持多种类型断点，包括
- 普通断点
- 方法断点
- 条件断点
- ...

关于断点调试，强烈推荐阅读 [Android Studio你不知道的调试技巧](http://weishu.me/2015/12/21/android-studio-debug-tips-you-may-not-know/)。

现在开始调试，有如下两种启动方法

- 从新启动应用，选择菜单 `run` -> `Debug 'app'`，或者点击工具栏按钮。
- 调试已经运行的应用，可以使用`attach`功能，选择菜单 `run` -> `attach debugger to android process`，或者点击工具栏按钮，如果连接手机，则会弹出进程列表，选择被调试的进程附着即可。

### 同时调试多个进程

Android Studio的调试功能非常强大，同时支持多个调试上下文，可以attach多个进程，每attach一个进程，都会生成在AS底部的debug窗口打开一个新的`Android debug`标签页。每个标签页都有独立的调试上下文，分别对应一个进程。AS可以自动在多个debugger之间切换。

### 调试Android框架层

首先下载Android官方源代码，具体方法请参考`Android代码下载编译并刷入Nexus6`。

参考 [如何使用Android Studio开发/调试Android源码](http://www.cnblogs.com/Lefter/p/4176991.html)

上面这篇文章配置较为繁琐，实际只需要第三步和第五步即可，其他步骤不需要。之后就可以使用attach方式附加被调试的进程。

### 使用 Debug.waitForDebugger 调试

可以应用于以下场景:
- 被调试程序运行时会创建一个新进程，该进程很快执行完毕，来不及触发并attach
- 被调试程序运行时会启动一个新进程，但是想要调试触发动作之前代码逻辑

此时，可以使用Android提供的调试机制，
```java
Debug.waitForDebugger();
```
> Wait until a debugger attaches. As soon as the debugger attaches, this returns, so you will need to place a breakpoint after the waitForDebugger() call if you want to start tracing immediately.

参考: https://developer.android.com/reference/android/os/Debug.html

该函数会等待调试器attach（附着进程）。该函数在调试器attach后立刻返回，因此如果想开始调试，那么需要在`waitForDebugger`后设置断点。

## 调试无源代码程序

### 使用AndBug

项目地址：https://github.com/swdunlop/AndBug

### Android Studio调试Smali

说明：以下部分摘自[Smalidea + AndroidStudio 调试 smali 代码](https://www.zybuluo.com/oro-oro/note/167401)，略有补充。

#### 1.准备

Android Studio 
http://tools.android.com/download/studio

smalidea-v0.03.zip 
https://bitbucket.org/JesusFreke/smali/downloads 
https://github.com/JesusFreke/smali/wiki/smalidea

#### 2.安装插件

Setting -> Plugin -> Install plugin from disk...

#### 3.反编译

```
$ java -jar ~/software/smali/baksmali-2.1.2.jar debug.apk -o debug/src
```

#### 4.导入和配置项目

- Import Project... -> Create project from existing sources
- 将Project（ALT+1）里面默认的Android视图切换为Project视图，将src设置为Sources Root.
- Project Structure（Ctrl+Shift+ALT+S），Project SDK设置为Android API 10 Platform
- 远程调试配置，Run -> Edit Configuration进入Run/Dubug Configurations；Add New Configuration(+符号) -> Remote，将5005端口，修改为8700端口。

#### 5.安装和配置调试应用

- adb install debug.apk（或者用其他方式） 
- 开发者选项，选择调试应用，等待调试器打勾（英文版本为 Developer页面，打开options wait for debugger选项，Select debug app选择要被调试的应用）。
- 启动应用，应用将挂起，等待调试器连接。

**补充说明**：也可是使用`am`命令配合`-D`参数启动应用，与上面的效果相同。
```bash
$ adb shell am start -D -n com.droi.helloinstantrun/.MainActivity
```

#### 6.连接调试

打开monitor（ddms），会发现有红色蜘蛛的进程，选中后，会显示为xxxx/8700。 
启动调试（刚才配置好），应用会启动起来，而Console视图会显示Connected to the target VM, address: 'localhost:8700', transport: 'socket'。 
断点就根据实际情况设置。

**补充说明**：
1. 根据参考文献的说法，设置端口为8700，打开ddms/monitor会自动完成端口转发；也可以使用adb命令手动设置端口转发，命令为`adb forward tcp:8700 jdwp:447`，其中8700为端口号，447为待调试进程的PID。
1. 这样创建的工程不完整，无法在AS直接打开`Android Device Monitor`（该按钮是灰色的），那么可以另外打开一个完整的AS工程，并打开该菜单，或者在android sdk 的`tools`目录下手动执行ddms或者monitor命令。

参考：
- [Android Studio 调试 smali 代码](http://kiya.space/2016/08/07/use-studio-debug-smali/)
- [使用android studio调试smali代码](http://www.cnblogs.com/gordon0918/p/5570811.html)
- [AndroidStudio+ideasmali动态调试smali汇编](http://www.cnblogs.com/lanrenxinxin/p/4891424.html)
