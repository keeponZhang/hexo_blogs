---
title: Android Service 的启动过程的源码分析
date: 2016-07-28 15:09:58
tags:
---
网上已经有很多篇关于 Service启动源码分析的文章，我今天在这里写下这篇文章，主要是为了在学习的过程中，做个总结，便于以后复习查看。源码基于Android版本号15。 为什么不选用最新的版本23，因为我觉得越新的版本，加入了越多的噪音代码，反而干扰了源码分析。

首先在AndroidManifest.xml中通过 android:process=":remote" 将该Service配置为子进程服务。然后在Activity中通过startService来启动这个Service。

## Activity.startService()

由于是在Activity中调用 startService()，所以我们先看看Activity中startService()是如何调用的。首先借助Android Studio 来看看调用Activity的继承关系图：
![](http://77fzym.com1.z0.glb.clouddn.com/QQ%E5%9B%BE%E7%89%8720160728152244.png)

通过上面的图我们可以看出 Activity 的父类是 ContextThemeWrapper，ContextThemeWrapper 的父类是 ContextWrapper，ContextWrapper 的父类是 Context。通过查找我们发现startService方法的定义是在ContextWrapper类中。我们来看看 ContextWrapper类的startService()方法：

## ContextWrapper.startService()

	@Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }

mBase这里指的是 ContextImpl类，为什么是 ContextImpl类，我为在后面给大家解释。我们再来看看 ContextImpl的startService()方法：

## ContextImpl.startService()

	@Override
    public ComponentName startService(Intent service) {
       
			...
            ComponentName cn = ActivityManagerNative.getDefault().startService(mMainThread.getApplicationThread(), 
			service,service.resolveTypeIfNeeded(getContentResolver()));
			...
           
    }

上面我省略了一下不重要的代码，只关注核心的代码。ActivityManagerNative.getDefault()返回的是 ActivityManagerProxy对象。参数mMainThread.getApplicationThread()返回ApplicationThread 这个对象，ApplicationThread是个Binder对象，这个Binder对象是用来让服务进程和当前应用程序进程通信的。这里的服务进程指的是ActivityManagerNative对象所在的进程。参数service 是一个Intent对象。我们进入到ActivityManagerProxy对象的startService()方法：

## ActivityManagerProxy.startService()

	public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        service.writeToParcel(data, 0);
        data.writeString(resolvedType);
        mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();
        ComponentName res = ComponentName.readFromParcel(reply);
        data.recycle();
        reply.recycle();
        return res;
    }

这里关于Binder通信的代码，我这里不做介绍。这里我们要知道 ActivityManagerProxy是ActivityManagerNative对象的客户端代理对象。我们通过ActivityManagerProxy这个代理对象的操作最终都会转移到 ActivityManagerNative 对象的onTransact方法中。在ActivityManagerNative类的onTransact方法中，有个switch语句

## 在ActivityManagerNative.onTransact()

	  public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
            throws RemoteException {
        switch (code) {
        case START_ACTIVITY_TRANSACTION:
        {

通过方法的第一个参数 code 来找到需要调用的方法，这个 code 是ActivityManagerProxy类中通过 mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0)传递过来的，所以当前code 的值 START_SERVICE_TRANSACTION，我们找到case是 START_SERVICE_TRANSACTION 分支
	
	case START_SERVICE_TRANSACTION: {
            data.enforceInterface(IActivityManager.descriptor);
            IBinder b = data.readStrongBinder();
            IApplicationThread app = ApplicationThreadNative.asInterface(b);
            Intent service = Intent.CREATOR.createFromParcel(data);
            String resolvedType = data.readString();
            ComponentName cn = startService(app, service, resolvedType);
            reply.writeNoException();
            ComponentName.writeToParcel(cn, reply);
            return true;
        }

 IApplicationThread app = ApplicationThreadNative.asInterface(b); 这句代码表示把 b这个Binder对象转化成代理对象。然后调用 startService(app, service, resolvedType)方法，由于ActivityManagerNative是个虚拟类，这个startService方法是在ActivityManagerNative的子类ActivityManagerService中实现的，进入到ActivityManagerService类的startService方法：


## ActivityManagerServic.startService()

	public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType) {
        // Refuse possible leaked file descriptors
        if (service != null && service.hasFileDescriptors() == true) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }

        synchronized(this) {
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            ComponentName res = startServiceLocked(caller, service,
                    resolvedType, callingPid, callingUid);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }

这个方法里调用了 startServiceLocked方法，进入到startServiceLocked方法：


## ActivityManagerServic.startServiceLocked()

	ComponentName startServiceLocked(IApplicationThread caller,
            Intent service, String resolvedType,
            int callingPid, int callingUid) {
        synchronized(this) {
           。。。

            ServiceLookupResult res =
                retrieveServiceLocked(service, resolvedType,
                        callingPid, callingUid);
            。。。

            if (!bringUpServiceLocked(r, service.getFlags(), false)) {
                return new ComponentName("!", "Service process is bad");
            }
            return r.name;

        }
    }


这里只关注 retrieveServiceLocked 和 bringUpServiceLocked 这2个方法。 先看看 retrieveServiceLocked这个方法：

## ActivityManagerServic.retrieveServiceLocked()

  	private ServiceLookupResult retrieveServiceLocked(Intent service,
            String resolvedType, int callingPid, int callingUid) {
        			ServiceRecord r = null;
       
			。。。
        
            r = new ServiceRecord(this, ss, name, filter, sInfo, res);
                  
			。。。
            if (checkComponentPermission(r.permission,
                    callingPid, callingUid, r.appInfo.uid, r.exported)
                    != PackageManager.PERMISSION_GRANTED) {

					if (!r.exported) {
                    Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                            + " from pid=" + callingPid
                            + ", uid=" + callingUid
                            + " that is not exported from uid " + r.appInfo.uid);
                    return new ServiceLookupResult(null, "not exported from uid "
                            + r.appInfo.uid);
                	}
                	Slog.w(TAG, "Permission Denial: Accessing service " + r.name
                        + " from pid=" + callingPid
                        + ", uid=" + callingUid
                        + " requires " + r.permission);
                	return new ServiceLookupResult(null, r.permission);
              
            }  
        }
     
    }


