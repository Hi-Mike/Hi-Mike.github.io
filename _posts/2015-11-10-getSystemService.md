---
layout: post
title: "getSystemService，没那么简单"
date: 2015-11-10
categories:
---

> 如果不记下，我怕忘记，我怕更加搞不懂逻辑

在平时，我们想得到系统那些服务的时候，只要 **getSystemService(String name)** 便可以取得我们想要的服务了。一句代码，就这么简单。但是如果深究下去，你就会发现，这涉及整个Android体系呀。。。

嗯，没那么简单！思来想去，还是理不清，索性把它记下来，慢慢理一理。

首先，需要指出三个特殊的系统服务， ~~算是UI层次的吧（TAT，错了别打我。。。)~~ （真的错了）：

1. WindowManager，在activity中attach的时候获得，每个activity都会有window，与getWindowManager返回相同
2. SearchManager，调用时，如果没有就创建，每次创建都 ~~从ServiceManager获取~~ ，重新绑定服务？
3. LayoutInflater,cloneInContext？从系统获取以后，clone

```java
	//Activity.java
	@Override
    public Object getSystemService(@ServiceName @NonNull String name) {
        if (getBaseContext() == null) {
            throw new IllegalStateException(
                    "System services not available to Activities before onCreate()");
        }

        if (WINDOW_SERVICE.equals(name)) {
            return mWindowManager;
        } else if (SEARCH_SERVICE.equals(name)) {
            ensureSearchManager();
            return mSearchManager;
        }
        return super.getSystemService(name);
    }
	
	private void ensureSearchManager() {
        if (mSearchManager != null) {
            return;
        }
        mSearchManager = new SearchManager(this, null);
    }
	
	//ContextThemeWrapper.java
	public Object getSystemService(String name) {
        if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            if (mInflater == null) {
                mInflater = LayoutInflater.from(getBaseContext()).cloneInContext(this);
            }
            return mInflater;
        }
        return getBaseContext().getSystemService(name);
    }
```

其他服务就直接从系统获取了，我们知道 **getBaseContext()** 中的mBase就是在activity中attach时得来的，其实就是一个ContextImpl对象。那么，

```java
	@Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
```

SystemServiceRegistry，从名字我们就可以知道一二了，在SystemServiceRegistry中可以看到一个静态代码块：

```java
	static {
        ....
		.....
        registerService(Context.ACCOUNT_SERVICE, AccountManager.class,
                new CachedServiceFetcher<AccountManager>() {
            @Override
            public AccountManager createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(Context.ACCOUNT_SERVICE);
                IAccountManager service = IAccountManager.Stub.asInterface(b);
                return new AccountManager(ctx, service);
            }});

        registerService(Context.ACTIVITY_SERVICE, ActivityManager.class,
                new CachedServiceFetcher<ActivityManager>() {
            @Override
            public ActivityManager createService(ContextImpl ctx) {
                return new ActivityManager(ctx.getOuterContext(), ctx.mMainThread.getHandler());
            }});

        registerService(Context.ALARM_SERVICE, AlarmManager.class,
                new CachedServiceFetcher<AlarmManager>() {
            @Override
            public AlarmManager createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(Context.ALARM_SERVICE);
                IAlarmManager service = IAlarmManager.Stub.asInterface(b);
                return new AlarmManager(service, ctx);
            }});

        registerService(Context.AUDIO_SERVICE, AudioManager.class,
                new CachedServiceFetcher<AudioManager>() {
            @Override
            public AudioManager createService(ContextImpl ctx) {
                return new AudioManager(ctx);
            }});
		........
	｝

	private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }

	public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
```

可以看到我们需要的服务都是会在这个地方注册的，而仔细看又会发现，这个管理类只是将服务获取回来存起来而已。对于大多数的服务，也许只是new一个对象或者获取一个单例等，而有些服务却需要与系统底层的服务绑定。那么系统底层的服务是什么时候注册的呢？

以AlarmManager为例，首先会从ServiceManager获取一个实现了IBinder的对象。在这里，由于能力有限，直接拿前人分析的结论来用。系统启动的时候会创建SystemServer作为一个守护进程，在main()方法最后，会调用initAndLoop()方法:

```java
	public void initAndLoop() {
		......
		alarm = new AlarmManagerService(context);
		ServiceManager.addService(Context.ALARM_SERVICE, alarm);
		......
	}
```

各种服务就是在这个时候创建的，可以看到这里创建了一个AlarmManagerService并添加到ServiceManager中。而上面我们在SystemServiceRegistry中注册时正是从ServiceManager中获取到的各种底层Service。下面看一下ServiceManager中的操作：

```java
	private static IServiceManager getIServiceManager() {
34        if (sServiceManager != null) {
35            return sServiceManager;
36        }
39        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
40        return sServiceManager;
41    }
42
49    public static IBinder getService(String name) {
50        try {
51            IBinder service = sCache.get(name);
52            if (service != null) {
53                return service;
54            } else {
55                return getIServiceManager().getService(name);
56            }
57        } catch (RemoteException e) {
58            Log.e(TAG, "error in getService", e);
59        }
60        return null;
61    }

70    public static void addService(String name, IBinder service) {
71        try {
72            getIServiceManager().addService(name, service, false);
73        } catch (RemoteException e) {
74            Log.e(TAG, "error in addService", e);
75        }
76    }
```

原来ServiceManager也只是一个躯壳，真正的操作是通过ServiceManagerNative获取一个IServiceManager的对象来操作的。查看ServiceManagerNative的源码，如果对android的AIDL及Binder有一定的了解，ServiceManagerNative的代码似乎很是熟悉，其实就是实现了Binder机制中的server和client，只是在建立连接的时候通过BinderInternal得到的是一个native的实现。ServiceManager算是建立起来了，回头看看AlarmManagerService：

```java
class AlarmManagerService extends IAlarmManager.Stub {
	......
}
```

还是Binder机制,这样，只要拿到一个实现了IAlarmManager的接口对象就可以调用方法了:

```java
	registerService(Context.ALARM_SERVICE, AlarmManager.class,
                new CachedServiceFetcher<AlarmManager>() {
            @Override
            public AlarmManager createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(Context.ALARM_SERVICE);
                IAlarmManager service = IAlarmManager.Stub.asInterface(b);
                return new AlarmManager(service, ctx);
            }});
```

在AlarmManager中管理着我们的服务对象。。。

唉，还是很混乱，毕竟很多细节的东西都可以写好多篇文章了，暂且这样，算一个大概的了解吧。。。(PS：老罗的文章有些东西能理解了，也算是一种进步吧)