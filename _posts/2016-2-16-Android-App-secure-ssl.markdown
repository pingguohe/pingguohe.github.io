# Android App数字证书校验

## 起因
前段时间，同事拿着一个代码安全扫描出来的 bug 过来咨询，我一看原来是个 https 通信时数字证书校验的漏洞，一想就明白了大概；其实这种问题早两年就有大规模的暴露，各大厂商App 也纷纷中招，想不到过了这么久天猫客户端里还留有这种坑；然后仔细研究了漏洞所在的代码片段，原来所属的是新浪微博分享 sdk 内部的，因为这个 sdk 是源码引用的，所以也就被扫描出来了。因此给出的解决方案是：

1. 先获取最新的 sdk，看其内部是否已解决，已解决的话升级 sdk 版本即可；
2. 第1步行不通，那就自己写校验逻辑，猫客全局通信基本已经使用 https 通信，参考着再写一遍校验逻辑也不是问题；

后来查了一下网上信息，早在2014年10月份，[乌云](http://www.wooyun.org/bugs/wooyun-2014-079358)平台里就已经暴露过天猫这个漏洞，想必当时一定是忙于双十一忽略了这个问题。

虽然这个问题通过升级 sdk 解决了，但是这个问题纯粹是由于开发者本身疏忽造成的；特别是对于初级开发人员来说，可能为了解决异常，屏蔽了校验逻辑；所以我还是抽空再 review 了一下这个漏洞，整理相关信息。

## 漏洞描述
对于数字证书相关概念、Android 里 https 通信代码就不再复述了，直接讲问题。缺少相应的安全校验很容易导致中间人攻击，而漏洞的形式主要有以下3种：

+ 自定义```X509TrustManager```，但覆盖了默认的安全校验逻辑且自己不校验，下面片段就是当时新浪微博 sdk 内部的代码片段。如果不提供自定义的```X509TrustManager```，代码运行起来会报异常，初学者就很容易在不明真相的情况下提供了一个自定义的```X509TrustManager```，却忘记正确地实现相应的方法。

```java
TrustManager tm = new X509TrustManager() {
    public void checkClientTrusted(X509Certificate[] chain, String authType)
            throws CertificateException {
              //do nothing，接受任意客户端证书
    }

    public void checkServerTrusted(X509Certificate[] chain, String authType)
            throws CertificateException {
              //do nothing，接受任意服务端证书
    }

    public X509Certificate[] getAcceptedIssuers() {
        return null;
    }
};

sslContext.init(null, new TrustManager[] { tm }, null);
```

+ 自定义了```HostnameVerifier```，默认接受所有域名；犯错的原因和第1点一样。代码示例。

```java
HostnameVerifier hnv = new HostnameVerifier() {
  @Override
  public boolean verify(String hostname, SSLSession session) {
    // Always return true，接受任意域名服务器
    return true;
  }
};
HttpsURLConnection.setDefaultHostnameVerifier(hnv);
```

+ 信任所有主机名。
```java
SSLSocketFactory sf = new MySSLSocketFactory(trustStore);
sf.setHostnameVerifier(SSLSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);
```

## 修复方案
分而治之，针对不同的漏洞点分别描述，这里就讲的修复方案主要是针对非浏览器App，非浏览器 App 的服务端通信对象比较固定，一般都是自家服务器，可以做很多特定场景的定制化校验。如果是浏览器 App，校验策略就有更通用一些。
+ 自定义```X509TrustManager```

+ 自定义```HostnameVerifier```，简单的话就是根据域名进行字符串匹配校验；业务复杂的话，还可以结合配置中心、白名单、黑名单、正则匹配等多级别动态校验；总体来说逻辑还是比较简单的，反正只要正确地实现那个方法。
```java
HostnameVerifier hnv = new HostnameVerifier() {
  @Override
  public boolean verify(String hostname, SSLSession session) {
    //示例
    if("yourhostname".equals(hostname)){  
      return true;  
    } else {  
      return false;  
    }
  }
};
```

+ 主机名验证策略改成严格模式
```java
SSLSocketFactory sf = new MySSLSocketFactory(trustStore);
sf.setHostnameVerifier(SSLSocketFactory.STRICT_HOSTNAME_VERIFIER);
```