retrieveServiceLocked方法中通过 service这个Intent对象生成ServiceRecord对象。ServiceRecord对象代表了客户端Service在服务端ActivityManagerService中的记录。ServiceRecord对象里面记录着客户端Service所在的进程对象，包名等其它对象。 然后通过传递进来的 callingPid 和 callingUid 来判断当前pid和uid是否有权限来访问当前Service。我们再继续看 bringUpServiceLocked方法：

## ActivityManagerServic.bringUpServiceLocked()

	private final boolean bringUpServiceLocked(ServiceRecord r,
            int intentFlags, boolean whileRestarting) {
        。。。

        final String appName = r.processName;
        ProcessRecord app = getProcessRecordLocked(appName, r.appInfo.uid);
        if (app != null && app.thread != null) {
            try {
                app.addPackage(r.appInfo.packageName);
                realStartServiceLocked(r, app);
                return true;
            } catch (RemoteException e) {
               
            }

        }

        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        if (startProcessLocked(appName, r.appInfo, true, intentFlags,
                "service", r.name, false) == null) {
           。。。
        }
        
        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }
        
        return true;
    
在bringUpServiceLocked方法中先判断当前Service所在的进程是否存在，因为在AndroidManifest.xml中通过 android:process=":remote" 将该Service配置为子进程服务。所以当前Service所在的进程是肯定不存在。接下来通过startProcessLocked这个方法来创建进程来存放需要创建的Service对象。因为等下流程会跳转到客户端中去执行，所在这里需要用mPendingServices这个队列是保存当前的ServiceRecord对象，等待后续继续处理。startProcessLocked方法这里有2个重载，一个是7个参数的，一个是3个参数的。这里先调7个参数的方法，在7个参数的方法中再调用3个参数的方法。这里7个参数方法没有什么这么好看的，这里我们直接进入到3个参数的startProcessLocked方法中：

## ActivityManagerServic.startProcessLocked()

	private final void startProcessLocked(ProcessRecord app,
            String hostingType, String hostingNameStr) {

       		。。。

            // Start the process.  It will either succeed and return a result containing
            // the PID of the new process, or else throw a RuntimeException.
            Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
                    app.processName, uid, uid, gids, debugFlags,
                    app.info.targetSdkVersion, null);

			。。。
           

            synchronized (mPidsSelfLocked) {
                this.mPidsSelfLocked.put(startResult.pid, app);
                。。。
            }
       
    }

这里关于创建进程的内容超过了本博客的范畴，这里我们只要知道在android中所有的java进程都是通过zygot进程fork出来的。Process.start方法的第一个参数是 "android.app.ActivityThread"。由此我们知道新创建的进程的主线程是android.app.ActivityThread类。

	this.mPidsSelfLocked.put(startResult.pid, app); 

mPidsSelfLocked是个Map对象，把新创建的进程的pid作为key，进程对象作为value保存在mPidsSelfLocked中，方便下次查询。           

跟着流程我们进入到android.app.ActivityThread类的入口方法main方法中，这个main方法时新创建的进程的入口。

## ActivityThread.main()

	public static void main(String[] args) {
        
		。。。

        Looper.prepareMainLooper();
        
		。。。

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

     	。。。   

        Looper.loop();
		
	}
     

 

Looper.prepareMainLooper(); 是为当前线程创建Looper对象，这就是为什么在主线程中创建Handler对象不需要首先创建Looper对象，因为这里已经创建了。

