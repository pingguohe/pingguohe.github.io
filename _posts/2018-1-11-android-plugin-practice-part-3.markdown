---

layout: post
title: Android插件化实践(3)--热修复
author: zhaolin1230

---

## 前言

之前的文章里讲到插件化最基础的内容，如何利用动态代码和父委托机制实现activity的动态下发。这篇文章会讲到如何利用ClassLoader进行hotfix。

## 类加载过程

### 父委托机制

首先回顾一下父委托机制，在Java中查找类的过程是从父ClassLoader向子ClassLoader进行的，具体参考[Android插件化实践(2)](http://pingguohe.net/2017/12/25/android-plugin-practice-part-2.html)，过程如下


<img src="/images/2018/1/20180111_1.png" width="150">

那么在某个ClassLoader内部是如何实现findClass的呢？看源码，首先看BaseDexClassLoader(源码位置libcore/dalvik/src/main/java/dalvik/system/)，它是PathClassLoader的父类，在构造方法中可以看到生成了一个DexPathList的实例，同样传入了dexPath。

```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
    super(parent);
    this.pathList = new DexPathList(this, dexPath, librarySearchPath, null);

    if (reporter != null) {
        reportClassLoaderChain();
    }
}
```

接下来在BaseDexClassLoader的findClass()方法中可以看到调用了DexPathList中的findClass()方法，代码如下

```java
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c == null) {
        ClassNotFoundException cnfe = new ClassNotFoundException(
                "Didn't find class \"" + name + "\" on path: " + pathList);
        for (Throwable t : suppressedExceptions) {
            cnfe.addSuppressed(t);
        }
        throw cnfe;
    }
    return c;
}
```

接着来看DexPathList(源码位置libcore/dalvik/src/main/java/dalvik/system/DexPathList.java)，这里的findClass()方法又调用了Element中的findClass()方法，代码如下

```java
public Class<?> findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
        Class<?> clazz = element.findClass(name, definingContext, suppressed);
        if (clazz != null) {
            return clazz;
        }
    }

    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}
```

再来看看dexElements的定义，是一个Element数组，这个类中储存真正的dex文件(DexFile)，具体内容可以看代码

```java
private final Element[] dexElements;
```

最终又调用了Element中的findClass()方法，代码如下

```java
public Class<?> findClass(String name, ClassLoader definingContext, List<Throwable> suppressed) {
    return dexFile != null ? dexFile.loadClassBinaryName(name, definingContext, suppressed) : null;
}
```

以上findClass的过程可以看下图

<img src="/images/2018/1/20180111_2.png" width="400">

最后一步中可以看到会遍历Element数组，里面存储着ClassLoader中的dexFile，而且是顺序遍历的。如果在类查找的过程中有机会把patch修复的类插到最前面，这样就可以在执行方法的时候替换掉有bug的类，完成热修复。

<img src="/images/2018/1/20180111_3.png" width="400">

## 实现

看核心代码，根据上面的思路，可以通过DexClassLoader加载一个patch，并将这个ClassLoader中的Element数组取出放到PathClassLoader中的Element数组前面即可，代码如下，变量中已dex开头的为DexClassLoader相关实例，以path开头的为PathClassLoader相关实例。

```java
private static void mergePathList(Context context, String dexPath) {
    File optPath = context.getDir("dex", Context.MODE_PRIVATE);

    ClassLoader parent = context.getClassLoader();
    if (parent == null) {
        return;
    }

    //通过DexClassLoader加载apk
    DexClassLoader dexClassLoader = new DexClassLoader(dexPath,
            optPath.getAbsolutePath(), null, parent);

    try {
        //获取外部dex中的pathList
        Class<?> baseDexClassLoader = Class.forName("dalvik.system.BaseDexClassLoader");

        //获取dex中的pathList
        Object dexPathList = getField(dexClassLoader, baseDexClassLoader, "pathList");
        Object dexElements = getField(dexPathList, dexPathList.getClass(), "dexElements");

        //获取本地apk中的pathList
        PathClassLoader pathClassLoader = (PathClassLoader) context.getClassLoader();
        Object pathPathList = getField(pathClassLoader, baseDexClassLoader, "pathList");
        Object pathElements = getField(pathPathList, pathPathList.getClass(), "dexElements");

        //合并pathList, 将修复bug的classLoader放在最前面
        Object merge = mergeDex(dexElements, pathElements);

        //将合并后的pathList设置回去
        Object pathList = getField(pathClassLoader, baseDexClassLoader, "pathList");
        setField(pathList, pathList.getClass(), "dexElements", merge);

        Log.d(TAG, "mergePathList: finish merge");

    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }

}
```

### 测试

我们自定义一个对象MyString，代码如下

```java
public class MyString {

    private String str;

    public MyString(String str) {
        this.str = str;
    }

    public int getLength() {
        return str.length();
    }
}

```

在activity中加一个按钮并实现onClick()方法，MyString传入null，这里必挂，因为成员变量str没有初始化

```java
public void onCrashClick(View v) {
    MyString myString = new MyString(null);
    Log.d(TAG, "onCrashClick: " + myString.getLength());
}
```

接下来我们写一个patch.apk，新加一个Android工程，里面只实现修复后的MyString，代码如下

```java
public class MyString {

    private String str;

    public MyString(String str) {
        this.str = str;
    }

    public int getLength() {
        return str == null ? 0 : str.length();
    }
}
```

将生成好的apk文件push到手机中，然后在启动的时候进行加载，然后再调用onCrashClick()的时候可以看到没有crash，并且看到日志如下

```shell
12-27 14:39:17.947 3555-3555/com.test.hotfix D/MainActivity: onCrashClick: 0
```

至此，可以看到已经通过热修复的方式修复了先前的bug。

注意：在Android6.0之后的手机上，如果将patch放到了/sdcard中一定要申请读权限，否则即使加载成功，已无法得到dex文件，造成patch失败，开始的时候就踩了这个坑，明明ClassLoader加载成功了但是patch失败。

## 小结

通过加载patch.pak的方式，并将Element插入到PathClassLoader中的Element最前面的方式可以进行热修复，在实际上是可行的。

但是这样有个缺点，就是要改一个类中的某个方法需要将整个类下发，而且不能是Android的四大组件，使用起来有局限性。另一方面，加载类的时机不好确定，很难做到立即生效，出问题的类在下次加载的时候才能修复，时效性一般。

附上ClassLoader相关源码的git地址: https://github.com/aosp-mirror/platform_dalvik.git

再附上这个demo的git地址: https://github.com/zhaolin1230/hotfix.git