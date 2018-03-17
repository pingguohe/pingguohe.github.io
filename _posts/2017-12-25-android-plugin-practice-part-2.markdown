---

layout: post
title: Android插件化实践(2)--ClassLoader
author: zhaolin1230

---

## 前言

在上一篇文章[如何启动一个没有在AndroidManifest中注册的activity](http://pingguohe.net/2017/12/25/android-plugin-practice-part-1.html)中简单介绍了如何绕开ActivityManagerSerivce(AMS)的校验启动一个没有在AndroidManifest.xml中声明的activity。然而启动这么一个写在应用内部的activity是没有多大意义的，想实现动态下发新的activity，就一定要想办法从外部获取activity并启动它。

## ClassLoader简单介绍

了解Java的同学一定都知道，Java通过是ClassLoader去加载类的，首先看一下CLassLoader初始化的时候做了什么。初始化的时候会createSystemClassLoader()生成一个ClassLoader。

```java
static private class SystemClassLoader {
    public static ClassLoader loader = ClassLoader.createSystemClassLoader();
}
```

在createSystemClassLoader()方法中返回了一个PathClassLoader()的实例，从构造方法里可以看到PathClassLoader对象传入的parent是BootClassLoader的实例。

```java
private static ClassLoader createSystemClassLoader() {
    String classPath = System.getProperty("java.class.path", ".");
    String librarySearchPath = System.getProperty("java.library.path", "");
    return new PathClassLoader(classPath, librarySearchPath, BootClassLoader.getInstance());
}
```

pathClassLoader的实现很简单，继承BaseDexClassLoader，代码如下

```java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    
    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

这里的PathClassLoader就是用来加载安装后的apk文件的。在android中还有另一类ClassLoader——DexClassLoader，它主要用来加载外部apk或者jar文件的。DexClassLoader实现很简单，代码如下。

```
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

可以看到PathClassLoader和DexClassLoader都继承BaseDexClassLoader，BaseDexClassLoader主要代码如下。

```java
public class BaseDexClassLoader extends ClassLoader {
    
    ...
    
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.originalPath = dexPath;
        this.pathList =
            new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        Class clazz = pathList.findClass(name);
        if (clazz == null) {
            throw new ClassNotFoundException(name);
        }
        return clazz;
    }
    
    ...
}
```

在构造方法中会将dex的ClassLoader、路径等存到DexPathList中。findClass时先从pathList查找类。
了解了这几个ClassLoader之后，我们再来看看ClassLoader是如何工作的。loadClass方法代码如下

```java
protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
    Class<?> clazz = findLoadedClass(className);
    if (clazz == null) {
        try {
            clazz = parent.loadClass(className, false);
        } catch (ClassNotFoundException e) {
            
        }
        if (clazz == null) {
            clazz = findClass(className);
        }
    }
    return clazz;
}
```

从代码中可以看到loadClass会先从BootClassLoader中查找，如果没有找到再从parent的ClassLoader中查找，如果还没有再从当前的ClassLoader中查找。过程如下图所示

![](/images/2017/12/20171225214227.png)

我们再来看一下上面几个ClassLoader的依赖关系，从代码中可以看到PathClassLoader的parent是BootClassLoader，下面通过一段代码来验证一下。

```java
ClassLoader classLoader = getClassLoader();
        Log.d(TAG, "ClassLoader: " + (classLoader == null ? " null" : classLoader.toString()));

ClassLoader parentClassLoader = classLoader.getParent();
Log.d(TAG, "parentClassLoader: " +  (parentClassLoader == null ? "null" : parentClassLoader.toString()));

ClassLoader pParentClassLoader = parentClassLoader.getParent();
Log.d(TAG, "parent's parentClassLoader: " +  (pParentClassLoader == null ? "null" : pParentClassLoader.toString()));    
```

运行这段代码之后结果如下

```
ClassLoader: dalvik.system.PathClassLoader

parentClassLoader: java.lang.BootClassLoader@de975c2

parent's parentClassLoader: null
```

当前的classLoader是PathClassLoader，parent的ClassLoader是BootClassLoader，而BootClassLoader没有parent的ClassLoader，和验证了之前代码的结论。这就是Java中的父委托机制，Java在查找类的时候会先从parent的ClassLoader查找，是从上到下的过程。比如我们写了一个类java.lang.String，这个类和Java中的String包名类名都一样，虚拟机在调用String对象的时候并不会调用我们自定义的类，而是会优先使用Java中的String，有兴趣的同学可以自己写段代码试验一下。

那么问题来了，上面说到要加载外部apk的类需要DexClassLoader，而从上面的结果可以看出app运行的过程中跟DexClassLoader并没有什么联系，父委托机制也和DexClassLoader没有关联，那么如何去加载外部apk的类呢？接着往下看。

## 如何加载外部apk的类

上面提到父委托机制，再联想一下ClassLoader的构造方法中有一个参数是parent，那么是不是有办法把PathClassLoader的parent替换成我们想要的DexClassLoader，在把DexClassLoader的parent设置成BootClassLoader，再加上父委托的机制，查找类的过程就变成BootClassLoader->DexClassLoader->PathClassLoader，这样是不是就可以加载外部apk的类了呢？下面我们来试验一下

```java
public static void loadApk(Context context, String apkPath) {
    File dexFile = context.getDir("dex", Context.MODE_PRIVATE);

    File apkFile = new File(apkPath);

    ClassLoader classLoader = context.getClassLoader();
    DexClassLoader dexClassLoader = new DexClassLoader(apkFile.getAbsolutePath(),
            dexFile.getAbsolutePath(), null, classLoader.getParent());

    try {
        Field fieldClassLoader = ClassLoader.class.getDeclaredField("parent");
        if (fieldClassLoader != null) {
            fieldClassLoader.setAccessible(true);
            fieldClassLoader.set(classLoader, dexClassLoader);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

首先获取当前的ClassLoader，然后新建一个DexClassLoader，把ClassLoader的parent设置为DexClassLoader的parent，这个parent就是BootClassLoader，这样就完成了第一步，将DexClassLoader的parent设置成BootCLassLoader。第二部，通过反射取出当前ClassLoader中的parent，当前的ClassLoader就是PathClassLoader，取出parent之后将上一步新建的DexClassLoader设置为parent，这样就可以轻松的在类加载的时候插入DexCLassLoader查找类的过程，下面通过一段代码试验一下。我们自定义一个Application，在比较早的时机加载一个外部apk，这个apk中只有一个activity，用来实验。

```java
public class HookApplication extends Application {

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        LoadApkUtils.loadApk(base, Environment.getExternalStorageDirectory().getPath()
                + File.separator + "lib.apk");
        HookUtils.hookActivityManagerService(getClassLoader());
        HookUtils.hookActivityThreadHandler();
    }
}
```

再次启动应用，结果如下

```
ClassLoader: dalvik.system.PathClassLoader

