做过Android开发的应该都知道`Handler`，不管是开发还是面试都应该了解，今天简单总结一下。  
我只总结重点。

文章参考：  
[Android消息机制1-Handler(Java层)](http://gityuan.com/2015/12/26/handler-message-framework/)  
[Android消息机制2-Handler(Native层)](http://gityuan.com/2015/12/27/handler-message-native/)  

[select/poll/epoll对比分析](http://gityuan.com/2015/12/06/linux_epoll/)  
欢迎大家去阅读原作者文章

# Handler
## 1. 模型
-  **Message**：消息分为硬件产生的消息(如按钮、触摸)和软件生成的消息；
-  **MessageQueue**：消息队列的主要功能向消息池投递消息(`MessageQueue.enqueueMessage`)和取走消息池的消息(`MessageQueue.next`)；
-   **Handler**：消息辅助类，主要功能向消息池发送各种消息事件(`Handler.sendMessage`)和处理相应消息事件(`Handler.handleMessage`)；
-  **Looper**：不断循环执行(`Looper.loop`)，按分发机制将消息分发给目标处理者。

## 2. 重点API
#### 2.1 sendEmptyMessage

```
public final boolean sendEmptyMessage(int what) {
    return sendEmptyMessageDelayed(what, 0);
}
```

#### 2.2 sendEmptyMessageDelayed

```
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}
```

#### 2.3 sendMessageDelayed

```
public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

#### 2.4 sendMessageAtTime

```
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
```

#### 2.5 sendMessageAtFrontOfQueue

```
public final boolean sendMessageAtFrontOfQueue(Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}
```
该方法通过设置消息的触发时间为0，从而使Message加入到消息队列的队头。

#### 2.6 post

```
public final boolean post(Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```

#### 2.7 postAtFrontOfQueue

```
public final boolean postAtFrontOfQueue(Runnable r) {
    return sendMessageAtFrontOfQueue(getPostMessage(r));
}
```

#### 2.8 enqueueMessage

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

#### 小节
`Handler.sendEmptyMessage()`等系列方法最终调用`MessageQueue.enqueueMessage(msg, uptimeMillis)`，将消息添加到消息队列中，其中uptimeMillis为系统当前的运行时间，不包括休眠时间。  
`sendMessageAtFrontOfQueue`和`postAtFrontOfQueue`两个当法的设置的触发时间为`0`，所以消息会在最前面，其余的方法传递的触发时间为`系统当前的运行时间`

## 3. 消息阻塞和唤醒
在MessageQueue中的native方法如下：

```
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

#### 3.1 nativeInit()
该方法会在native层创建NativeMessageQueue，NativeMessageQueue创建的过程中又会创建native层Looper。  
该方法会返回一个long型数值，该值由native层创建的NativeMessageQueue强转而来，其实就代表了native层创建的NativeMessageQueue。  
调用epoll的epoll_create()/epoll_ctl()来完成对mWakeEventFd和mRequests的可读事件监听

#### 3.2 nativeDestroy()
NativeMessageQueue移除强引用和若引用，释放资源。

### 3.3 nativePollOnce()
该方法最终调用`epoll_wait`用于等待事件发生或者超时

```
//等待事件发生或者超时，在nativeWake()方法，向管道写端写入字符，则该方法会返回； 
int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);
```
对于epoll_wait()返回，当且仅当以下3种情况出现：

-   POLL_ERROR，发生错误，直接跳转到Done；
-   POLL_TIMEOUT，发生超时，直接跳转到Done；
-   检测到管道有事件发生，则再根据情况做相应处理：
    -   如果是管道读端产生事件，则直接读取管道的数据；
    -   如果是其他事件，则处理request，生成对应的reponse对象，push到reponse数组；

### 3.4 nativeWake()
```
// 向管道mWakeEventFd写入字符1 
ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
```
通过向管道写字符1，让epoll_wait()返回
## 4. IdleHandler
IdleHandler是当Msg队列为空的时候执行的一种任务队列。代码逻辑仔```MessageQueue next()```中，看下源码

```
// 添加的所有IdleHandler
private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
// 每次循环的时候要操作的IdleHandler
private IdleHandler[] mPendingIdleHandlers;

Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    // 1.初始化的时候，定义的mPendingIdleHandlers的默认数量，后面会重置
    int pendingIdleHandlerCount = -1; 
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            // 2. 如果运行到这里，证明 Msg 队列为空或者 Msg 要延时执行
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                // 3. pendingIdleHandlerCount设置为程序中设置的所有的IdleHandler的数量
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            // 4. 将所有的 IdleHandler 转换为数组，进行本次循环
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        // 5.进行本次循环
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                // 6. 获取循环结果，是否需要下次执行
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                // 7. 如果不需要下次执行，在 mIdleHandlers 中删除这一条任务
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        // 8. 重置pendingIdleHandlerCount值
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

# select/poll/epoll

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。)

**epoll优势**

1.  监视的描述符数量不受限制，所支持的FD上限是最大可以打开文件的数目，具体数目可以`cat /proc/sys/fs/file-max`查看，一般来说这个数目和系统内存关系很大，以3G的手机来说这个值为20-30万。
1.  IO性能不会随着监视fd的数量增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现的，只有就绪的fd才会执行回调函数。

如果没有大量的空闲或者死亡连接，epoll的效率并不会比select/poll高很多。但当遇到大量的空闲连接的场景下，epoll的效率大大高于select/poll。

# ThreadLocal
### 使用场景：
**典型场景1：**  每个线程需要一个独享的对象（通常是工具类，典型需要使用的类有SimpleDateFormat和Random）

**典型场景2：**  每个线程内需要保存全局变量（例如在拦截器中获取用户信息），可以让不同方法直接使用，避免参数传递的麻烦。

### 原理
> ThreadLocal有一个内部类ThreadLocalMap作为存储结构，  
> ThreadLocalMap内部有一个内部类Entry作为最终的KV存储结构，  
> ThreadLocalMap内部有一个Entry数组保存Entry  
>   
> Thread类内部会持有一个ThreadLocalMap存储数据，每个对象一份，所以能做到线程隔离
> 
> ThreadLocal本质作为一个工具类和key,本身作为key，要存储的数据作为value，并将KV存储到调用线程内部持有的ThreadLocalMap中，完成数据存储和线程隔离

### 代码
#### 1. Thread类
```
public
class Thread implements Runnable {
    ...

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
```

### 2. ThreadLocalMap
```
static class ThreadLocalMap {


    /*
     * Entry继承WeakReference，并且用ThreadLocal作为key.
     * 如果key为null(entry.get() == null)，意味着key不再被引用，
     * 因此这时候entry也可以从table中清除。
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
    
    ```
    /**
     * 初始容量 —— 必须是2的整次幂
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * 存放数据的table，Entry类的定义在下面分析
     * 同样，数组长度必须是2的整次幂。
     */
    private Entry[] table;

    /**
     * 数组里面entrys的个数，可以用于判断table当前使用量是否超过阈值。
     */
    private int size = 0;

    /**
     * 进行扩容的阈值，表使用量大于它的时候进行扩容。
     */
    private int threshold; // Default to 0
}
```

### 3. ThreadLocal set方法
```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);//（3.1）
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### 3.1 ThreadLocalMap.set
```
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    // 计算哈希，保持数据均匀分布
    int i = key.threadLocalHashCode & (len-1);

    /**
     * ThreadLocalMap使用`线性探测法`来解决哈希冲突的。
     * 该方法一次探测下一个地址，直到有空的地址后插入，若整个空间都找不到空余的地址，则产生溢出。  
     * 举个例子，假设当前table长度为16，也就是说如果计算出来key的hash值为14，如果  
     * table[14]上已经有值，并且其key与当前key不一致，那么就发生了hash冲突，这个时候将14加1得到15，  
     * 取table[15]进行判断，这个时候如果还是冲突会回到0，取table[0],以此类推，直到可以插入。  
     * 按照上面的描述，可以把Entry[] table看成一个环形数组。
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        // key为 null，但是值不为 null，说明之前的 ThreadLocal 对象已经被回收了，
        // 当前数组中的 Entry 是一个陈旧（stale）的元素
        if (k == null) {
            //用新元素替换陈旧的元素，这个方法进行了不少的垃圾清理动作，防止内存泄漏
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    /**
     * cleanSomeSlots用于清除那些e.get()==null的元素，
     * 这种数据key关联的对象已经被回收，所以这个Entry(table[index])可以被置null。
     * 如果没有清除任何entry,并且当前使用量达到了负载因子所定义(长度的2/3)，那么进行
     * rehash（执行一次全表的扫描清理工作）
     */
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}


private final int threadLocalHashCode = nextHashCode();

//AtomicInteger是一个提供原子操作的Integer类，通过线程安全的方式操作加减,适合高并发情况下的使用
private static AtomicInteger nextHashCode =
    new AtomicInteger();

//`HASH_INCREMENT = 0x61c88647`,这个值跟斐波那契数列（黄金分割数）有关，其主要目的就是为了让哈希码能均匀的分布在2的n次方的数组里, 也就是Entry[] table中，这样做可以尽量避免hash冲突。
private static final int HASH_INCREMENT = 0x61c88647;

/**
 * Returns the next hash code.
 */
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}


private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

### 3. ThreadLocal get方法
```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);//（0）
        // 拿到数据，直接返回
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // 否则进行初始化，返回初始化的值，默认为null
    return setInitialValue();//（1）
}

//（0）
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 当前key直接命中
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);// （0.1）
}

// （0.1）
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    // 查找和清理仔查找
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}

//（1）
private T setInitialValue() {
    T value = initialValue();//（2）
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
//（2）
protected T initialValue() {
    return null;
}

```
### 难点：
- 内存泄漏怎么解决
    - 原因是在下一次set数据的时候才会进行`replaceStaleEntry`进行空value的清除
    - 建议在使用完手动remove掉数据

- Hash冲突怎么解决 
    - 线性探测法

参考 ： 
- https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/multi-thread/%E4%B8%87%E5%AD%97%E8%AF%A6%E8%A7%A3ThreadLocal%E5%85%B3%E9%94%AE%E5%AD%97.md

- https://zhuanlan.zhihu.com/p/167937566




