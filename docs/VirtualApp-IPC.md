#VirtualApp IPC
By Howie.Hxu

#前言
Host & Plugin存在一些问题，首当其冲的就是如何解决进程间通讯/数据共享的问题。  
首先，基于android平台，很自然就会想到Binder，而Binder在java世界的表现主要有：

- Service
- Content Provider
- BroadCast

其中Service和BroadCast都是异步的，只有Content Provider是同步的

>service第一次start的时候需要设置回调  
>BroadCast则更是...  
>Content Provider的同步仅仅是从调用者/被调用者的角度来讲..虽然底层sql仍旧可能是异步

其次，对于IPC来说，AIDL承担了非常重要的角色们，无论怎么样的操作，只要是IPC就必定会省不掉AIDL
所以，因此对于VA来说，年轻的loddy选择了Content Provider + AIDL的方式来实现进程间的同步通讯
>主要还是Host & Client间对于各个service function call的使用

因此本文的

##知识储备
- [AIDL](http://baike.baidu.com/link?url=mY0oLO2lx4WVLcE86U_SXrTLl3bgRpKtcfA_YbhuLMjTfSyqaSU3JUoeoxWnPq8ZqEQzRQQ09GsRktJDCDsRUa)
- [Content Provider](http://blog.csdn.net/luoshengyang/article/details/6950440)
- binder，java层的知识即可

#RTFSC
我们以大家喜闻乐见的startActivity为例来讲述这一次hook & ipc之旅
##startActivity
这里有一个认真尽责的startrActivity

```java
	Intent loadingPageIntent = new Intent(context, LoadingActivity.class);
	loadingPageIntent.putExtra(MODEL_ARGUMENT, model);
	loadingPageIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
	loadingPageIntent.putExtra(ExtraConstants.EXTRA_INTENT, intent);
	context.startActivity(loadingPageIntent);
```

通过之前老罗的文章[Android应用程序启动过程源代码分析](http://blog.csdn.net/luoshengyang/article/details/6689748)，经历了层层外包最终会落点在：Instrumentation::execStartActivity

```java
	public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ......
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
        ......
    }
```

但是当代码走到这里，大家都笑了，根据之前的分析VirtualApp之Hook实现  
我们知道，这里call：ActivityManagerNative.getDefault().startActivity，其实已经被我们hook掉了

##Hook的套路：Hook_StartActivity
首先是动态代理的Handler会捕获，触发invoke（这里是HookObject中的HookHandler）：

###动态代理HookHandler先走

```java
	private class HookHandler implements InvocationHandler {
		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			Hook hook = getHook(method.getName());
			......
				if (hook != null && hook.isEnable()) {
					if (hook.beforeHook(mBaseObject, method, args)) {
						Object res = hook.onHook(mBaseObject, method, args);
						res = hook.afterHook(mBaseObject, method, args, res);
						return res;
					}
				}
				return method.invoke(mBaseObject, args);
			......
		}
	}
```

根据ActivityManagerPatch中的注解@Path，我们拿到的是Hook_StartActivity:

###onHook
这里的class name虽然是Hook_StartActivity,但是getName返回的是“startActivity”。

```java
	class Hook_StartActivity extends Hook_BaseStartActivity {

	@Override
	public String getName() {
		return "startActivity";
	}

	@Override
	public boolean beforeHook(Object who, Method method, Object... args) {
		return super.beforeHook(who, method, args);
	}

	@Override
	public Object onHook(Object who, Method method, Object... args) throws Throwable {
		super.onHook(who, method, args);
		......
		VActRedirectResult result = VActivityManager.getInstance().redirectTargetActivity(req);
		......
		return method.invoke(who, args);
	}
```

##The start of Remote Serivce :x 
在onHook函数中，有一句：
	
```java
	VActivityManager.getInstance().redirectTargetActivity(req);
```

下面我们来逐步剖析一下这个函数的前置事件，在这其中，我们会看到

- AndroidManifest.xml中定义的remote process：x的启动
- 利用content provider的同步调用特性，获取serivce binder接口
- ：x的地位和意义

##VActivityManager.java
继续分析源代码：

```java
	public class VActivityManager {
		private static final VActivityManager sAM = new VActivityManager();
		private IActivityManager service;
		......
		public static VActivityManager getInstance() {
			return sAM;
		}
		public IActivityManager getService() {
			if (service == null) {
				service = IActivityManager.Stub
						.asInterface(ServiceManagerNative.getService(ServiceManagerNative.ACTIVITY_MANAGER));
			}
			return service;
		}
		public VActRedirectResult redirectTargetActivity(VRedirectActRequest request) {
			try {
				return getService().redirectTargetActivity(request);
			} catch (RemoteException e) {
				return VirtualRuntime.crash(e);
			}
		}
		......
	}
```

首先getInstance的返回值是**sAM**
>它是一个静态成员变量：VActivityManager sAM = new VActivityManager();

###VActivityManager::redirectTargetActivity
这次有点类似by pass，先是通过获取service对象，然后调用service对象的接口来做事：
	
```java
	public VActRedirectResult redirectTargetActivity(VRedirectActRequest request) {
	try {
		return getService().redirectTargetActivity(request);
	} catch (RemoteException e) {
		return VirtualRuntime.crash(e);
	}
```

###VActivityManager::getService
来到了重点防守区域，getService：

```java
		public IActivityManager getService() {
			if (service == null) {
				service = IActivityManager.Stub
						.asInterface(ServiceManagerNative.getService(ServiceManagerNative.ACTIVITY_MANAGER));
			}
			return service;
		}
```

对于这个函数，我们可以分为两步来分析：

- ServiceManagerNative.getService(ServiceManagerNative.ACTIVITY_MANAGER)
- IActivityManager.Stub.asInterface（......）

第二步，了解binder的话就可以直接省略了，所以我们研究第一步

###ServiceManagerNative::getService
依旧直接上源码：

```java
	public static IBinder getService(String name) {
		......
		IServiceFetcher fetcher = getServiceFetcher();
		if (fetcher != null) {
			try {
				return fetcher.getService(name);
			} catch (RemoteException e) {
				e.printStackTrace();
			}
		}
		VLog.e(TAG, "GetService(%s) return null.", name);
		return null;
	}
```

然而并没有看出来点什么，而且这边又出现了一个新的类：IServiceFetcher，它是一个AIDL的东西：

```java
	// IServiceFetcher.aidl
	package com.lody.virtual.service.interfaces;
	
	interface IServiceFetcher {
	    IBinder getService(String name);
	    void addService(String name,in IBinder service);
	    void removeService(String name);
	}
```

所以，我们继续研究一下getServiceFetcher

###ServiceManagerNative::getServiceFetcher
这一部分就会出现关于Content Provider的东西了

```java
	public synchronized static IServiceFetcher getServiceFetcher() {
		if (sFetcher == null) {
			Context context = VirtualCore.getCore().getContext();
			Bundle response = new ProviderCaller.Builder(context, SERVICE_CP_AUTH).methodName("@").call();
			if (response != null) {
				IBinder binder = BundleCompat.getBinder(response, ExtraConstants.EXTRA_BINDER);
				linkBinderDied(binder);
				sFetcher = IServiceFetcher.Stub.asInterface(binder);
			}
		}
		return sFetcher;
	}
```

由于是研究需要，我们假定是第一次call到getServiceFetcher，所以很自然，sFetcher是null。

```java
	Context context = VirtualCore.getCore().getContext();
```

这边的content，通过[VirtualApp之Hook实现](http://10.0.12.9:8090/pages/viewpage.action?pageId=11502356)我们知道，其实它是Virtual App Host进程启动的时候application context  
也就是call startup的时候传入参数，所以我们来到：

```java
	Bundle response = new ProviderCaller.Builder(context, SERVICE_CP_AUTH).methodName("@").call();
```

这边是个链式构造，其中：**SERVICE_CP_AUTH = virtual.service.BinderProvider**，所以最后的：
	
```java
	public Bundle call() {
		return ProviderCaller.call(auth, context, methodName, arg, bundle);
	}
```

###ProviderCaller::call
从这边看起来，整个call的命令就是让content provider去做事

```java
	public class ProviderCaller 
	{
		......
		public static Bundle call(String authority, Context context, String methodName, String arg, Bundle bundle) {
			Uri uri = Uri.parse("content://" + authority);
			ContentResolver contentResolver = context.getContentResolver();
			return contentResolver.call(uri, methodName, arg, bundle);
		}
		......
	}
```

其中：

- methodName = “@”
- arg = null
- url = “**virtual.service.BinderProvider**”
- bundle = Builder初始化时候新创建的new Bundle()

###Content Provider
关于content provider的部分，我们直接看一下xml会比较好，注意是lib目录下的AndroidManifest.xml

```xml
    <provider
    	android:name="com.lody.virtual.service.BinderProvider"
    	android:authorities="virtual.service.BinderProvider"
    	android:exported="false"
    	android:process=":x"
	/>
```

从这里我们可以看到:

- authorities字段就是：*SERVICE_CP_AUTH* = **virtual.service.BinderProvider**
- process属性：x，代表了这个provider会是以独立process的形式存在，并且是在主进程之后加“：x”以示区别

对应的class文件是**com.lody.virtual.service.BinderProvider**，所以我们直接来到下一个坑：

##BinderProvider的启动
先看一下类继承关系图：

![](http://7xvfe9.com1.z0.glb.clouddn.com/class.png)

根据我们对content provider的认知（[老罗](http://blog.csdn.net/luoshengyang/article/details/6950440) & [weishu](http://weishu.me/2016/07/12/understand-plugin-framework-content-provider/)），我们知道：  
当我们每次调用Content Provider都会去做acquireProvider，而在acquireProvider中，  
会根据provider是否存活去做唤醒的动作。具体的过程可以看 [weishu](http://weishu.me/2016/07/12/understand-plugin-framework-content-provider/)。
>以上的动作发生在content provider client端，也就是context.getContentResolver()的调用进程

###onCreate
当Content Provider被唤起的时候，我们来看看onCreate函数

```java
	@Override
	public boolean onCreate() {
		Context context = getContext();
		......
		VActivityManagerService.systemReady(context);
		addService(ServiceManagerNative.ACTIVITY_MANAGER, VActivityManagerService.getService());
		......
	}
```

这里跟SystemServer中的写法类似，会先做一下systemReady，然后addService。  
由于我们关注的重点是IPC那一段，所以先略过VA中VActivityManagerService的相关部分。  
直接看addService：

###addService
代码中并没有太多的惊喜：

```java
	private void addService(String name, IBinder service) {
		ServiceCache.addService(name, service);
	}
```

ServiceCache是一个工具类，其中使用类静态变量sCache的key-map来维护各个service

```java
	public class ServiceCache {
	
		private static final Map<String, IBinder> sCache = new HashMap<String, IBinder>(5);
	
		public static void addService(String name, IBinder service) {
			sCache.put(name, service);
		}
		......
	}
```

说到这里，整个provider的初始化就算是做完了，然后我们需要回到正题：

```java
	contentResolver.call(uri, methodName, arg, bundle);
```

##回归Remote Content Provider：Call
我们来到BinderProvider的call

```java
	@Override
	public Bundle call(String method, String arg, Bundle extras) {
		if (method.equals(MethodConstants.INIT_SERVICE)) {
			// Ensure the server process created.
			return null;
		} else {
			Bundle bundle = new Bundle();
			BundleCompat.putBinder(bundle, ExtraConstants.EXTRA_BINDER, mServiceFetcher);
			return bundle;
		}
	}
```

由于method为“@”，所以很自然第一个if判定为false，直接来到：
	
```java
	Bundle bundle = new Bundle();
	BundleCompat.putBinder(bundle, ExtraConstants.EXTRA_BINDER, mServiceFetcher);
	return bundle;
```

###迷之mServiceFetcher
**mServiceFetcher**，是BinderProvider的一个内部成员变量

```java
	public final class BinderProvider extends BaseContentProvider {
		private final ServiceFetcher mServiceFetcher = new ServiceFetcher();
	}
```

对应的class ServiceFetcher，则是BinderProvider的内部类：

```java
	private class ServiceFetcher extends IServiceFetcher.Stub {
		@Override
		public IBinder getService(String name) throws RemoteException {
			if (name != null) {
				return ServiceCache.getService(name);
			}
			return null;
		}
		@Override
		public void addService(String name, IBinder service) throws RemoteException {
			if (name != null && service != null) {
				ServiceCache.addService(name, service);
			}
		}
		@Override
		public void removeService(String name) throws RemoteException {
			if (name != null) {
				ServiceCache.removeService(name);
			}
		}
	}
```

从ServiceFetcher的继承关系以及之前的理解，我们可以猜到，它是一个AIDL的调用，按下不表。

##Remote走完，Local继续
因为remote的部分通过**return bundle**就回去了。  
回到之前的**return contentResolver.call(uri, methodName, arg, bundle);**，我们就来到：  

```java
	public synchronized static IServiceFetcher getServiceFetcher() {
		if (sFetcher == null) {
			Context context = VirtualCore.getCore().getContext();
			Bundle response = new ProviderCaller.Builder(context, SERVICE_CP_AUTH).methodName("@").call();
			if (response != null) {
				IBinder binder = BundleCompat.getBinder(response, ExtraConstants.EXTRA_BINDER);
				linkBinderDied(binder);
				sFetcher = IServiceFetcher.Stub.asInterface(binder);
			}
		}
		return sFetcher;
	}
```

显然，这里的response就是之前remote端回传的Bundle，其中**ExtraConstants.EXTRA_BINDER**字段就是在Remote端：
>BundleCompat.putBinder(bundle, ExtraConstants.EXTRA_BINDER, mServiceFetcher);

ok，看到这里我们就明白了，Remote端回传一个ServiceFetcher的binder对象，在Local端作为IServiceFetcher来使用。  

###退栈到ServiceManagerNative::getService
走到这里，整个getServiceFetcher的流程就走完了，再看一下源码：

```java
	public static IBinder getService(String name) {
		......
		IServiceFetcher fetcher = getServiceFetcher();
		if (fetcher != null) {
			try {
				return fetcher.getService(name);
			} catch (RemoteException e) {
				e.printStackTrace();
			}
		}
		VLog.e(TAG, "GetService(%s) return null.", name);
		return null;
	}
```

这里可以看到是用fetcher.getService，根据之前的了解AIDL的调用又来到了Remote：
	
```java
	private class ServiceFetcher extends IServiceFetcher.Stub {
		@Override
		public IBinder getService(String name) throws RemoteException {
			if (name != null) {
				return ServiceCache.getService(name);
			}
			return null;
		}
		......
	}
```

###VA的ActivityManager
再次退栈：

```java
	public IActivityManager getService() {
		if (service == null) {
			service = IActivityManager.Stub
					.asInterface(ServiceManagerNative.getService(ServiceManagerNative.ACTIVITY_MANAGER));
		}
		return service;
	}
```

可以看到，运行到这里，整个getService的flow就完成了，通过调用：
	
```java
	ServiceManagerNative.getService(ServiceManagerNative.ACTIVITY_MANAGER)
```

我们获取到的是VActivityManagerService的client对象，而对于VActivityManagerService

```java
	public class VActivityManagerService extends IActivityManager.Stub
```

其实又是一个AIDL的类（binder对象），再次通过**IActivityManager.Stub.asInterface**的转换。  
我们就在client端获取了一个ActivityManager（也即VActivityManagerService）在本地的代理对象。  
我们直接操作这个代理对象，就可以无感知的去使用Remote端VActivityManagerService的那些方法。  
因此：

```java
	public VActRedirectResult redirectTargetActivity(VRedirectActRequest request) {
	try {
		return getService().redirectTargetActivity(request);
	} catch (RemoteException e) {
		return VirtualRuntime.crash(e);
	}
```

最终：**getService().redirectTargetActivity(request)**，就会来到：
	
```java
	VActivityManagerService::redirectTargetActivity
```
	
也就是：

```java
	public VActRedirectResult redirectTargetActivity(final VRedirectActRequest request) throws RemoteException {
		synchronized (this) {
			return redirectTargetActivityLocked(request);
		}
	}
```

分析完毕。

#写在最后
纵观整个flow，我们可以明白清楚的看到第一次ServiceFecther的获取是通过Content Provider来实现的。  
话说回来，对于远端对象的获取，其实方法有很多，诸如像startService，binderService都是可以的。  
但在VA中作者另辟蹊径使用Content Provider来做，根据lody & weishu的说法，主要还是想通过content provider  
的同步特性，规避掉使用binderSerivce这样的异步操作，以陷入回调噩梦的境地，也算的上是一种创新之举。

而回顾一下最早的三个点：

- AndroidManifest.xml中定义的remote process：x的启动
- 利用content provider的同步调用特性，获取serivce binder接口
- ：x的地位和意义

其实看完整个flow，我们也清楚地明白：

- remote process虽然是一个content provider的对象，
- content provider的同步特性，帮助我们脱离异步等待的[回调地狱](http://www.jianshu.com/p/a0379bef5913)
- remote：x，实际上它更像是SystemServer中的ServiceManager，通过ServiceCache中的sCache来管理所有的service对象。


