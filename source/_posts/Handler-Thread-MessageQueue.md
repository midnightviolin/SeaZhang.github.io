---
title: Handler Thread MessageQueue
date: 2018-09-11 10:12:50
tags: [Handler,Thread,MessageQueue,Looper,ThreadLocal]
---


### 异步消息处理


对于普通线程，执行完 run() 方法内代码后线程就结束了。
异步线消息处理线程，线程启动后会进入一个无线循环体这种，每循环异常，从其内部的消息队列中取出一个消息，并回调相应的消息处理函数，执行完成一个消息后则继续执行循环。如果消息队列为空，线程会暂停，直到消息队列中有新的消息。

异步线程内包含一个消息队列 `MessageQueue` ,消息先进先出原则进行处理；

线程的执行体使用无线循环，混合体中从消息中取出消息，根据消息来源，回调相应的消息处理函数；

其他线程可以向本线程的消息队列中发送消息，消息队列内部的读/写操作必须进行加锁，即消息队列不能同时进行读/写操作。



### Android 系统异步线程的实现方法：

在线程内部有一个或多个 `Handler` 对象，外部城市可以通过该 `Handler` 对象向线程发送异步消息，消息由 `Handler` 传递到 `MessageQueue` 中。线程内部只包含一个 `MessageQueue` 对象，调用 `loop（）` 方法，线程主执行函数从 `MessageQueue` 中读取消息，并回调 `Handler` 对象中的回调函数 `handleMessage（）`；

<!-- more -->

Handler 消息处理机制 

Handler,Looper,Message

关联类
MessageQueue ThreadLocal

相关类

HandlerThread


#### 0. 线程局部存储(Thread Local Storage) ThreadLocal 类

一个线程只能创建一个` Looper` 实例，通过 `ThreadLocal` 类进行控制及判断处理。
当该进程第一次调用 `Looper.prepare()` 时内部会将` Looper` 实例添加到 `ThreadLocal` 实例中；
如果尝试再次调用 `Looper.prepare()` 会抛出异常，提示一个线程只能有一个 `Looper` 对象。

#### 1.Looper类

用于给一个线程 执行一个消息循环。默认卡线程没有关联消息循环。
用 `prepare()` 方法创建一个消息循环。然后可以使用 `loop()` 方法处理消息。
通常处理循环消息是通过 `Handler` 类。

常用使用例子：
{% codeblock lang:Java %}

    class LooperThread extends Thread {
        public Handler mHandler;
      
        public void run() {
            Looper.prepare();
      
            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // process incoming messages here
                }
            };
      
            Looper.loop();
        }
    }

  {% endcodeblock %}



内部使用类 ThreadLocal，MessageQueue，Thread。

主要方法

##### 构造方法

构造方法私有，只通过 `prepare()` 初始化 `Looper` 实例。

Looper 初始化实例 会初始化 消息队列。

{% codeblock lang:Java %}

private Looper(boolean quitAllowed) {     
	
	mQueue = new MessageQueue(quitAllowed);     
	mThread = Thread.currentThread(); 

}

{% endcodeblock %}


##### perpare() 

为当前线程初始化一个消息循环，可以在调用该方法后，开始消息循环之前 设置关联到 消息循环 looper 上。

{% codeblock lang:Java %}

 	public static void prepare() {
        prepare(true);
    }

 	private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

{% endcodeblock %}


##### loop() 

为当前线程开启消息队列。确保循环结束时调用` quit()` 退出方法。
方法内部循环 `MessageQueue` 通过 `queue.next()` 方法取出消息，通过消息的 handler 实例的 `dispatchMessage` 方法分发消息。
 `msg.target.dispatchMessage(msg)` 转发到 handler 对应的 `handleMessage` 方法。

最后消息处理完成后，会调用 msg.recycleUnchecked() 方法回收该 Message 对象占用的系统资源。因为 Message 类内部使用了一个数据池保存 Message 对象，从而避免不停地创建和删除 Message 对象。因此，每次消息处理完成后，需要将该消息对象表明为空闲状态，以便于 Message 对象可以重复调用。

##### quit() 

对应调用 `MessageQueue` 中 `quit` 方法。


##### getMainLooper() 

返回整个应用 Application 的主 looper 实例，该实例运行在应用主进程中。

{% codeblock lang:Java %}

  public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }

{% endcodeblock %}



### 2.MessageQueue 消息队列



#### next() 取出消息



#### enquenceMessage() 添加消息