然后实例化当前的ActivityThread对象，然后调用attach方法：

## ActivityThread.attach()
	
    private void attach(boolean system) {
        。。。
        if (!system) {
            。。。
            IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
        } else {
           。。。
        }
        
        。。。
    }

因为syster参数在外面传进来是false,所以会执行 if分支里的代码，前面说过  ActivityManagerNative.getDefault() 返回的是ActivityManagerProxy对象，ActivityManagerProxy是服务端ActivityManagerNative在前端的代理对象，对于ActivityManagerProxy对象的方法调用会通过Binder驱动来调用ActivityManagerNative的onTransact方法。省去中间Binder调用的跳转我们直接进入到ActivityManagerService中的attachApplication方法中：

## ActivityManagerService.attachApplication()

	    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
      


里面调用 attachApplicationLocked(thread, callingPid);  

## ActivityManagerService.attachApplicationLocked()

	private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        // Find the application record that is being attached...  either via
        // the pid if we are running in multiple processes, or just pull the
        // next app record if we are emulating process with anonymous threads.
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        }

      。。。

   
    thread.bindApplication(processName, appInfo, providers,
            app.instrumentationClass, profileFile, profileFd, profileAutoStop,
            app.instrumentationArguments, app.instrumentationWatcher, testMode, 
            isRestrictedBackupMode || !normalMode, app.persistent,
            new Configuration(mConfiguration), app.compat, getCommonServicesLocked(),
            mCoreSettingsObserver.getCoreSettingsLocked());
          
          。。。

        // Find any services that should be running in this process...
        if (!badApp && mPendingServices.size() > 0) {
            ServiceRecord sr = null;
            try {
                for (int i=0; i<mPendingServices.size(); i++) {
                    sr = mPendingServices.get(i);
                    if (app.info.uid != sr.appInfo.uid
                            || !processName.equals(sr.processName)) {
                        continue;
                    }

                    mPendingServices.remove(i);
                    i--;
                    realStartServiceLocked(sr, app);
                    
                }
            } catch (Exception e) {
             。。。
            }
        }

     。。。

        return true;
    } 

attachApplicationLocked方法里面调用了2个重要的方法，thread.bindApplication() 和 realStartServiceLocked()。我们先看看thread.bindApplication()方法，这里的 thread是一个实现了IApplicationThread对象，实际上是ApplicationThreadProxy对象，这个对象也是个Binder代理对象，调用它的方法会通过Binder驱动会调用到ApplicationThreadNative对象的onTransact()方法中去。ApplicationThreadNative和ActivityManagerNative一样，也是个抽象类，ApplicationThreadNative方法里面的实现是在ApplicationThread类里面实现的。省去中间Binder调用的步骤，直接进入到ApplicationThread类的bindApplication方法里面,ApplicationThread类是ActivityThread的内部类。

## ApplicationThread.bindApplication()

	public final void bindApplication(String processName,
                ApplicationInfo appInfo, List<ProviderInfo> providers,
                ComponentName instrumentationName, String profileFile,
                ParcelFileDescriptor profileFd, boolean autoStopProfiler,
                Bundle instrumentationArgs, IInstrumentationWatcher instrumentationWatcher,
                int debugMode, boolean isRestrictedBackupMode, boolean persistent,
                Configuration config, CompatibilityInfo compatInfo,
                Map<String, IBinder> services, Bundle coreSettings) {

            if (services != null) {
                // Setup the service cache in the ServiceManager
                ServiceManager.initServiceCache(services);
            }

            setCoreSettings(coreSettings);

            AppBindData data = new AppBindData();
            data.processName = processName;
            data.appInfo = appInfo;
            data.providers = providers;
            data.instrumentationName = instrumentationName;
            data.instrumentationArgs = instrumentationArgs;
            data.instrumentationWatcher = instrumentationWatcher;
            data.debugMode = debugMode;
            data.restrictedBackupMode = isRestrictedBackupMode;
            data.persistent = persistent;
            data.config = config;
            data.compatInfo = compatInfo;
            data.initProfileFile = profileFile;
            data.initProfileFd = profileFd;
            data.initAutoStopProfiler = false;
            queueOrSendMessage(H.BIND_APPLICATION, data);
        }

通过queueOrSendMessage方法，把AppBindData对象放到主线程的消息队列中去，H是定义在ActivityThread类里面的一个Handler对象，进入到H的handleMessage方法中去，找到case是 H.BIND_APPLICATION 的switch分支：


	case BIND_APPLICATION:
                    AppBindData data = (AppBindData)msg.obj;
                    handleBindApplication(data);
                    break;

进入到ActivityThread类的handleBindApplication方法里面去

