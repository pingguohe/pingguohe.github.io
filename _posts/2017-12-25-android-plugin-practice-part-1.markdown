---

layout: post
title: Android插件化实践(1)--动态代理
author: zhaolin1230

---

## 前言

作为一个android开发者，一定都知道每个activity都需要在AndroidManifest.xml中显示的声明一下，否则在启动的activity的时候就会抛出ActivityNotFoundException的异常。那么真的就没有办法去启动一个没有声明的activity吗？一切答案都在源码里，来让我们从源码看起。

## activity启动过程

想要知道能不能启动一个不在manifest中注册的Activity，一切答案都在源码里，我们先来看一下activity启动的过程，下面以Android7.1.2的代码为例。
在启动activity时会调用startActivity，首先我们来看Activity中的startActivity看起，代码在frameworks/base/core/java/android/app/Activity.java中。

```java
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
```

之后有调用了两个参数的startActivity()，又经过一连串的调用，调用到了startActivityForResult()

```java
ublic void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
```

其中真正启动activity的方法是通过Instrumentation中的execStartActivity()方法启动的。mInstrumentation是Activity中的一个成员变量，源码在frameworks/base/core/java/android/app/Instrumentation.java

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

下面是整个流程中比较关键的方法

```java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    ...

    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

先看checkStartActivityResult()这个方法，这里会检测activity启动的各种状态，其中就包括了"have you declared this activity in your AndroidManifest.xml?"，没有在AndroidManifest.xml"注册activity的异常。

```java
public static void checkStartActivityResult(int res, Object intent) {
    if (res >= ActivityManager.START_SUCCESS) {
        return;
    }

    switch (res) {
        case ActivityManager.START_INTENT_NOT_RESOLVED:
        case ActivityManager.START_CLASS_NOT_FOUND:
            if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                throw new ActivityNotFoundException(
                        "Unable to find explicit activity class "
                        + ((Intent)intent).getComponent().toShortString()
                        + "; have you declared this activity in your AndroidManifest.xml?");
            throw new ActivityNotFoundException(
                    "No Activity found to handle " + intent);
        case ActivityManager.START_PERMISSION_DENIED:
            throw new SecurityException("Not allowed to start activity "
                    + intent);
        case ActivityManager.START_FORWARD_AND_REQUEST_CONFLICT:
            throw new AndroidRuntimeException(
                    "FORWARD_RESULT_FLAG used while also requesting a result");
        case ActivityManager.START_NOT_ACTIVITY:
            throw new IllegalArgumentException(
                    "PendingIntent is not an activity");
        case ActivityManager.START_NOT_VOICE_COMPATIBLE:
            throw new SecurityException(
                    "Starting under voice control not allowed for: " + intent);
        case ActivityManager.START_VOICE_NOT_ACTIVE_SESSION:
            throw new IllegalStateException(
                    "Session calling startVoiceActivity does not match active session");
        case ActivityManager.START_VOICE_HIDDEN_SESSION:
            throw new IllegalStateException(
                    "Cannot start voice activity on a hidden session");
        case ActivityManager.START_CANCELED:
            throw new AndroidRuntimeException("Activity could not be started for "
                    + intent);
        default:
            throw new AndroidRuntimeException("Unknown error code "
                    + res + " when starting " + intent);
    }
}
```

其中启动activity的方法是ActivityManagerNative.getDefault().startActivity()。ActivityManagerNative继承了Binder，同时实现了IActivityManager接口，启动activity用到了Android中binder机制，这里先不做具体讨论，代码在frameworks/base/core/java/android/app/ActivityManagerNative.java。先来看getDefault()，获取的是一个IActivityManager单例(注: 我只看了Android6.0到7.1.2的代码是这样实现的，在Android8.x上，这里的实现发生了变化，需要进行适配)。

```java
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

