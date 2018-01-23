---

layout: post
title:  VirtualView 工具使用说明
author: Longerian

---

前文[《天猫客户端组件动态化的方案——VirtualView 上手体验.md》](http://pingguohe.net/2018/01/09/a-taste-of-virtualview-android.html)都提到了自定义模板编译成二进制数据的过程，在 Android 版的 Playground 里内置了一个编译工具可以实时调测，然而业务开发过程中，不可能在手机上编译，而是在电脑或者后台去编译模板。因此这里提供了一个独立的工具来编译模板，这里介绍下它的使用方法。

### 前世今生
工具的源码也提交在 [github](https://github.com/alibaba/virtualview_tools/) 上。在一开始的设计里，编译模块是针对控件来设计的，每一个控件如 `NText`、`NImage`、`VHLayout` 等都有自己的编译类，编译类的继承结构体系与控件本身的继承结构体系一样。它们各自会解析对应控件的属性及控件本身的类型，将其转换成整型值或索引序列化到最终输出文件里；这就带来了一个问题，当需要扩展控件属性，或者添加自定义控件的时候，需要编写一个新的编译类注册到编译工具里，这对开发者来说十分不友好，且不说还得搭建编译工具工程的本身的运行环境，还得熟悉编译器类的编写逻辑才行。从下图的原始设计图就可以看出其复杂性。

![](https://gw.alicdn.com/tfs/TB1sW.pn8fH8KJjy1XbXXbLdXXa-1854-798.png)

为了解决这一弊端，对原始设计进行了重构，重构的核心目标就是不依赖源码就能进行模板编译，思路是通过配置化的方式声明描述控件类型、属性类型、解析方式等，方案实施上是将原先针对控件编写编译类的模式改成针对属性来编译写编译类，只需要一个通用的属性编译类即可，具体行为都通过配置文件来描述。将重构后的工具模板导出可执行文件，就可以在项目中直接使用了。废话了这么多，接下来就是本文的重点了。

### 使用说明

#### 独立运行模式

##### 文件介绍

下载源码之后，可执行工具存放在目录 TemplateWorkSpace 里，包含以下几个文件/目录（或运行后产生）：

| 文件                      | 作用                        |
| ----------------------- | ------------------------- |
| config.properties       | 配置组件 ID、xml 属性对应的 value 类型    |
| templatelist.properties | 编译的模板文件列表               |
| build                   | 二进制文件的输出目录                |
| template                | xml 的存放路径                  |
| compiler.jar            | java 代码编译后 jar 文件，执行 xml 的编译逻辑 |
| buildTemplate.sh        | 编译执行文件                    |

##### 如何运行

- 打开命令行 执行 `sh buildTemplate.sh`
- 模板编译后的文件会输出到 build 路径下

##### 配置 config.properties

- `VIEW_ID_XXXX`
  - 配置 xml 节点 id
  - 如配置 `VIEW_ID_FrameLayout=1`，则 xml 节点中的 `<FrameLayout>` 在编译后会用数值1代替
  - 节点配置以 **`VIEW_ID_`** 开头
- `property=ValueType`
  - 配置属性值的类型，配置对所有模板生效，不支持在 1.xml 和 2.xml 中对相同的属性用不同的 ValueType 解析
  - 目前已经支持
    - 常规类型：`String`(默认，不需要配置)、`Float`、`Color`、`Expr`、`Number`、`Int`、`Bool`
    - 特殊类型 `Flag`、`Type`、`Align`、`LayoutWidthHeight`、`TextStyle`、`DataMode`、`Visibility`
    - 枚举类型 `Enum<name:value,……>`
	  - 枚举说明：
	    - 如配置 `flexDirection=Enum<row:0,row-reverse:1,column:2,column-reverse:3>`
	    - 在解析属性是配置 `row` 直接转化成 `int:0`，`row-reverse` 转成 `int:1`
- `DEFAULT_PROPERTY_XXXX`
  - 为了兼容就模板的编译，写的强制在二进制中写入一些属性类型定义，可以忽略

##### 配置 templatelist.properties

- 格式 `xmlFileName=outFileName,Version[,platform]`
  - `xmlFileName` 标识 template 目录下需要编译的 xml 文件名建议不带 `.xml` 后缀，目前做了兼容
  - `outFileName` 输出到 build 目录下的 `.out` 文件名
  - `Version` 表示 xml 编译后的版本号
  - `platform` 同时兼容 iOS 和 android 时不写，可填的值为 `android` 和`iphone`

##### xml 文件模板编写

- 和以前的方式一样，不需要额外写 Java 代码，只需要对新增的属性在config.properties 中配置 ValueType

#### 接口模式

除了直接使用命令行执行工具，还可以基于此搭建完整成熟的模板工具，它可以是个客户端，也可以是个后端服务，或者是个插件，所以需要提供接口模式供宿主程序调用。

```
//初始化构建对象
ViewCompilerApi viewCompiler = new ViewCompilerApi();
//设置配置文件加载器
viewCompiler.setConfigLoader(new LocalConfigLoader());
//读取模板数据
FileInputStream fis = new FileInputStream(rootDir);
//调用接口，传入必备参数，此时不区分平台，如果要区分平台，使用方单独编译即可
byte[] result = viewCompiler.compile(fis, "icon", 13);
```
