---
layout: post
title:  阿里巴巴开源项目TAC——新的服务端开发模式尝试
author: Longerian

---

TAC是一个基于java的微服务容器，提供从业务代码编写、编译、发布、jar动态加载、运行等一系列常用开发流程的支持，是天猫App在服务端开发模式下的新尝试。TAC和客户端框架Tangram结合，极大提高开发效率；TAC目前在天猫App、手机淘宝特价版广泛使用；

## 疼痛之初——产生背景

天猫App(以下简称猫客)首页从2015年的坑位运营走向2016年的全面个性化，当时猫客首页的个性化业务多大50多处。以首页为例，这个过程中除了接入导购链路的二方服务之外，接入了大量的三方服务；同时我们发现：

+ 首页应用越来越复杂、庞大，应用代码修改、编译部署耗费时间太长，大部分时间都是苦力劳动；
+ 线上定位问题周期太长，从问题发现到定位再到处理，需要经过客户端、服务端、下游各种服务提供方，链路太长影响业务快速支撑；
+ 大量服务的接入导致发布线上部署频繁，一个简单服务的接入，几行代码的修改导致首页应用重新开发编译部署，影响整个首页稳定性；

当时微服务的概念已经很火，同时阿里内部也开始大力推广docker(现在大部分应用都跑在docker上)。我们想过将庞大的首页应用拆分成微服务，但是和基础服务不同，前台业务变化快，修改频繁，拆分之后开发同学依然要面临各种服务接入等体力劳动，同时需要维护拆分之后的多个应用，反而增加了劳动成本，因此拆分成微服务的方式治标不治本；

![](https://cdn.yuque.com/lark/0/2018/png/2827/1526779032906-3d27d7b3-a99e-44a0-a893-ec34dfcad570.png)

## 解决之道——TAC

在此背景下TAC孕育而生，它提供低成本开发与发布流程、低成本搭建与维护开发环境、高稳定性保障；TAC通过热部署的方式使得研发同学够从苦力劳动中解放出来，回归到业务开发中去，同时一个基础服务接入之后，能够提供给多个业务使用；在此模式下，业务能够进行更细粒度的拆分，且故障隔离业务A的改动不会影响业务B；

在TAC的帮助下，频繁修改的新业务可快速上线，不会出现因为修改一个字段、几行代码就需要重新发布整个应用的情况；同时与tangram结合，实现页面卡片、坑位的快速调整；

![](https://cdn.yuque.com/lark/0/2018/png/2827/1526779052711-79746a4f-6ca4-4624-9a3c-5f60d6588b71.png)

经过近三年的沉淀，我们今日放出了开源版本，将集团版本的TAC剥离与阿里相关的中间件、网络、协议、部署环境等，保留其核心功能；开源版本提供了编译、热加载、运行的基础能力；

## TAC开源版本

### 系统结构

![](https://cdn.yuque.com/lark/0/2018/png/2827/1526779086525-184fc86c-bbf4-42d6-95b0-fba9f7eaeffc.png)

+ 开源版本分两部分，tac容器和tac控制台，存储和通信都依赖redis。
+ 为了简便，容器与外部服务直接的交互只通过http进行；
+ 同时为了提升开发体验，提供了与gitlab集成的能力，用户可直接gitlab提交代码并在控制台操作发布；快速验证；

### 核心类加载器

![](https://cdn.yuque.com/lark/0/2018/png/2827/1526779098483-35111af7-5444-4306-b25d-bd4105331e55.png)

上图是tac的类加载器结构，每个线上的微服务实例都通过一个新的classloader加载，同时为了方便用户扩展新的数据源，在AppClassLoader上扩展了一个classloader以加载第三方数据源(当然也可以直接在代码中扩展,见tac-infrastructure)；

## Quick Start —— 如何使用

+ 安装 [redis](https://redis.io/)
+ 运行 container

```
java -jar tac-container.jar
```

+ 运行 console 控制台

```
java -jar tac-console.jar --admin
```

+ 成功后可打开控制台

```
http://localhost:7001/#/tacMs/list
```

+ 代码开发
	+ 添加 SDK 依赖

```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>tac-sdk</artifactId>
    <version>${project.version}</version>
</dependency>
```

+ 编写代码

```
public class HelloWorldTac implements TacHandler<Object> {

    /**
     * 引入日志服务
     */
    private TacLogger tacLogger = TacServiceFactory.getLogger();

    /**
     * 编写一个实现TacHandler接口的类
     *
     * @param context
     * @return
     * @throws Exception
     */

    @Override
    public TacResult<Object> execute(Context context) throws Exception {

        // 执行逻辑
        tacLogger.info("Hello World");

        Map<String, Object> data = new HashMap<>();
        data.put("name", "hellotac");
        data.put("platform", "iPhone");
        data.put("clientVersion", "7.0.2");
        data.put("userName", "tac-userName");
        return TacResult.newResult(data);
    }
}
```

+ 本地编译、打包

+ 发布及测试

![](https://gw.alicdn.com/tfs/TB14LFbk2uSBuNkHFqDXXXfhVXa-3790-1510.png)

+ 正式发布

+ 线上验证

```
curl  http://localhost:8001/api/tac/execute/helloworld -s|json
```

```
{
  "success": true,
  "msgCode": null,
  "msgInfo": null,
  "data": {
    "helloworld": {
      "data": {
        "name": "hellotac",
        "clientVersion": "7.0.2",
        "userName": "tac-userName",
        "platform": "iPhone"
      },
      "success": true,
      "msCode": "helloworld"
    }
  },
  "hasMore": null,
  "ip": "127.0.0.1"
}
```

## 相关链接

+ [苹果核](http://tangram.pingguohe.net/)
+ [在 TAC 上使用 Tangram4TAC SDK](http://pingguohe.net/2018/05/24/tangram-for-tac.html)
+ [TAC](https://github.com/alibaba/tac)
+ [Tangram-Android](https://github.com/alibaba/tangram-android)
+ [Tangram-iOS](https://github.com/alibaba/tangram-ios)
