## Message全局池
原文地址：[http://www.jianshu.com/p/f6f357b3db89]
#### 总结：
1.UI不能在子线程中更新不太准确，UI必须在创建他的线程中更新（也不太准确，稍后分析源码）

2.Message是一个实现了Parcelable的Java类，
handler.postDelayed(Runnable r)这个方法并没有在一个新的线程中运行，内部仅仅是回调。

3.Message会绑定Handler对象，当从MessageQueue取出来的时候会调用
msg.target.dispatchMessage(msg)进行分发，分发给相应的handler去处理。

4.Message使用的时候最好不要new，调用obtain去获取一个Message（从池中获取）。
SDK-25中池的默认池的大小为50(private static final int MAX_POOL_SIZE = 50)。

5.Message自带一个Message next 属性，所以Message能自己维护一个链表，其实也就是
MessageQueue的实现原理。

具体内容请看原文。

## ThreadLocal
详解请参考[谈谈我对ThreadLocal的理解]

## Looper
Looper的SDK源码中给出了一个例子:
```
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
```
这是在一个线程中使用Looper的过程，首先在使用Looper的时候要先创建一个Looper对象，
在使用的时候首先调用Looper.prepare()，看一下源码：
```
    public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

   private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
   }

```
在创建Looper的时候其内部就会绑定一个MessageQueue对象；

在Looper内部有一个静态的ThreadLocal，
```
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
```
所有的线程持有的Looper都在sThreadLocal中保存,当在本线程中要使用Looper的时候从sThreadLocal
中获取。
Looper.loop(),是一个死循环,线程会不断的从本线程对应的MessageQueue中取出Message，
交给相对应的Handler去处理：
```
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
所以在调用Looper.loop()之后的代码不会执行。
内容补充：[http://www.jianshu.com/p/325d3cd79fd5]

## Handler

Handler我们最长用的就是发送Message和处理Message，首先看一个使用的例子：
```
     Handler myHandler = new Handler() {
          public void handleMessage(Message msg) {
               switch (msg.what) {
                    case TestHandler.GUIUPDATEIDENTIFIER:
                         myBounceView.invalidate();
                         break;
               }
               super.handleMessage(msg);
          }
     };
```
new一个Handler对象，重写handleMessage(Message msg)方法，此方法用于处理相应的Message，
否则调用父类的方法，看一下父类源码：
```
public void handleMessage(Message msg) {}
```
若不重写，则什么都不做。

看一下如何发送一个Message；
```
      Message msg =Message.obtain();  //从全局池中返回一个message实例，避免多次创建message（如new Message）
      msg.obj = data;
      msg.what=1;   //标志消息的标志
      handler.sendMessage(msg);
```
看过Looper的补充内容，你已经知道:sendMessage(msg)是如何调用的了。

![](https://github.com/AerialLadder/StudyNotes/blob/master/PIC/2017_10_10.png?raw=true)

这里就不详细讲解了。

说明一点，Handler的使用过程中注意内存泄露问题！
内容补充：[http://www.jianshu.com/p/338cce832cc9]

Handler改进：
> * 内存方面：使用静态内部类创建 handler 对象，且对 Activity 持有弱引用
> * 异常方面：不加 try catch，而是在 onDestory 中把消息队列 MessageQueue 中的消息给 remove 掉。
```
    /**
     为避免handler造成的内存泄漏
     1、使用静态的handler，对外部类不保持对象的引用
     2、但Handler需要与Activity通信，所以需要增加一个对Activity的弱引用
    */
      private static class MyHandler extends Handler {
        private final WeakReference<Activity> mActivityReference;

        MyHandler(Activity activity) {
            this.mActivityReference = new WeakReference<Activity>(activity);
        }

        @Override
            public void handleMessage(Message msg) {
            super.handleMessage(msg);
            MainActivity activity = (MainActivity) mActivityReference.get();  //获取弱引用队列中的activity
            switch (msg.what) {    //获取消息，更新UI
                case 1:
                    byte[] data = (byte[]) msg.obj;
                    activity.threadIv.setImageBitmap(activity.getBitmap(data));
                    break;
            }
        }
    }

```
并在 onDesotry 中销毁：
```
@Override
protected void onDestroy() {
    super.onDestroy();
    //避免activity销毁时，messageQueue中的消息未处理完；故此时应把对应的message给清除出队列
    handler.removeCallbacks(postRunnable);   //清除runnable对应的message
    //handler.removeMessage(what)  清除what对应的message
}
```
## HandlerThread

这部分内容比较简单，HandlerThread将loop转到子线程中处理，说白了就是将分担
MainLooper的工作量。
内容补充：[http://www.jianshu.com/p/35c8567419fa]

写完了这些我来总结一下他们之间的关系
1个线程只能拥有1个Looper
1个Looper只能拥有1个MessageQueue
1个Handler对应1个Looper
1个Message对应1个Handler

Message通过Handler被放到相应线程的MessageQueue中，
Looper循环取出MessageQueue中的Message，交给Handler处理。







[http://www.jianshu.com/p/f6f357b3db89]:http://www.jianshu.com/p/f6f357b3db89
[谈谈我对ThreadLocal的理解]:https://github.com/AerialLadder/StudyNotes/blob/master/Part1_Android/%E8%B0%88%E8%B0%88%E6%88%91%E5%AF%B9ThreadLocal%E7%9A%84%E7%90%86%E8%A7%A3.md
[http://www.jianshu.com/p/325d3cd79fd5]:http://www.jianshu.com/p/325d3cd79fd5
[http://www.jianshu.com/p/338cce832cc9]:http://www.jianshu.com/p/338cce832cc9
[http://www.jianshu.com/p/35c8567419fa]:http://www.jianshu.com/p/35c8567419fa