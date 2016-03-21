---

layout: post
title: Ajax跨域问题在SpringMVC中的解决方案总结
author: ljinshuan

--- 

## 一次正常的请求

最近别人需要调用我们系统的某一个功能，对方希望提供一个api让其能够更新数据。由于该同学是客户端开发，于是有了类似以下代码。

```java
@RequestMapping(method = RequestMethod.POST, value = "/update.json", produces = MediaType.APPLICATION_JSON_VALUE)
	public @ResponseBody Contacter update(@RequestBody Contacter contacterRO) {

		logger.debug("get update request {}", contacterRO.toString());
		if (contacterRO.getUserId() == 123) {

			contacterRO.setUserName("adminUpdate-wangdachui");
		}

		return contacterRO;
	}
```
客户端通过代码发起http请求来调用。接着，该同学又提出：希望通过浏览器使用js调用，于是便有跨域问题。

## 为何跨域
简单的说即为浏览器限制访问A站点下的js代码对B站点下的url进行ajax请求。假如当前域名是www.abc.com，那么在当前环境中运行的js代码，出于安全考虑，正常情况下不能访问www.zzz.com域名下的资源。

+ 例如：以下代码再本域名下可以通过js代码正常调用接口

```js
(function() {
    var url = "http://localhost:8080/api/Home/update.json";

    var data = {
        "userId": 123,
        "userName": "wangdachui"
    };
    $.ajax({
        url: url,
        type: 'POST',
        dataType: 'json',
        data: $.toJSON(data),
        contentType: 'application/json'
    }).done(function(result) {
        console.log("success");
        console.log(result);
    }).fail(function() {
        console.log("error");
    })
})()
```

输出为:

```
Object {userId: 123, userName: "adminUpdate-wangdachui"}
```
+ 但是在其他域名下访问则出错:

![image](http://aligitlab.oss-cn-hangzhou-zmf.aliyuncs.com/uploads/tmallconfigcenter/minsk-report-wiki/1e8374d451f7e0c10416b2a46fc5584c/image.png)

```
OPTIONS http://localhost:8080/api/Home/update.json
XMLHttpRequest cannot load http://localhost:8080/api/Home/update.json. Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'null' is therefore not allowed access. The response had HTTP status code 403.
```

## 解决方案

### JSONP
使用jsonp来进行跨域是一种比较常见的方式，但是在接口已经写好的情况下，无论是服务端还是调用端都需要进行改造且要兼容原来的接口，工作量偏大，于是我们考虑其他方法。

### CORS协议

按照参考资料的说法：每一个页面需要返回一个名为‘Access-Control-Allow-Origin’的HTTP头来允许外域的站点访问。你可以仅仅暴露有限的资源和有限的外域站点访问。在COR模式中，访问控制的职责可以放到页面开发者的手中，而不是服务器管理员。当然页面开发者需要写专门的处理代码来允许被外域访问。
我们可以理解为：如果一个请求需要允许跨域访问，则需要在http头中设置Access-Control-Allow-Origin来决定需要允许哪些站点来访问。如假设需要允许www.foo.com这个站点的请求跨域，则可以设置：Access-Control-Allow-Origin:http://www.foo.com。或者Access-Control-Allow-Origin: * 。 CORS作为HTML5的一部分，在大部分现代浏览器中有所支持。

#### CORS具有以下常见的header
```
Access-Control-Allow-Origin: http://foo.org

Access-Control-Max-Age: 3628800

Access-Control-Allow-Methods: GET，PUT, DELETE

Access-Control-Allow-Headers: content-type

"Access-Control-Allow-Origin"表明它允许"http://foo.org"发起跨域请求

"Access-Control-Max-Age"表明在3628800秒内，不需要再发送预检验请求，可以缓存该结果

"Access-Control-Allow-Methods"表明它允许GET、PUT、DELETE的外域请求

"Access-Control-Allow-Headers"表明它允许跨域请求包含content-type头
```

#### CORS基本流程

首先发出预检验（Preflight）请求，它先向资源服务器发出一个OPTIONS方法、包含“Origin”头的请求。该回复可以控制COR请求的方法，HTTP头以及验证等信息。只有该请求获得允许以后，才会发起真实的外域请求。

#### Spring MVC支持CORS
```
Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource. Origin 'null' is therefore not allowed access. The response had HTTP status code 403.
```
![image](http://aligitlab.oss-cn-hangzhou-zmf.aliyuncs.com/uploads/tmallconfigcenter/minsk-report-wiki/ca575279f459fbbd52ca748dc73360fb/image.png)

从以上这段错误信息中我们可以看到，直接原因是因为请求头中没有Access-Control-Allow-Origin这个头。于是我们直接想法便是在请求头中加上这个header。服务器能够返回403，表明服务器确实对请求进行了处理。


##### MVC 拦截器
首先我们配置一个拦截器来拦截请求，将请求的头信息打日志。

```
DEBUG requestURL:/api/Home/update.json 
DEBUG method:OPTIONS 
DEBUG header host:localhost:8080 
DEBUG header connection:keep-alive 
DEBUG header cache-control:max-age=0 
DEBUG header access-control-request-method:POST 
DEBUG header origin:null 
DEBUG header user-agent:Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/49.0.2623.87 Safari/537.36 
DEBUG header access-control-request-headers:accept, content-type 
DEBUG header accept:*/* 
DEBUG header accept-encoding:gzip, deflate, sdch 
DEBUG header accept-language:zh-CN,zh;q=0.8,en;q=0.6 
```
在postHandle里打印日志发现，此时response的status为403。跟踪SpringMVC代码发现，在org.springframework.web.servlet.DispatcherServlet.doDispatch中会根据根据request来获取HandlerExecutionChain，SpringMVC在获取常规的处理器后会检查是否为跨域请求，如果是则替换原有的实例。

```java
@Override
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	Object handler = getHandlerInternal(request);
	if (handler == null) {
		handler = getDefaultHandler();
	}
	if (handler == null) {
		return null;
	}
	// Bean name or resolved handler?
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = getApplicationContext().getBean(handlerName);
	}

	HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
	if (CorsUtils.isCorsRequest(request)) {
		CorsConfiguration globalConfig = this.corsConfigSource.getCorsConfiguration(request);
		CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
		CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
		executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
	}
	return executionChain;
}
```
检查的方法也很简单，即检查请求头中是否有origin字段

```java
public static boolean isCorsRequest(HttpServletRequest request) {
	return (request.getHeader(HttpHeaders.ORIGIN) != null);
}
```

请求接着会交由 HttpRequestHandlerAdapter.handle来处理，根据handle不同，处理不同的逻辑。前面根据请求头判断是一个跨域请求，获取到的Handler为PreFlightHandler,其实现为：

```java
@Override
public void handleRequest(HttpServletRequest request, HttpServletResponse response) throws IOException {
	corsProcessor.processRequest(this.config, request, response);
}
```

继续跟进

```java
@Override
public boolean processRequest(CorsConfiguration config, HttpServletRequest request, HttpServletResponse response)
		throws IOException {

	if (!CorsUtils.isCorsRequest(request)) {
		return true;
	}

	ServletServerHttpResponse serverResponse = new ServletServerHttpResponse(response);
	ServletServerHttpRequest serverRequest = new ServletServerHttpRequest(request);

	if (WebUtils.isSameOrigin(serverRequest)) {
		logger.debug("Skip CORS processing, request is a same-origin one");
		return true;
	}
	if (responseHasCors(serverResponse)) {
		logger.debug("Skip CORS processing, response already contains \"Access-Control-Allow-Origin\" header");
		return true;
	}

	boolean preFlightRequest = CorsUtils.isPreFlightRequest(request);
	if (config == null) {
		if (preFlightRequest) {
			rejectRequest(serverResponse);
			return false;
		}
		else {
			return true;
		}
	}

	return handleInternal(serverRequest, serverResponse, config, preFlightRequest);
}
```

此方法首先会检查是否为跨域请求，如果不是则直接返回，接着检查是否同一个域下，或者response头里是否具有Access-Control-Allow-Origin字段或者request里是否具有Access-Control-Request-Method。如果满足判断条件，则拒绝这个请求。
由此我们知道，可以通过在检查之前设置response的Access-Control-Allow-Origin头来通过检查。我们在拦截器的preHandle的处理。加入如下代码:

```java
response.setHeader("Access-Control-Allow-Origin", "*");
```

此时浏览器中OPTIONS请求返回200。但是依然报错：

![image](http://aligitlab.oss-cn-hangzhou-zmf.aliyuncs.com/uploads/tmallconfigcenter/minsk-report-wiki/424d7bb8449f453afa50226767060192/image.png)


```
Request header field Content-Type is not allowed by Access-Control-Allow-Headers in preflight response.
```

我们注意到：在request的请求头里有Access-Control-Request-Headers:accept, content-type，但是这个请求头的中没有，此时浏览器没有据需发送请求。尝试在response中加入:

```
response.setHeader("Access-Control-Allow-Headers", "Origin, X-Requested-With, Content-Type, Accept");
```

执行成功：Object {userId: 123, userName: "adminUpdate-wangdachui"}。

至此：我们通过分析原理使SpringMVC实现跨域，原有实现以及客户端代码不需要任何改动。

#### SpringMVC 4

+ 此外，在参考资料2中，SpringMVC4提供了非常方便的实现跨域的方法。
+ 在requestMapping中使用注解。 @CrossOrigin(origins = "http://localhost:9000")
+ 全局实现 .定义类继承WebMvcConfigurerAdapter

```java
public class CorsConfigurerAdapter extends WebMvcConfigurerAdapter{

	@Override
	public void addCorsMappings(CorsRegistry registry) {
		
		registry.addMapping("/api/*").allowedOrigins("*");
	}
}
```
将该类注入到容器中：

```html
<bean class="com.tmall.wireless.angel.web.config.CorsConfigurerAdapter"></bean>
```

## 参考资料

+ [Cross-origin resource sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)
+ [Enabling Cross Origin Requests for a RESTful Web Service](http://spring.io/guides/gs/rest-service-cors/)