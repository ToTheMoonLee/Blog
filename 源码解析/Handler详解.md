# Handler详解
> 只要提到handler，大家肯定都会想到其他的几个类Looper、MessageQueue、Message，我也不例外，让我们先来看看这个几个类的官方介绍

### 概述
关于官方文档我就不截图了，大家可以自行点击前往官网进行阅读，下面是传送门：
[Handler](https://developer.android.com/reference/android/os/Handler#nested-classes)
[Looper](https://developer.android.com/reference/android/os/Looper)
[MessageQueue](https://developer.android.com/reference/android/os/MessageQueue)
[Message](https://developer.android.com/reference/android/os/Message#setData(android.os.Bundle))

下面我总结一下：

1. Handler可以发送和处理与消息队列关联的`Message`或者`Runnable`，每个`Handler`都只与一个`Thread`、这个`Thread`中的`Looper`和这个`Looper`中的`MessageQueue`相关联。**一个Thread**可以对应**多个Handler**，但是**一个Handler**只能对应**一个Thread**，**一个Looper**和**一个MessageQueue**。
2. Handler的两个主要用途： 1> 处理在将来某个时间执行的`Message`或者`Runnable`； 2> 将一个要在其他线程执行的功能包装成一个`Message`加入这个`Thread`对应的队列中，执行完成后再切回当前线程。也就是我们经常使用的，子线程做耗时操作，然后切回主线程更新UI。
3. Handler有两类API用来安排消息：1> `postXxx()`类型的，这种可以将`Runnable`插入到消息队列等待被执行；2> `sendMessageXxx`,这种可以将`Message`插入到消息队列，当遍历到该消息后，会执行到`Handler`的`handleMessage(Message)`方法。如果只是执行一段代码逻辑的话，直接使用`postXxx()`就可以；如果需要传递一些数据到`Handler`所在的线程执行，并且对要执行的`Message`进行组织的话，就使用`sendMessageXxx()`的方法。同时上述两种API可以允许你控制插入的消息立即执行、延时执行或者到某个时间点再执行。
4. 如果你不想实现一个`Handler`的子类，那么你可以使用带`Handler.Callback`参数的构造方法去初始化`Handler`。
5. `Looper`的作用是用来进行消息循环的，在一个普通的线程中，并没有关联`Looper`，我们需要使用`Looper`类的`prepare()`和`loop()`方法来启动消息循环，典型的使用如下：
		
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
	     
6. `MessageQueue`使用一个单链表组织一系列被`Handler`分发的`Message`对象，`Message`对象是通过对应`Looper`关联的`Hanlder`添加到`MessageQueue`的，通过`Looper#myQueue()`可以获取到所关联的`MessageQueue`的对象
7. `Message`可以携带要发送到`Handler`的信息，`Message`类有多个int的成员变量和其他类型的成员变量，int类型成员变量arg1和arg2，可以携带轻量级的数据，`Object`类型的成员变量obj可以携带任意类型的数据；初始化`Message`推荐使用`Message.obtain()`或者`Handler#obtainMessage()`，这样可以复用回收池里的`Message`对象

以上只是对官方文档对他们的描述的总结，下面简单介绍几种典型的用法

### API简介
API分为以下几类：

* 初始化方法：

| 方法             | 简介 |
| ------------- | ------------- |
| Handler()  |直接使用创建Handler线程的Looper做关联  |
| Handler(Handler.Callback callback) | 直接使用创建Handler线程的Looper做关联，<br>同时传入Handler.Callback，当你创建Handler，并且不想搞一个<br>handler的子类的话，可以用这个方法  |
 上述两个方法，由于隐式的关联`Looper`可能导致bug，官方标记为deprecated，不建议使用。下面两个是显式传入`Looper`的两个与之对应的方法，推荐使用：

|  方法  | 简介 |
| ------------- | ------------- |
| Handler(Looper looper) | 使用传入的Looper做关联 |
| Handler(Looper looper, Handler.Callback callback) | 使用传入的Looper做关联，同时传入Handler.Callback，<br>当你创建Handler，并且不想搞一个handler的子类的话，<br>可以用这个方法  |
	
* 初始化`Message`的一类方法

|  方法   |
| ---------- |
|obtainMessage(int what, int arg1, int arg2, Object obj)|
|obtainMessage(int what, int arg1, int arg2)|
|obtainMessage(int what)|
|obtainMessage()|
|上述四个重写的方法，当传入对应参数时，就会创建一个带了这些参数的`Message`，<br>创建`Message`时，会复用消息池中的消息，推荐使用|
	
* 添加消息的一类方法

|  方法  | 简介 |
| ------------- | ------------- |
|post(Runnable r) |直接发送一个Runnable|
|postAtFrontOfQueue(Runnable r)|将Runnable插入到队首|
|postAtTime(Runnable r, long uptimeMillis)|插入一个在给定的uptimeMillis执行的Runnable|
|postAtTime(Runnable r, Object token, long uptimeMillis)|插入一个在给定的uptimeMillis执行的Runnable，<br>token可以用来移除Runnable|
|postDelayed(Runnable r, long delayMillis)|插入一个延时delayMillis执行的Runnable|
|postDelayed(Runnable r, Object token, long delayMillis)|插入一个延时delayMillis执行的Runnable，<br>token可以用来移除Runnable|


|  方法   |
| -------------------------- |
|sendEmptyMessage(int what)|
|sendEmptyMessageAtTime(int what, long uptimeMillis)|
|sendEmptyMessageDelayed(int what, long delayMillis)|
以上三个是发送不带`Message`的一类方法，what是用来区分消息的

|  方法   |
| -------------------------- |
|sendMessage(Message msg)|
|sendMessageAtFrontOfQueue(Message msg)|
|sendMessageAtTime(Message msg, long uptimeMillis)|
|sendMessageDelayed(Message msg, long delayMillis)|
以上四个是发送带`Message`的一类方法，各个方法的功能与`postXxx()`的方法一样，`postXxx()`的方法源码其实最终还是使用的`sendMessageXxx()`的方法

其他API就不做过多赘述了

### 使用
	
* 子线程耗时操作，在主线程更新UI，使用的是`sendMessage()`方法，子线程拿到`Person`，并送到主线程中去执行

```
public static final String TAG = HandlerActivity.class.getSimpleName();

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler);
        final UIHandler handler = new UIHandler();
        findViewById(R.id.execute).setOnClickListener(view -> {
            new Thread() {
                @Override
                public void run() {
                    Log.i(TAG, "current Thread is " + Thread.currentThread().getName());
                    SystemClock.sleep(500);
                    Person person = new Person("lily", 23, 165);
                    Message message = handler.obtainMessage(0, person);
                    handler.sendMessage(message);
                }
            }.start();
        });

    }

    static class UIHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            if (msg.what == 0) {
                Person person = null;
                if (msg.obj instanceof Person) {
                    person = (Person) msg.obj;
                }
                Log.i(TAG, "current thread is " + Thread.currentThread().getName() + ", msg.what : 0, " + person);
            }
        }
    }
```

* 延时执行的方法，下面模拟的是延时1s后执行对应的操作，当然是在同一个线程中。这个使用的是post传入Runnable的方法，延时1s会执行run()方法

```
Handler delayHandler = new Handler();
findViewById(R.id.execute2).setOnClickListener(view -> {
  long start = SystemClock.uptimeMillis();
  delayHandler.postDelayed(new Runnable() {
      @Override
      public void run() {
          Log.i(TAG, "test handler delay " + (SystemClock.uptimeMillis() - start));
      }
  }, 1000);
});
```

### 源码解析
> 以下源码基于Android-29分析

##### 核心类图

![handle](media/16124310139409/handler.png)


##### 初始化分析

先从官方给出的Hanlder的典型使用方法开始入手，首先是创建Looper，会调用

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
	     
Looper.prepare()会调用到如下代码：

```
*Initialize the current thread as a looper.
* This gives you a chance to create handlers that then reference
* this looper, before actually starting the loop. Be sure to call
* {@link #loop()} after calling this method, and end it by calling
* {@link #quit()}.
public static void prepare() {
   prepare(true);
}

private static void prepare(boolean quitAllowed) {
   if (sThreadLocal.get() != null) {
       throw new RuntimeException("Only one Looper may be created per thread");
   }
   sThreadLocal.set(new Looper(quitAllowed));
}
```
从官方注释中可以看出，prepare()可以将当前线程初始化成一个Looper，调用loop()来开启循环，调用quit()来结束循环。在这里，sThreadLocal用来将Looper与当前线程绑定，也就是在哪个线程调用了Looper.prepare()方法，那么初始化出来的Looper对象就会绑定在哪个线程，一般情况下同一进程内的不同进程是共享内存的，但是ThreadLocal是让线程的内存私有化的。从代码中可以看出一个线程只允许有一个Looper。传入的quitAllowed参数，是用来标志该Looper是否可以退出的。

Looper.loop()方法代码如下：

```

```


### 参考

[Android消息机制1-Handler(Java层)](http://gityuan.com/2015/12/26/handler-message-framework/)
[Android中为什么主线程不会因为Looper.loop()里的死循环卡死？](https://www.zhihu.com/question/34652589/answer/90344494)
[android - what is message queue native poll once in android?](https://stackoverflow.com/questions/38818642/android-what-is-message-queue-native-poll-once-in-android)