再看单例里做了什么，获取了系统的ActivityManagerService(AMS)，ActivityManagerNative实际上就是ActivityManagerService(AMS)这个远程对象的Binder代理对象

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};
```

到这里，就是整个activity启动的流程了，也找到了ActivityNotFoundException的原因，那么到底有没有办法启动一个没有在Android.xml中声明的activity呢？

## hook分析

还是从gDefault这个单例看起，有没有办法把这个单例替换掉，自己实现startActivity方法来绕过系统对activity的校验呢？同时我们看到IActivityManager是一个接口，那么是不是有什么办法可以动态修改这个接口的实现呢？熟悉Java的同学一定会想到**动态代理**。我们先来实现一个InvocationHandler，代码如下

```java
public class HookActivityHandler implements InvocationHandler {

    private static final String TAG = "HookActivityHandler";

    private Object mBase;

    public HookActivityHandler(Object base) {
        this.mBase = base;
    }

    @Override
    public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
        if ("startActivity".equalsIgnoreCase(method.getName())) {
            Log.d(TAG, "invoke: startActivity");
        }
        return method.invoke(mBase, objects);
    }
}
```

接着，我们需要通过反射将ActivityManagerNative中的gDefault中的gDefault换成我们自己的实现，代码如下

```java
public static final void hookActivityManagerService(ClassLoader classLoader) {
    try {
        Class<?> activityManagerNativeClass = Class.forName("android.app.ActivityManagerNative");
        if (activityManagerNativeClass == null) {
            return;
        }
        Field gDefaultField = activityManagerNativeClass.getDeclaredField("gDefault");
        if (gDefaultField == null) {
            return;
        }
        gDefaultField.setAccessible(true);
        Object gDefault = gDefaultField.get(null);

        Class<?> singleton = Class.forName("android.util.Singleton");
        if (singleton == null) {
            return;
        }

        Field mInstanceField = singleton.getDeclaredField("mInstance");
        if (mInstanceField == null) {
            return;
        }
        mInstanceField.setAccessible(true);

        Object activityManager = mInstanceField.get(gDefault);
        Class<?> activityManagerInterface = Class.forName("android.app.IActivityManager");
        Object proxy = Proxy.newProxyInstance(classLoader,
                new Class[] {activityManagerInterface}, new HookActivityHandler(activityManager));
        mInstanceField.set(gDefault, proxy);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

然后新建一个正常activity看看能否正常调用到hook的方法中，输出日志如下。

```
08-15 23:10:10.476 13208-13208/com.test.hookactivity D/HookActivityHandler: invoke: startActivity 
```

可以看到，已经成功的将startActivity()方法hook住了，接下来就可以在startActivity的时候做些事情了。在启动activity的时候，会对activity做校验，那么我们能不能用一个注册在AndroidMenifest.xml中的activity作为桥梁，在校验的时候用这个存在的activity，校验过后再换会真正想要的activity。再回到HookActivityHandler中，只看invoke方法。
在startActivity之前先将目标intent取出来并缓存起来，将intent换成已经存在的PlaceHolderActivity，这样校验的时候就可以绕过系统对activity的校验。大家可以试着运行一次代码，并不会抛出ActivityNotFoundException的异常，验证了之前的想法。

```java
@Override
public Object invoke(Object o, Method method, Object[] objects) throws Throwable {
    if ("startActivity".equalsIgnoreCase(method.getName())) {
        Log.d(TAG, "invoke: startActivity");
        Intent rawIntent = null;
        int index = 0;

        for (int i = 0; i < objects.length; i++) {
            if (objects[i] instanceof Intent) {
                index = i;
                break;
            }
        }

        rawIntent = (Intent) objects[index];

        Intent newIntent = new Intent();
        String targetPackageName = "com.test.hookactivity";

        ComponentName componentName = new ComponentName(targetPackageName,
                PlaceHolderActivity.class.getCanonicalName());
        newIntent.setComponent(componentName);
        newIntent.putExtra("extra_target_intent", rawIntent);

        objects[index] = newIntent;
        return method.invoke(mBase, objects);
    }

    return method.invoke(mBase, objects);
}
```

那么绕过校验之后怎样恢复到我们想要的activity呢？从源码中可以看到，在启动activity时最后会通过ActivityThread中的handleMessage()方法，代码如下

```java
public void handleMessage(Message msg) {
    if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
    switch (msg.what) {
        case LAUNCH_ACTIVITY: {
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
            final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

            r.packageInfo = getPackageInfoNoCheck(
                    r.activityInfo.applicationInfo, r.compatInfo);
            handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        } break;

        ...
    }
}
```

这里的handleLaunchActivity()就是启动activity真正的方法，这里就是另一个hook点，在handleLaunchActivity()中将目标activity的intent替换回来，首先实现Handler.Callback，代码如下。通过反射讲message中的intent换会原来的intent。

```java
public class ActivityThreadHandlerCallback implements Handler.Callback {

    private Handler mHandler;

    public ActivityThreadHandlerCallback(Handler handler) {
        this.mHandler = handler;
    }

    @Override
    public boolean handleMessage(Message message) {
        int what = message.what;
        switch (what) {
            case 100: //这里对应的是LAUNCH_ACTIVITY
                handleStartActivity(message);
                break;
        }
        mHandler.handleMessage(message);
        return true;
    }

    private void handleStartActivity(Message message) {
        Object object = message.obj;

        try {
            Field intent = object.getClass().getDeclaredField("intent");
            if (intent == null) {
                return;
            }
            intent.setAccessible(true);
            Intent rawIntent = (Intent) intent.get(object);

            Intent targetIntent = rawIntent.getParcelableExtra("extra_target_intent");
            if (targetIntent == null) {
                return;
            }
            rawIntent.setComponent(targetIntent.getComponent());

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

还差最后一步，就是将ActivityThread中的mCallback换成我们hook的callback，还是用老办法——反射。在ActivityThread中handler定义的变量为mH，所以主要替换的目标是mH，代码如下

```java
public static final void hookActivityThreadHandler() {
    try {
        Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
        Field currentActivityThreadField = activityThreadClass.getDeclaredField("sCurrentActivityThread");
        if (currentActivityThreadField == null) {
            return;
        }
        currentActivityThreadField.setAccessible(true);
        Object currentActivityThread = currentActivityThreadField.get(null);

        Field mHField = activityThreadClass.getDeclaredField("mH");
        if (mHField == null) {
            return;
        }
        mHField.setAccessible(true);
        Handler mH = (Handler) mHField.get(currentActivityThread);
        Field mCallbackField = Handler.class.getDeclaredField("mCallback");
        if (mCallbackField == null) {
            return;
        }

        mCallbackField.setAccessible(true);
        mCallbackField.set(mH, new ActivityThreadHandlerCallback(mH));

    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

到此，就神不知鬼不觉的绕过系统的校验，在启动的时候就是真正我们想要的页面了，而这个页面并没有在AndroidManifest.xml中注册。

## 小结

通过以上的方法，用动态代理分别hook了ActivityManagerNative中的IActivityManager和ActivityThread中handler的callback，用这两个hook点就成功的绕过了系统的校验，实现了启动没有声明的activity。看似简单，但是一定要对activity启动的流程十分熟悉，还要理解android中的binder机制，这就是android插件化框架用到的原理之一。除了动态代理，利用双亲委派机制也可以实现对相关方法的hook。

有些人也许会问，启动一个没有在AndroidManifest.xml中的activity有什么用。的确，正常应用开发的确没什么用。但是如果有一天你想动态下发一个activity的时候，就有用了。应用发布的时候不可能预埋不存在的activity，当有一天因为产品的变化，需求的变更的时候就只能发版解决了。但是如果可以做到动态下发，就可以静默对应用进行升级，而新增activity就是基础能力之一。

### 参考

+ [Android插件化原理解析——概要](http://weishu.me/2016/01/28/understand-plugin-framework-overview/)
+ [Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)
+ [Android应用程序内部启动Activity过程的源代码分析](http://blog.csdn.net/luoshengyang/article/details/6703247)
+ [DroidPlugin](https://github.com/DroidPluginTeam/DroidPlugin)
+ [VirtualApk](https://github.com/didi/VirtualAPK)