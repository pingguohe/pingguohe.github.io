---

layout: post
title: Java事件总线编程初探
author: 李金栓

--- 

Why?
===
在平时写代码的过程中，我们需要实现这样一种功能：当执行某个逻辑时，希望能够进行其他逻辑的处理。最粗暴的方法是直接依赖其他模块，调用该模块的相应函数或者方法。但是，这样做带来一些问题。

+ 模块间相互依赖，耦合度高。以下订单为例，订单提交后需要进行支付以及进行一些其他处理，如发邮件等操作。相关的代码可能是这样。可以看到：订单模块依赖了支付服务以及用户服务。
+ 维护困难。由于模块间相互依赖，当需要修改订单逻辑时则需要修改submitOrder方法的源代码，而某些时候可能无法修改。再者，如果有多个这种逻辑，修改时可能涉及到多处操作。

```java
public class OrderPage {
 
        private PaymentService paymentService;
        private UserService userService;
         
        public void submitOrder() {
                Integer userId = 1;
                BigDecimal amount = BigDecimal.TEN;
                 
                paymentService.doPayment(userId, amount);
                userService.registerPayment(userId, amount);
        }
}
public class PaymentService {
 
        private MailService mailService;
         
        public void doPayment(Integer userId, BigDecimal amount) {
                //Do payment...
                mailService.sendPaymentEmail(userId, amount);
        }
}
public class UserService {
 
        public String getEmailAddress(Integer userId) {
                return "foo@bar.com";
        }
         
        public void registerPayment(Integer userId, BigDecimal amount) {
                //Register payment in database...
        }
}
 
public class MailService {
 
        private UserService userService;
         
        public void sendPaymentEmail(Integer userId, BigDecimal amount) {
                String emailAddress = userService.getEmailAddress(userId);
                //Send email...
        }
}
```

观察者模式
===

> 有时被称作发布/订阅模式，观察者模式定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。这个主题对象在状态发生变化时，会通知所有观察者对象，使它们能够自动更新自己。

通过观察者模式来进行解耦，当对象发生变化时，通知其观察者，由观察者进行相应的处理。体现在订单逻辑中时即为,定义多个观察者观察下订单这个主题，当下订单的动作发生时，通知其所有观察者。再由每个观察者进行处理。依据观察者模式的实现，以上逻辑可改为如下代码：


```java
interface OrderListener {

	public void onSubmitOrder(Integer userId, BigDecimal amount);
}


public class OrderPage {

	private List<OrderListener> orderListeners=new ArrayList<OrderListener>();

	public void submitOrder() {
		Integer userId = 1;
		BigDecimal amount = BigDecimal.TEN;

		for (OrderListener orderListener : orderListeners) {
			orderListener.onSubmitOrder(userId, amount);
		}
	}
	
	public void addOrderListener(OrderListener orderListener){
		this.orderListeners.add(orderListener);
	}
}

class PaymentService implements OrderListener {

	private MailService mailService;

	public void doPayment(Integer userId, BigDecimal amount) {
		// Do payment...
		mailService.sendPaymentEmail(userId, amount);
	}

	@Override
	public void onSubmitOrder(Integer userId, BigDecimal amount) {

		doPayment(userId, amount);
	}
}

class UserService implements OrderListener {

	public String getEmailAddress(Integer userId) {
		return "foo@bar.com";
	}

	public void registerPayment(Integer userId, BigDecimal amount) {
		// Register payment in database...
	}

	@Override
	public void onSubmitOrder(Integer userId, BigDecimal amount) {

		registerPayment(userId, amount);

	}
}

class MailService {

	private UserService userService;

	public void sendPaymentEmail(Integer userId, BigDecimal amount) {
		String emailAddress = userService.getEmailAddress(userId);
		// Send email...
	}
}
```

可以看到，首先定义了OrderListener接口，接口中有一个onSubmitOrder方法。原始的实现中的PayService和UserService实现了该接口。OrderPage中维护了一个OrderListener列表，当提交订单时调用所有监听者的onSubmitOrder方法。可以看到此实现的订单逻辑没有直接依赖付款模块和用户模块。
主程序通过添加监听器来使其得到通知.

```java
 public static void main(String[] args) {
    	PaymentService paymentService=new PaymentService();
    	UserService userService=new UserService();
    	
    	OrderPage orderPage=new OrderPage();
    	
    	orderPage.addOrderListener(paymentService);
    	orderPage.addOrderListener(userService);
	}
```

