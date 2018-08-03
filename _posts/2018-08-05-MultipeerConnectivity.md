---
layout: post
title:  iOS端近场围栏检测（二） ——MultipeerConnectivity
author: wuhanbo555

---
![](https://upload-images.jianshu.io/upload_images/1767950-75b223abb22710e6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

### 前言
P2P已经成为时下很流行的一种交互方式，穿戴设备连手机、智能硬件连手机等等，那么在只能手机领域是否存在这样一种协议，能在没有WIFI，没有中间服务器的情况下，能直接2个手机传输信息呢？没错，之前的蓝牙是一种方式，苹果的AirDrop也是一种方式。今天我们就来聊一下MultipeerConnectivity这个在ios7系统上公开但是又极少有人使用的P2P方案。本章与其说是近场围栏检测，还不如说是近场通信更贴切，因为只是利用了其通信的能力才能发现周边设备并作出下一步动作。

### 如何使用
#### 使用条件
+ Multipeer Connectivity其功能与AirDrop非常接近，可以说苹果关闭了一扇门，又为我们打开了另一扇门。相比AirDrop，Multipeer Connectivity在进行发现和会话时并不要求同时打开WiFi和蓝牙，也不像AirDrop那样强制打开这两个开关，而是根据条件适时选择使用蓝牙或WiFi。
    + 关闭蓝牙和Wifi：无法使用
    + 只打开蓝牙：可以正常使用
    + 只打开wifi但是未连接同一个路由器：可以正常使用，会通过WiFi Direct传输
    + 同时打开wifi、蓝牙：那就是一个airDrop的完整功能

#### 流程分析

![P2P时序图](media/15144329132519/P2P%E6%97%B6%E5%BA%8F%E5%9B%BE.png)


#### 简单实现
##### 1.准备阶段
+ 发送者

```
//创建session
MCPeerID *peerID = [[MCPeerID alloc] initWithDisplayName:[[UIDevice currentDevice] name]];
self.session = [[MCSession alloc] initWithPeer:peerID securityIdentity:nil encryptionPreference:MCEncryptionRequired];
self.session.delegate = self;
    
//创建接收者信号监听
self.nearbyServiceBrowser = [[MCNearbyServiceBrowser alloc] initWithPeer:peerID serviceType:@"p2p-test"];
self.nearbyServiceBrowser.delegate = self;
[self.nearbyServiceBrowser startBrowsingForPeers];
```

+ 接收者

```
//创建session
MCPeerID *peerID = [[MCPeerID alloc] initWithDisplayName:[UIDevice currentDevice].name];
self.session = [[MCSession alloc] initWithPeer:peerID securityIdentity:nil encryptionPreference:MCEncryptionRequired];
self.session.delegate = self;
    
//发送接收者信号
self.nearbyServiceAdveriser = [[MCNearbyServiceAdvertiser alloc] initWithPeer:peerID discoveryInfo:nil serviceType:@"p2p-test"];
self.nearbyServiceAdveriser.delegate = self;
[self.nearbyServiceAdveriser startAdvertisingPeer];
```
这里一定要注意的是：
1.session的加密方式必须一致。
2.serviceType在1-15个字符之间，由ASCII字母、数字和“-”组成，不能以“-”为开头或结尾。如果出现MCErrorInvalidParameter错误，请自己检查是否满足要求。
3.广播者和监听广播者的serviceType必须一致才能匹配。

##### 2.session连接
+ 发送者

```
// 发现接收者
- (void)browser:(MCNearbyServiceBrowser *)browser foundPeer:(MCPeerID *)peerID
withDiscoveryInfo:(nullable NSDictionary<NSString *, NSString *> *)info
{
    self.peerID = peerID;
    //发起会话邀请
    [self.nearbyServiceBrowser invitePeer:self.peerID toSession:self.session withContext:nil timeout:10];
}

// 接收者消失，清空peerID
- (void)browser:(MCNearbyServiceBrowser *)browser lostPeer:(MCPeerID *)peerID
{
    self.peerID = nil;
}

// 监听失败
- (void)browser:(MCNearbyServiceBrowser *)browser didNotStartBrowsingForPeers:(NSError *)error
{
    [browser stopBrowsingForPeers];
}
```

+ 接收者

```
// 收到会话邀请
- (void)advertiser:(MCNearbyServiceAdvertiser *)advertiser
didReceiveInvitationFromPeer:(MCPeerID *)peerID withContext:(nullable NSData *)context invitationHandler:(void (^)(BOOL accept, MCSession * __nullable session))invitationHandler
{ 
    //弹窗提示接收者是否连接session
    UIAlertController *alert = [UIAlertController alertControllerWithTitle:nil message:[NSString stringWithFormat:@"接收到%@的邀请", peerID.displayName] preferredStyle:UIAlertControllerStyleAlert];
    UIAlertAction *accept = [UIAlertAction actionWithTitle:@"接受" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        invitationHandler(YES, self.session);
    }];
    [alert addAction:accept];
    UIAlertAction *reject = [UIAlertAction actionWithTitle:@"拒绝" style:UIAlertActionStyleDestructive handler:^(UIAlertAction * _Nonnull action) {
        invitationHandler(NO, self.session);
    }];
    [alert addAction:reject];
    [self presentViewController:alert animated:YES completion:nil];
}

// 广播失败回调
- (void)advertiser:(MCNearbyServiceAdvertiser *)advertiser didNotStartAdvertisingPeer:(NSError *)error
{
   [advertiser stopAdvertisingPeer];
}
```

当接收者答应了发送者的会话邀请后，一切就开始发生了。反之一切回到原点。

##### 3.传输内容
+ 发送者

```
- (void)session:(MCSession *)session peer:(MCPeerID *)peerID didChangeState:(MCSessionState)state
{
    //session连接情况
    switch (state) {
        case MCSessionStateNotConnected:
            //未连接
            break;
        case MCSessionStateConnecting:
            //连接中
            break;
        case MCSessionStateConnected:
        {
            //已连接
            /* 此处可以选择以下几个方法发送
            
            发送Data
            - (BOOL)sendData:(NSData *)data
                     toPeers:(NSArray<MCPeerID *> *)peerIDs
                    withMode:(MCSessionSendDataMode)mode
                       error:(NSError * __nullable * __nullable)error;
           
            发送resource
            - (nullable NSProgress *)sendResourceAtURL:(NSURL *)resourceURL
                                  withName:(NSString *)resourceName
                                    toPeer:(MCPeerID *)peerID
                     withCompletionHandler:(nullable void (^)(NSError * __nullable error))completionHandler;

            发送stream
            - (nullable NSOutputStream *)startStreamWithName:(NSString *)streamName
                                          toPeer:(MCPeerID *)peerID
                                           error:(NSError * __nullable * __nullable)error;
            
            */
            
        }
            break;
    }
}
```

+ 接收者

```
// 传输进度
- (void)session:(MCSession *)session didStartReceivingResourceWithName:(NSString *)resourceName fromPeer:(MCPeerID *)peerID withProgress:(NSProgress *)progress
{
    //通过progress字段可以获取到指定内容的传输进度。
}

// 数据传输完成回调
- (void)session:(MCSession *)session didFinishReceivingResourceWithName:(NSString *)resourceName fromPeer:(MCPeerID *)peerID atURL:(NSURL *)localURL withError:(nullable NSError *)error
{
    if (error) 
    {
        //传输出错
    }
    else 
    {
        //通过localURL获取到一个传输完成的临时文件，此时在此方法中应将其存入到业务指定的沙盒文件中固化。
    }
}
```

### 场景联想
##### 下面YY了一些场景，可以作为MultipeerConnectivity的业务落脚点。
+ 1.文件传输
    + 这个就是一个最为常规的使用方式，本文中将注释代码补全就能达到其效果。我们可以利用MultipeerConnectivity自己完成一个AirDrop。
+ 2.近场围栏
    + 这个案例可谓是逆向思考，从上图中我们会发现发送者会一直搜索附近接收者发出的信号，所以我们将其反之，把一个接收者作为围栏的中心发射信号，附近所有的发送者就能获取到此信号尝试session，但是我们并不需要真正连接session，还是直接认为发送者就在我们指定围栏范围内。
+ 3.多屏互动
    + 这个是我个人比较看好的一个业务落脚点，我自己YY了2个Demo
        + 1.乒乓球小游戏：利用其发送数据包的能力，2台手机上各是半张乒乓球桌，而发送的数据就是乒乓球，通过玩家手在手机屏幕上的滑动转换为乒乓球方向、旋转、速度的数据丢给对面的玩家，这样，一场有趣的乒乓球赛就开始了。
        + 2.会场拼接屏：我们知道在大型的娱乐活动会场往往少不了LED拼接屏。而利用MultipeerConnectivity，我们手中的iphone手机相当于一块块的三星拼接屏，将每一台手机标号后（不同的serviceType），就能实现万人手机拼接屏互动的效果。
+ 4.聊天工具
    + 目前App Store上利用MultipeerConnectivity的聊天工具有很多（最有名的是FireChat），其不需要中间服务器就可以发现身边的他，是不是邂逅、撩妹必备神器。

### 结论

+ 道法有云：一生二，二生三，三生万物。而MultipeerConnectivity恰恰是可以像接力一样无限信号传播的一个方式。只要每个手机都遵循一定的规则，发挥我们的想象力，就可以用其做出非常多有意思的好玩的新功能。





