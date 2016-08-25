# VirtualAppDoc

说明：本工程是[VirtualApp](https://github.com/asLody/VirtualApp)项目的非官方文档。

理解VirtualApp代码的过程中，对我帮助很大两组系列文章：

- https://github.com/tiann/understand-plugin-framework
- http://gityuan.com/

PS.还有很多文章无法一一列举, 谨表谢忱。

## Server Process 启动流程

**getService**
![VirtualCoreGetSerivce](https://rawgit.com/prife/VirtualAppDoc/master/pngs/VirtualCoreGetSerivce.svg)

**Binder Provider调用过程**
![VABinderProvider](https://rawgit.com/prife/VirtualAppDoc/master/pngs/VABinderProvider.svg)

**VirtualActivityManagerService启动流程**
![VAMS](https://rawgit.com/prife/VirtualAppDoc/master/pngs/VAMS.svg)

## startActivity 流程

![VAStartActivity](https://rawgit.com/prife/VirtualAppDoc/master/pngs/VAStartActivity.svg)

![VStubContentProvider](https://rawgit.com/prife/VirtualAppDoc/master/pngs/VStubContentProvider.svg)

## install流程

![VAInstall](https://rawgit.com/prife/VirtualAppDoc/master/pngs/VAInstall.svg)

## Broadcast流程

注意：VA对client的xml中定义的receiver（静态广播接收器）做了处理，详细参考VAInstall流程图。

### 标准Broadcast发送/接收

![标准Broadcast发送/接收](https://rawgit.com/prife/VirtualAppDoc/master/pngs/Broadcast.svg)

### VA中Broadcast发送/接收

![VA中Broadcast发送/接收](https://rawgit.com/prife/VirtualAppDoc/master/pngs/VABroadcast.svg)

### VA中动态Broadcast注册

动态注册Broadcast，最终都会调用ActivityManagerNatvie#registerReceiver方法，因此VA中hook了这个方法然后改造IntentFilter的ACTION字段，具体改造方法与静态广播接收器的方式相同。然后创建一个新的`IIntentReceiver$Stub`对象，传递给AMS。也就是所谓静态代理方式。

代码：RegisterReceiver.java

请参考下面类结构图。

![类结构图](https://rawgit.com/prife/VirtualAppDoc/master/pngs/BroadcastClass.svg)

**PS**

[添加SVG图片的方法](http://stackoverflow.com/questions/13808020/include-an-svg-hosted-on-github-in-markdown)