Guava EventBus——监听者模式的优雅实现
===
虽然监听者模式对源代码进行了解耦，但是还是有一些不足。

+ 相关模块需要实现相应接口；
+ 需要主动调用相关的addListener方法设置监听器。
+ 一个监听器智能监听一种操作.

EventBus是Guava对于监听者模式的实现，其使用非常简单。使用EventBus来实现监听者模式，只需要三步操作。

 1. 通过注解@Subscribe来声明事件回调方法；
 2. 调用EventBus的register方法来注册监听器；
 3. 通过post方法来触发事件；

订单逻辑通过EventBus事件总线来实现，大概是以下这个样子。

```java
public class OrderPage {

	public static EventBus eventBus = new EventBus();

	public void submitOrder() {
		Integer userId = 1;
		BigDecimal amount = BigDecimal.TEN;

		eventBus.post(new PayEvent(userId, amount));
	}

}

class PaymentService {

	private MailService mailService;

	@Subscribe
	public void doPayment(PayEvent  payEvent) {
		// Do payment...
		mailService.sendPaymentEmail(payEvent.getUserId(), payEvent.getAmount());
	}

}

class UserService {

	public String getEmailAddress(Integer userId) {
		return "foo@bar.com";
	}

	@Subscribe
	public void registerPayment(PayEvent payEvent) {
		// Register payment in database...
	}
}

class PayEvent {

	private Integer userId;
	private BigDecimal amount;
	
	public PayEvent(Integer userId, BigDecimal amount) {
	}
	
	public Integer getUserId() {
		return userId;
	}
	public BigDecimal getAmount() {
		return amount;
	}
}

 public static void main(String[] args) {
    	PaymentService paymentService=new PaymentService();
    	UserService userService=new UserService();
    	
    	OrderPage orderPage=new OrderPage();
    	
    	orderPage.eventBus.register(paymentService);
    	orderPage.eventBus.register(userService);
	}
```

要实现监听者模式，时需要调用eventBus的register方法进行注册，在需要处理事件的方法上使用@Subscribe注解。最后通过eventBus发布事件即可。使用事件总线，不需要定义特定的接口，不需要主动添加监听器；

### 事件订阅
EventBus通过register方法来注册处理相应事件的类，

```java
public void register(Object object) {
    Multimap<Class<?>, EventSubscriber> methodsInListener =
        finder.findAllSubscribers(object);
    subscribersByTypeLock.writeLock().lock();
    try {
      subscribersByType.putAll(methodsInListener);
    } finally {
      subscribersByTypeLock.writeLock().unlock();
    }
  }
```

其核心是findAllSubscribers，找到实例中所有有Subscribe注解的方法并保存。返回的是一个Multimap < Class<?>,EventSubscriber>类型，其中Class是事件类型，EventSubsciber包含了类实例和具体处理事件的方法。Multimap保证了一种事件可以有多个监听者来处理。

```java
public Multimap<Class<?>, EventSubscriber> findAllSubscribers(Object listener) {
    Multimap<Class<?>, EventSubscriber> methodsInListener = HashMultimap.create();
    Class<?> clazz = listener.getClass();
    for (Method method : getAnnotatedMethods(clazz)) {
      Class<?>[] parameterTypes = method.getParameterTypes();
      Class<?> eventType = parameterTypes[0];
      EventSubscriber subscriber = makeSubscriber(listener, method);
      methodsInListener.put(eventType, subscriber);
    }
    return methodsInListener;
  }
```

### 发布事件

EventBus通过post方法来发布事件，首先通过事件类型找到需要处理的事件：事件本身以及其父类。根据事件类型从事件订阅的缓存中取出处理该事件的订阅者，并将其入队。最后处理该队列中的数据。

```java
public void post(Object event) {
    Set<Class<?>> dispatchTypes = flattenHierarchy(event.getClass());

    boolean dispatched = false;
    for (Class<?> eventType : dispatchTypes) {
      subscribersByTypeLock.readLock().lock();
      try {
        Set<EventSubscriber> wrappers = subscribersByType.get(eventType);

        if (!wrappers.isEmpty()) {
          dispatched = true;
          for (EventSubscriber wrapper : wrappers) {
            enqueueEvent(event, wrapper);
          }
        }
      } finally {
        subscribersByTypeLock.readLock().unlock();
      }
    }

    if (!dispatched && !(event instanceof DeadEvent)) {
      post(new DeadEvent(this, event));
    }

    dispatchQueuedEvents();
  }
```