parentClassLoader: dalvik.system.DexClassLoader

parent's parentClassLoader: java.lang.BootClassLoader@4546fd3
```

从这里可以看到，已经成功的将DexClassLoader插到了PathClassLoader和BootClassLoader之间。再通过上一篇文章中介绍的[启动没有在AndroidManifest.xml注册的activity的方法](http://pingguohe.net/2017/12/25/android-plugin-practice-part-1.html)，就可以加载一个外部apk的activity。

![](/images/2017/12/20171225214310.png)


## 小结

通过以上方法，就可以从外部apk动态加载一个activity了，可以为应用动态的下发apk，实现新功能。但是目前为止外部的activity中不能包含资源，因为资源文件在编译的时候会生成一个id，而加载外部的activity是找不到id的。在后面会的文章会一步一步介绍如何通过hook AssetManager获取外部资源，实现完整的activity的加载。

## 小结之后

上面的代码在android 8.0上运行之后，在hook ActivityManagerNative的时候找不到gDefault的单例了，不了解gDefault可以看一看[前一篇文章](http://pingguohe.net/2017/12/25/android-plugin-practice-part-1.html)，通过查看源码发现，ActivityManagerNative不在承载相关单例，而是将IActivityManager的单例放在了ActivityManager中，这样hook点也要发生相应的变化，这就暴露了动态代理hook方法的一个缺点，需要为各个版本去适配。上面的代码只在android5.1和6.0上测试通过(理论上7.x也可以)。附上android8.0相关改动源码地址

+ [ActivityManagerNative](https://github.com/android/platform_frameworks_base/blob/oreo-r6-release/core/java/android/app/ActivityManagerNative.java)
+ [ActivityManager](https://github.com/android/platform_frameworks_base/blob/oreo-r6-release/core/java/android/app/ActivityManager.java)