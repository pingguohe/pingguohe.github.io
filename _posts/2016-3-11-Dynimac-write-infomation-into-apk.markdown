---

layout: post
title: 一种动态为apk写入信息的方案
author: 赵林

--- 


## 背景
我们在日常使用应用可能会遇到以下场景。
场景1：
用户浏览h5页面时看到一个页面，下载安装app后启动会来到首页而不是用户之前浏览的页面，造成使用场景的割裂。

场景2：
用户通过二维码把一个页面分享出去，没有装猫客的用户如果直接安装启动之后无法回到分享的页面。

如果用户在当前页面下载了应用，安装之后直接跳转到刚才浏览的界面，不仅可以将这一部分流量引回客户端，还可以让用户获得完整的用户体验。下面提出一种方案来满足这个业务需求。  

## 原理

android使用的apk包的压缩方式是[zip](https://en.wikipedia.org/wiki/Zip_(file_format))，与zip有相同的文件结构，在zip的Central directory file header中包含一个File comment区域，可以存放一些数据。File comment是zip文件如果可以正确的修改这个部分，就可以在不破坏压缩包、不用重新打包的的前提下快速的给apk文件写入自己想要的数据。  
comment是在Central directory file header末尾储存的，可以将数据直接写在这里，下[表](https://en.wikipedia.org/wiki/Zip_(file_format)#File_headers)是header末尾的结构。
![](/images/2016/3/11/header.png)

由于数据是不确定的，我们无法知道comment的长度，从表中可以看到zip定义comment的长度的位置在comment之前，所以无法从zip中直接获取comment的长度。这里我们需要自定义comment的长度，在自定义comment内容的后面添加一个区域储存comment的长度，结构如下图。  
![](/images/2016/3/11/comment.png)
这里可以将一个固定的结构写在comment中，然后根据自定义的长度分区获取每个部分的内容，还可以添加其它数据，如校验码、版本等。

## 实现

### 1.将数据写入comment
这一部分可以在本地进行，需要定义一个长度为2的byte[]来储存comment的长度，直接使用Java的api就可以把comment和comment的长度写到apk的末尾，代码如下。

```java
public static void writeApk(File file, String comment) {
    ZipFile zipFile = null;
    ByteArrayOutputStream outputStream = null;
    RandomAccessFile accessFile = null;
    try {
        zipFile = new ZipFile(file);
        String zipComment = zipFile.getComment();
        if (zipComment != null) {
            return;
        }

        byte[] byteComment = comment.getBytes();
        outputStream = new ByteArrayOutputStream();

        outputStream.write(byteComment);
        outputStream.write(short2Stream((short) byteComment.length));

        byte[] data = outputStream.toByteArray();

        accessFile = new RandomAccessFile(file, "rw");
        accessFile.seek(file.length() - 2);
        accessFile.write(short2Stream((short) data.length));
        accessFile.write(data);
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        try {
            if (zipFile != null) {
                zipFile.close();
            }
            if (outputStream != null) {
                outputStream.close();
            }
            if (accessFile != null) {
                accessFile.close();
            }
        } catch (Exception e) {

        }

    }
}
```

### 2.读取apk包中的comment数据
首先获取apk的路径，通过context中的getPackageCodePath()方法就可以获取，代码如下。

```java
public static String getPackagePath(Context context) {
    if (context != null) {
        return context.getPackageCodePath();
    }
    return null;
}
```
获取路径之后就可以读取comment的内容了，这里不能直接使用ZipFile中的getComment()方法直接获取comment，因为这个方法是Java7中的方法，**在android4.4之前是不支持Java7的**，所以我们需要自己去读取apk文件中的comment。首先根据之前自定义的结构，先读取写在最后的comment的长度，根据这个长度，才可以获取真正comment的内容，代码如下。

```java
public static String readApk(File file) {
    byte[] bytes = null;
    try {
        RandomAccessFile accessFile = new RandomAccessFile(file, "r");
        long index = accessFile.length();

        bytes = new byte[2];
        index = index - bytes.length;
        accessFile.seek(index);
        accessFile.readFully(bytes);

        int contentLength = stream2Short(bytes, 0);

        bytes = new byte[contentLength];
        index = index - bytes.length;
        accessFile.seek(index);
        accessFile.readFully(bytes);

        return new String(bytes, "utf-8");
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
    return null;
}
```
这里的stream2Short()和short2Stream()参考了MultiChannelPackageTool中的方法。

## 测试
在生成apk后，调用下面的代码写入我们想要的数据，

```java
File file = new File("/Users/zhaolin/app-debug.apk");
writeApk(file, "test comment");
```

安装这个apk之后运行，让comment显示在屏幕上，运行结果如下。
![](/images/2016/3/11/screen.png)  
运行结果符合预期，安装包也没有被破坏，可以正常安装。

## 结论
+ 通过修改comment将数据传递给APP的方案是可行的，由于是修改apk自有的数据，并不会对apk造成破坏，修改后可以正常安装。
+ 这种方案不用重新打包apk，并且在服务端只是写文件的操作，效率很高，可以适用于动态生成apk的场景。
+ 可以通过这个方案进行h5到APP的引流，用户操作不会产生割裂感，保证用户体验的统一。

### 参考
+ [packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)
+ [MultiChannelPackageTool](https://github.com/seven456/MultiChannelPackageTool)