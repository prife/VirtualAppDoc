#Hook 
By Howie.Hxu

#前言
作为一个插件框架，VirtualApp自然会使用到大量的hook。  
本文会解析在VA中，那些hook是如何用的，用到了哪些技术，flow是怎样的。

##技术储备
- [java泛型](http://baike.baidu.com/link?url=2lsPAKuaAcZXdiUZcEQuZ4n7mbOffIMIbSym8siODvSdrlfc2x7bTMbaJl0NSEZlMiY-BfeADDg7jk3JAe_ZFa)
- [java反射](http://baike.baidu.com/link?url=oEfSVO_OvzqTwYxiLbqQYD9XxJVAVfRMJtqHJ48UPMRWhG1k8w36tkPDkYFbOHI-o3TGh4fl0MDwewNVoeh4t_)
- 接口编程
- [java动态代理](http://www.cnblogs.com/flyoung2008/archive/2013/08/11/3251148.html)
- [java注解编程](http://baike.baidu.com/link?url=VAfN9tnE90H7zKVUDIWEJoHW8NX-dNK4kJLwwcwc-9p_SSXFMQ2-QfP6nvYu_hmeMQ0b2oM88K-B-NSOTUc58a)  

以上几个知识点希望大家可以在看文章之前先大概了解一下，不然后面看代码会比较累。  
后面的文章部分不会对上述的知识点做二次展开，仅仅以理顺整个Hook flow为主。

#万物起源
##protected void Application::attachBaseContext(Context base)
根据VirtualApp的使用手册，在使用VA libs的时候需要在自己的Application::attachBaseContext中call：
	
	VirtualCore.getCore().startup(base);

这短短的一句话就完整了整个Virutal Core的初始化，也完成了绝大部分Hooker的建立。  
本文的全篇也就是在分析这个function call的其中一小部分......

#RTFSC
##public void VirtualCore::startup(Context context)
如果细究源码，在整个startup的flow中，VA会先去检查当前进程的是否已经启动过；记录一系列的process信息，诸如mainThread，包名，进程名；判定当前进程是否与VA有关，记录process type；然后**会call：PatchManager的function进行hack（hook｜inject）**

	public void startup(Context context) throws Throwable {
		......
			PatchManager patchManager = PatchManager.getInstance();
			patchManager.injectAll();
			patchManager.checkEnv();
		......
	}

其中**PatchManager.getInstance()**就先略过了，我们直接看后面两个。

##public void PatchManager::injectAll()
直接上源码：

	public void injectAll() throws Throwable {
		if (PatchManagerHolder.sInit) {
			throw new IllegalStateException("PatchManager Has been initialized.");
		}
		injectInternal();
		PatchManagerHolder.sInit = true;
	}

这段函数中，先是进行了一个判定是否只初始化过一次；然后是调用了内部private的函数：injectInternal。

###private void PatchManager::injectInternal()
从函数的名字上，我们不难看出来其实它就是在做inject。  
但通过代码的分析，实际上它只是在做一些inject的初始化工作，并没有真正inject到系统。  
查看该函数的源码，大部分都是在做addPatch的动作，而且是通过各种条件进行各种花式add：

	private void injectInternal() throws Throwable {
		addPatch(new ActivityManagerPatch());
		......
	}

抛开具体addPatch的逻辑（是否是MIUI；SDK版本多少；是否是VA Process），我们先看addPatch。   
但是，在addPatch函数生效之前，我们发现它先有一个：**new ActivityManagerPatch()**

##不得不提的ActivityManagerPatch
要弄明白VA Hook，这边的各个XXXXPatch就是不得不提的，不得不看，不得不弄明白的东西了。首先我们看一下它的源文件。

	@Patch({Hook_StartActivities.class,
		......
        Hook_GetServices.class,
	})
	public class ActivityManagerPatch extends PatchObject<IActivityManager, HookObject<IActivityManager>> {
		public static IActivityManager getAMN(){...}
		
		@Override
  		protected HookObject<IActivityManager> initHookObject(){...}

		@Override
  		public void inject(){...}

		@Override
		public boolean isEnvBad(){...}
  	}

开篇就是一个java注解，然后是类定义：
  
- 继承自PatchObject<T, H extends IHookObject<T\>\>
  - 其中 T = IActivityManager, H = HookObject<IActivityManager\>

继续看PatchObject。

###初见泛型：PatchObject

	public abstract class PatchObject<T, H extends IHookObject<T>> implements Injectable {
		protected H hookObject;
		protected T baseObject;
		......
	}

从PatchObject中我们可以清晰地看到泛型的影子，再回溯一下：	

- 其中 T = IActivityManager, H = HookObject<IActivityManager\>

所以简单转化一下就是：
	
	public abstract class PatchObject<IActivityManager, HookObject<IActivityManager>> implements Injectable {
		protected HookObject<IActivityManager> hookObject;
		protected IActivityManager baseObject;
		......
	}

从代码中我们也看到，PatchObject实现了接口**Injectable**，而内部维护的对象**hookObject**显然不是善茬。  
先看一下Injectable的定义，只有两个接口，而显然从之前**ActivityManagerPatch**的定义中，我们发现ActivityManagerPatch已经实现了它们：

	public interface Injectable {
		void inject() throws Throwable;
		boolean isEnvBad();
	}

接着，让我们再来看看**hookObject**的庐山真面目：

###隐藏的：H extends IHookObject<T>  <-> HookObject<IActivityManager\>

	public class HookObject<T> implements IHookObject<T> {
		private T mBaseObject;
		private T mProxyObject;
		private Map<String, Hook> internalHookMapping = new HashMap<String, Hook>();
		......
	}

从上文，我们已经知道T对应的就是IActivityManager，因此这边转化一下就是：

	public class HookObject<IActivityManager> implements IHookObject<IActivityManager> {
		private IActivityManager mBaseObject;
		private IActivityManager mProxyObject;
		private Map<String, Hook> internalHookMapping = new HashMap<String, Hook>();
		......
	}

所以，在HookObject内部，会维护两个成员对象，分别是mBaseObject（原始的），mProxyObject（hook后的），同时还会有一个以各个方法名字String为key，Hook对象为value的HashMap。那么问题又来了，Hook又是什么？

###最底端的Hook
直接看代码：

	public abstract class Hook {
		private boolean enable = true;
		public abstract String getName();
		public boolean beforeHook(Object who, Method method, Object... args) {......}
		public Object onHook(Object who, Method method, Object... args) {......}
		public Object afterHook(Object who, Method method, Object[] args, Object result) {......}
        ......
	}

看起来Hook就是我们真正做hook的时候调用到的地方了，所以，怎么样把Hook添加到**internalHookMapping**，VA又是如何关联起原本的系统对象？

##再次回到：new ActivityManagerPatch()
通过上一章节的阅读，我们已经明白了ActivityManagerPatch是什么。  
使用一张UML的图来总结&复习一下：  
![](http://i.imgur.com/viPGbv2.png)

new的第一步，是进入到ActivityManagerPatch的构造函数，只不过，ActivityManagerPatch并没有写显示构造函数，所以我们来到了它的父类PatchObject的构造函数。

###Construct of PatchObject
PatchObject的构造函数如下：

	public PatchObject() {
		this(null);
	}

	public PatchObject(T baseObject) {
		this.baseObject = baseObject;
		this.hookObject = initHookObject();
		applyHooks();
		afterHookApply(hookObject);
	}

这边又要不厌其烦的说一下：**T = IActivityManager**,因此对于PatchObject中的**baseObject**而言，它是null。   
那么hookObject是什么呢？

###千回百转：PatchObject::initHookObject

	protected abstract H initHookObject();

所以我们需要去看它的子类实现，也即：ActivityManagerPatch::initHookObject

	  @Override
	  protected HookObject<IActivityManager> initHookObject() {
	    return new HookObject<IActivityManager>(getAMN());
	  }

非常简单，狗仔了一个HookObject<IActivityManager\>对象，传入的参数：**getAMN()**

	public static IActivityManager getAMN() {
    	return ActivityManagerNative.getDefault();
	}

但是到这里还没有结束，我们需要进一步看看HookObject的构造函数做了什么。

###Construct of HookObject

	public HookObject(T baseObject, Class<?>... proxyInterfaces) {
		this(baseObject == null ? null : baseObject.getClass().getClassLoader(), baseObject, proxyInterfaces);
	}

	public HookObject(ClassLoader cl, T baseObject, Class<?>... proxyInterfaces) {
		this.mBaseObject = baseObject;
		if (mBaseObject != null) {
			if (proxyInterfaces == null) {
				proxyInterfaces = baseObject.getClass().getInterfaces();
			}
			mProxyObject = (T) Proxy.newProxyInstance(cl, proxyInterfaces, new HookHandler());
		}
	}

	public HookObject(T baseObject) {
		this(baseObject, (Class<?>[]) null);
	}

根据构造函数new HookObject<IActivityManager>(getAMN())，**这里的T = IActivityManager**  
从源代码可以看出，三个构造函数有两个都是在by pass，我们也知道：**getAMN()**拿到的其实就是IActivityManager对象，转换一下：

	public HookObject(ClassLoader cl, IActivityManager baseObject, Class<?>... proxyInterfaces) {
		this.mBaseObject = baseObject;
		if (mBaseObject != null) {
			if (proxyInterfaces == null) {
				proxyInterfaces = baseObject.getClass().getInterfaces();
			}
			mProxyObject = (IActivityManager) Proxy.newProxyInstance(cl, proxyInterfaces, new HookHandler());
		}
	}

分析如下：

- HookObject的mBaseObject，保存了ActivityManagerNative.getDefault()得到的IActivityManager对象
- proxyInterfaces获得了IActivityManager类的所有接口
- HookObject的mProxyObject则通过动态代理技术，代理了proxyInterfaces的那些接口操作，handler为new HookHandler()对象

>注意，这里的HookObject的mBaseObject不为null，而先前PatchObject的baseObject为null。

分析完HookObject的构造函数，我们也就大体上明白了整个Hook大概的flow：

- 获取系统的Interface对象（IActivityManager）
- 通过获取Interface的Interfaces，构造代理对象
- 使用动态代理对象的InovacationHandler实现Hook

###瞄一眼 HookHandler
其中HookHandler是HookObject的内部类：

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

再看看**getHook**

	@SuppressWarnings("unchecked")
	@Override
	public <H extends Hook> H getHook(String name) {
		return (H) internalHookMapping.get(name);
	}

这里有一个小trick，因为H并没有在HookObject的class中定义过，所以使用了：**@SuppressWarnings("unchecked")**
这个函数的理解我们暂时先看字面意义吧，也就是从HookObejct的internalHookMapping中返回了一个Hook对象（子类对象）。  
对于实际存在（已经add到internalHookMapping）的Hook对象，我们进行了一些列的操作

- beforeHook
- onHook
- afterHook 

###initHookObject的本质
	
	this.hookObject = initHookObject();

回到最初的地方，initHookObject其实是

- 利用ActivityManagerNative.getDefault()，获取到IActivityManager
- 利用IActivityManager获得Interfaces
- 利用Interfaces创建动态代理mProxyObject，并绑定HookHandler
- 通过动态代理的HookHandler，在每次invoke的时候去internalHookMapping寻找合适的Hooker实现hook
- hookObject的最终值：HookObject<IActivityManager>

只是目前为止，internalHookMapping还是空的，所以我们继续往下看。

##Next Step：protected void PatchObject::applyHooks()
上一节我们分析完了initHookObject，这一节我们继续看：

	public PatchObject(T baseObject) {
		this.baseObject = baseObject;
		this.hookObject = initHookObject();
		applyHooks();
		afterHookApply(hookObject);
	}

先看看源码：

	protected void applyHooks() {
		......
		Class<? extends PatchObject> clazz = getClass();
		Patch patch = clazz.getAnnotation(Patch.class);
		int version = Build.VERSION.SDK_INT;
		if (patch != null) {
            Class<? extends Hook>[] hookTypes = patch.value();
            for (Class<? extends Hook> hookType : hookTypes) {
                ApiLimit apiLimit = hookType.getAnnotation(ApiLimit.class);
                ......
                if (needToAddHook) {
                    addHook(hookType);
                }
            }

        }
	}

这边用到了java的注解，由于当前的class环境是ActivityManagerPatch，所以很自然的：

	Patch patch = clazz.getAnnotation(Patch.class);

也就是刚才在ActivityManagerPatch.java中看到那一大段：

	@Patch({
		Hook_StartActivities.class,
		......
        Hook_GetServices.class,
	})

先放下**clazz.getAnnotation(Patch.class);**得到的最终返回值，我们看看这些Hook_XXXXX.class是什么呢？

###那些：Hook_XXXXX.class是什么

	public class Hook_GetServices extends Hook {
	    @Override
	    public String getName() {
	        return "getServices";
	    }
	
	    @Override
	    public Object onHook(Object who, Method method, Object... args) throws Throwable {
	        int maxNum = (int) args[0];
	        int flags = (int) args[1];
	        return LocalServiceManager.getInstance().getServices(maxNum, flags);
	    }
	
	    @Override
	    public boolean isEnable() {
	        return isAppProcess();
	    }
	}

很好，它们都是继承自Hook的，那些可以被放到HookObject::internalHookMapping里去的家伙们，再继续往下看。

###Return to ： PatchObject::applyHooks()
	
    Class<? extends Hook>[] hookTypes = patch.value();
    for (Class<? extends Hook> hookType : hookTypes) {
        ApiLimit apiLimit = hookType.getAnnotation(ApiLimit.class);
        ......
        if (needToAddHook) {
            addHook(hookType);
        }
    }

显而易见，patch.value获取的就是@Path注解的那一大段Hook\_XXXXX.class，而for循环中的hookType，其实就对应了每一个Hook\_XXXXX.class，这边原作者非常细致的做了一个apiLimit的判定（同样是利用注解，只是这边写在了Hook\_XXXXX.java中，注解关键字编程了@ApiLimit），更加详细地区分了某个level api是否需要做hook。

在PatchObject::applyHooks()的for循环最后，我们来到了**addHook(hookType);**

###PatchObject.addHook(hookType);

	private void addHook(Class<? extends Hook> hookType) {
		try {
			Constructor<?> constructor = hookType.getDeclaredConstructors()[0];
			if (!constructor.isAccessible()) {
				constructor.setAccessible(true);
			}
			Hook hook;
			if (constructor.getParameterTypes().length == 0) {
				hook = (Hook) constructor.newInstance();
			} else {
				hook = (Hook) constructor.newInstance(this);
			}
			hookObject.addHook(hook);
		} catch (Throwable e) {
			throw new RuntimeException("Unable to instance Hook : " + hookType + " : " + e.getMessage());
		}
	}

	public void addHook(Hook hook) {
		hookObject.addHook(hook);
	}

在PatchObject::addHook，我们看到其实通过反射创建了Hook_XXXX.class的实例对象，然后再调用hookObject.addHook把对应的Hook对象传进去，我们继续来到HookObject.addHook：

###HookObject.addHook

	public void addHook(Hook hook) {
		if (hook != null && !TextUtils.isEmpty(hook.getName())) {
			if (internalHookMapping.containsKey(hook.getName())) {
				VLog.w(TAG, "Hook(%s) from class(%s) have been added, can't add again.", hook.getName(),
						hook.getClass().getName());
				return;
			}
			internalHookMapping.put(hook.getName(), hook);
		}
	}

因此，在这边就完成了HookObject中internalHookMapping对象的add，整个hashmap在这边就被逐步建立。

###applyHooks的本质
- 查找注解Patch，获取Patch List
- 实例化Patch注解中的class对象
- 查看Patch注解中class的注解ApiLimit是否满足sdk版本
- 满足的情况下，把Hook\_XXXX对象添加到HookOject的internalHookMapping对象

##Third Step：protected void PatchObject::afterHookApply(H hookObject)

	protected void afterHookApply(H hookObject) {
	}

空空如也的一段函数，猜测后续会增加一些东西吧。

##new ActivityManagerPatch的完结
还记得为什么会分析PatchObject吗？  
因为ActivityManagerPatch是继承自PatchObject，而ActivityManagerPatch并没有显示的构造函数，所以：

	public PatchObject(T baseObject) {
		this.baseObject = baseObject;
		this.hookObject = initHookObject();
		applyHooks();
		afterHookApply(hookObject);
	}

而经过当菜的分析，我们明白当做完了PatchObject::initHookObject以及PatchObject::applyHooks。  
整个hook的机制就建立起来了，但如何去hook掉系统的关键变量，让整套hook系统可以运作起来？  
让我们继续往下走。

##回到PatchManager

	private void injectInternal() throws Throwable {
		addPatch(new ActivityManagerPatch());
		......
	}

分析**new ActivityManagerPatch()**的由头，是因为在PatchManager有addPatch的动作：

	public final class PatchManager {
		......
		private Map<Class<?>, Injectable> injectableMap = new HashMap<Class<?>, Injectable>(12);
		......
		private void addPatch(Injectable injectable) {
			injectableMap.put(injectable.getClass(), injectable);
		}
		......
	}

所以，当做完了injectInternal，其实就是在injectableMap放入了一个key-value，value：ActivityManagerPatch。  
继续退栈，其实**patchManager.injectAll();**就做完了，代码继续往下走：

	public void startup(Context context) throws Throwable {
		......
			PatchManager patchManager = PatchManager.getInstance();
			patchManager.injectAll();
			patchManager.checkEnv();
		......
	}

###public void PatchManager::checkEnv()

	public void checkEnv() throws Throwable {
		for (Injectable injectable : injectableMap.values()) {
			if (injectable.isEnvBad()) {
				injectable.inject();
			}
		}
	}

###Injectable::inject
通过分析之前的flow我们可以知道，这边的injectableMap中包含了ActivityManagerPatch的实例。  
而ActivityManagerPatch是实现了接口Injectable。因此我们直接来到：ActivityManagerPatch::inject

	public void inject() throws Throwable {
	    Field f_gDefault = ActivityManagerNative.class.getDeclaredField("gDefault");
	    if (!f_gDefault.isAccessible()) {
	      f_gDefault.setAccessible(true);
	    }
	    if (f_gDefault.getType() == IActivityManager.class) {
	      f_gDefault.set(null, getHookObject().getProxyObject());
	
	    } else if (f_gDefault.getType() == Singleton.class) {
	
	      Singleton gDefault = (Singleton) f_gDefault.get(null);
	      Field f_mInstance = Singleton.class.getDeclaredField("mInstance");
	      if (!f_mInstance.isAccessible()) {
	        f_mInstance.setAccessible(true);
	      }
	      f_mInstance.set(gDefault, getHookObject().getProxyObject());
	    } else {
	      ....
	    }
	
	    HookBinder<IActivityManager> hookAMBinder = new HookBinder<IActivityManager>() {
	      @Override
	      protected IBinder queryBaseBinder() {
	        return ServiceManager.getService(Context.ACTIVITY_SERVICE);
	      }
	
	      @Override
	      protected IActivityManager createInterface(IBinder baseBinder) {
	        return getHookObject().getProxyObject();
	      }
	    };
	    hookAMBinder.injectService(Context.ACTIVITY_SERVICE);
	  }

这部分的代码从上到下分为两部分:

- 第一部分是hook掉ActivityManagerNative对象的gDefault。
- 第二部分是hook掉ServiceManager对象的sCache

##Round One：Hook -> ActivityManagerNative::gDefault

	Field f_gDefault = ActivityManagerNative.class.getDeclaredField("gDefault");
    if (!f_gDefault.isAccessible()) {
      f_gDefault.setAccessible(true);
    }
    if (f_gDefault.getType() == IActivityManager.class) {
      f_gDefault.set(null, getHookObject().getProxyObject());

    } else if (f_gDefault.getType() == Singleton.class) {

      Singleton gDefault = (Singleton) f_gDefault.get(null);
      Field f_mInstance = Singleton.class.getDeclaredField("mInstance");
      if (!f_mInstance.isAccessible()) {
        f_mInstance.setAccessible(true);
      }
      f_mInstance.set(gDefault, getHookObject().getProxyObject());
    } 

这一部分非常简单，可以通过阅读AOSP的源码，对应不同的api level，gDefault会是不同的实例，因此做了if...else...来区分。  
最终都是通过调用：
	
	XXXXX.set(AAAAA, getHookObject().getProxyObject());

把getHookObject().getProxyObject()对象赋进去，而这一步让ActivityManagerNative与ActivityManagerPatch建立的联系。

##Round Two：Hook -> getService from ServiceManager
	
	 HookBinder<IActivityManager> hookAMBinder = new HookBinder<IActivityManager>() {
	  @Override
	  protected IBinder queryBaseBinder() {
	    return ServiceManager.getService(Context.ACTIVITY_SERVICE);
	  }
	
	  @Override
	  protected IActivityManager createInterface(IBinder baseBinder) {
	    return getHookObject().getProxyObject();
	  }
	};
	hookAMBinder.injectService(Context.ACTIVITY_SERVICE);

这一段代码比较奥妙，刚才我们明明已经建立好ActivityManagerNative与ActivityManagerPatch，为什么这边还要来一次hook呢？  
且看HookBinder。

###HookBinder.java

	public abstract class HookBinder<Interface extends IInterface> implements IHookObject<Interface>, IBinder {
		private static Map<String, IBinder> sCache;
			static {
				try {
					Class.forName(ServiceManager.class.getName());
					Field f_sCache = ServiceManager.class.getDeclaredField("sCache");
					f_sCache.setAccessible(true);
					sCache = (Map<String, IBinder>) f_sCache.get(null);
				} catch (Throwable e) {
					// 不考虑
				}
			}
		private IBinder baseBinder;
		private Interface mBaseObject;
		private Interface mProxyObject;
		......
		public HookBinder() {
			bind();
		}
		......
		public final void bind() {
			this.baseBinder = queryBaseBinder();
			if (baseBinder != null) {
				this.mBaseObject = createInterface(baseBinder);
				mProxyObject = (Interface) Proxy.newProxyInstance(mBaseObject.getClass().getClassLoader(),
						mBaseObject.getClass().getInterfaces(), new HookHandler());
			}
		}
		......
	}

###static block
从代码中，我们看到static代码块把ServiceManager的sCache对象赋给了HookBinder的静态成员变量：sCache（其实就是做了关联），也就是说后续操作HookBinder对象的sCache，就改变ServiceManager的sCache。

###construct of HookBinder
在HookBinder的构造函数中，我们直接调用了bind函数，而在bind函数中：

	public final void bind() {
		this.baseBinder = queryBaseBinder();
		if (baseBinder != null) {
			this.mBaseObject = createInterface(baseBinder);
			mProxyObject = (Interface) Proxy.newProxyInstance(mBaseObject.getClass().getClassLoader(),
					mBaseObject.getClass().getInterfaces(), new HookHandler());
		}
	}

看起来又是老一套的思路，使用动态代理来完成hook，不过要注意，这边的HookHandler是HookBinder的内部类了。  
而queryBaseBinder，createInterface这两个函数，都是abstract：

	protected abstract IBinder queryBaseBinder();
	protected abstract Interface createInterface(IBinder baseBinder);

所以让我们回到new HookBinder的地方：
	
	HookBinder<IActivityManager> hookAMBinder = new HookBinder<IActivityManager>() {
	  @Override
	  protected IBinder queryBaseBinder() {
	    return ServiceManager.getService(Context.ACTIVITY_SERVICE);
	  }
	
	  @Override
	  protected IActivityManager createInterface(IBinder baseBinder) {
	    return getHookObject().getProxyObject();
	  }
	};

下面我们再来看bind就很简单了：

###HookBinder::bind
直接替换一下：

	public final void bind() {
		this.baseBinder = ServiceManager.getService(Context.ACTIVITY_SERVICE);
		if (baseBinder != null) {
			this.mBaseObject = getHookObject().getProxyObject();
			mProxyObject = (Interface) Proxy.newProxyInstance(mBaseObject.getClass().getClassLoader(),
					mBaseObject.getClass().getInterfaces(), new HookHandler());
		}
	}

所以做完这一步，HookBinder内部的成员变量：

- baseBinder是系统中真实的service对象，如假包换没有更改过的：ServiceManager.getService(Context.ACTIVITY_SERVICE)
- mBaseObject是我们在HookObject通过动态代理建立出来的Proxy对象
- mProxyObject，通过mBaseObject对象建立出来的动态代理对象

再来看看HookBinder的内部类HookHandler

###HookBinder::HookHandler

	private class HookHandler implements InvocationHandler {

		@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			Hook hook = getHook(method.getName());
			try {
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

如之前在HookObject中所述，这边getHook也会去查询HookBinder内部的internalHookMapping对象，但是很显然internalHookMapping并没有addHook，因此这边查到的hook为null，所以就会走到：

	return method.invoke(mBaseObject, args);

而进入到这边，mBaseObject又是我们之前HookObject通过代理对象创建出来的Proxy对象，自然就会落到HookObject的HookHander的invoke函数，所以最终还是会调用到Hook_XXXXX的那些实例里去。

###最后的收尾：hookAMBinder.injectService(Context.ACTIVITY_SERVICE);

	public void injectService(String name) throws Throwable {
		if (sCache != null) {
			sCache.remove(name);
			sCache.put(name, this);
		} else {
			throw new IllegalStateException("ServiceManager is invisible.");
		}
	}

直接撸掉sCache中的元素，对于ServiceManager来说，从此以后sCache只有HookBinder<IActivityManager>
>当然了，HookBinder<IActivityManager>也是实现了IBinder接口  
>class HookBinder<Interface extends IInterface\> implements IHookObject<Interface>, **IBinder**

##为什么AMS要hook两次
因为AMS的特殊性，它会存在与ActivityManagerNative.gDefault中，也会存在于ServiceManager.sCache。  
对比其他的Service，都只有Round Two而已。