## ActivityThread.handleBindApplication()


	private void handleBindApplication(AppBindData data) {
     
		。。。

        // If the app is being launched for full backup or restore, bring it up in
        // a restricted environment with the base application class.
        Application app = data.info.makeApplication(data.restrictedBackupMode, null);
        
      
        try {
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            
        }
    }

data.info 是一个LoadedApk对象，进入到LoadedApk类的makeApplication方法

## LoadedApk.makeApplication()

	public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        if (mApplication != null) {
            return mApplication;
        }

        Application app = null;

     	String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            ContextImpl appContext = new ContextImpl();
            appContext.init(this, null, mActivityThread);
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }

        mApplication = app;
       
        return app;
    }

方法的开始处判断当前应用程序的Application对象是否已经创建，如果已经创建了直接返回。appClass代表的是在清单文件中配置的Application对象，如果没有指定就使用默认的android.app.Application类 ，通过getClassLoader()方法获得的ClassLoad来加载Application类。我们返回到上面的handleBindApplication方法，接着看下面的
	
	try {
            mInstrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            
        }

mInstrumentation是Instrumentation对象。对Application和Activity对象的生命周期的控制都是通过这个Instrumentation类来进行的。网上有人说Instrumentation的作用类似一个管家。进入到Instrumentation类的callApplicationOnCreate方法里面

## Instrumentation.callApplicationOnCreate()

	public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }

这个方法非常简单，就一段代码，就是调用app.onCreate()方法。从handleBindApplication这个方法开始的几个流程我们看出这个方法就是创建当前应用进程的Application对象然后调用这个对象的onCreate()方法。**由于Binder通信是阻塞的，所以app.onCreate()方法里不应该做一些耗时的操作**。我们接着返回到ActivityManagerService对象的attachApplicationLocked方法里面，我们已经讲完了thread.bindApplication这个步骤，接着看realStartServiceLocked这个方法的调用过程。realStartServiceLocked(sr, app)方法的第一个参数sr是一个ServiceRecord对象。这个sr来自于mPendingServices这个ArrayList对象里面，前面说过mPendingServices队列保存了待处理的ServiceRecord对象。我们进入到realStartServiceLocked方面里面


## ActivityManagerService.realStartServiceLocked()
 

	    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app) throws RemoteException {
     
            。。。
    
        try {
      
          。。。
       
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo));
       
        } finally {
          
        }
    
    }

这里面最重要的一句话 

	app.thread.scheduleCreateService(r, r.serviceInfo,
                    compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo));

这也是一个Binder通信。通过上面的解释我们知道这个Binder通信最终会调用到ApplicationThread类里面的方法，我们省去中间调用的步骤，直接进入到 ApplicationThread类的scheduleCreateService方法
	
##  ApplicationThread.scheduleCreateService()

	public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo) {
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            queueOrSendMessage(H.CREATE_SERVICE, s);
        }

这里面是一个Handler通信。我们直接进入到H对象handleMessage方法中，找到case是H.CREATE_SERVICE的分支

	case CREATE_SERVICE:
                    handleCreateService((CreateServiceData)msg.obj);
                    break;

这里没调用了 handleCreateService方法，进入到handleCreateService方法


## ActivityThread.handleCreateService()

	private void handleCreateService(CreateServiceData data) {
    
     
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
           
            ContextImpl context = new ContextImpl();
            context.init(packageInfo, null, this);


            context.setOuterContext(service);
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManagerNative.getDefault());
            service.onCreate();
            mServices.put(data.token, service);
            。。。
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }

方法的开始处先通过ClassLoader加载需要创建的Service类，然后通过newInstance()来实例化。然后接着创建 ContextImpl 对象。然后把ContextImpl对象作为参数调用 service.attach 的方法，进入service.attach方法

##Service.attach()

	public final void attach(
            Context context,
            ActivityThread thread, String className, IBinder token,
            Application application, Object activityManager) {
        attachBaseContext(context);
        mThread = thread;           // NOTE:  unused - remove?
        mClassName = className;
        mToken = token;
        mApplication = application;
        mActivityManager = (IActivityManager)activityManager;
        mStartCompatibility = getApplicationInfo().targetSdkVersion < Build.VERSION_CODES.ECLAIR;
                
    }

通过传进来的参数对Service对象进行初始化操作。方法里面调用了 attachBaseContext(context); 方法，我们进入到attachBaseContext方法

## ContextWrapper.attachBaseContext()

	protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }

这个方法的定义是在Service的父类ContextWrapper中定义的。把base参数赋值给mBase这个变量。这个base就是ContextImpl对象。这就解释我在文章一开始处留下的问题。我们接着返回到 handleCreateService方法中，调用了Service的attach方法之后，接着调用了Service的onCreate方法。然后把CreateServiceData参数的token字段作为key，service作为value保存到mServices这个字典中。

到此，基本上已经把Service的启动过程分析完了。
	