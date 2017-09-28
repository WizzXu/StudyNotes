今天我们分析的是ThreadLocal这个类（JDK1.8中。安卓中对这个类进行了改写，方法和原理都差不多，但是具体实现上有区别，对数据存储以及获取的方式进行了更改）

```
/**
* Implements a thread-local storage, that is, a variable for which each thread
* has its own value. All threads share the same {@codeThreadLocal} object,
* but each sees a different value when accessing it, and changes made by one
* thread do not affect the other threads. The implementation supports
* {@codenull} values.
*
*@seejava.lang.Thread
*@authorBob Lee
*/
```
这是JDK中的注释，英文好的可以翻译一下。我Google了一下，意思大概是：

> 实现一个线程本地的存储，也就是说，每个线程都有自己的局部变量。所有线程都共享一个ThreadLocal对象，但是每个线程在访问这些变量的时候能得到不同的值，每个线程可以更改这些变量并且不会影响其他的线程，并且支持null值。

我们先来看一下ThreadLocal类中的方法：
<center>![](https://github.com/AerialLadder/StudyNotes/blob/master/PIC/2017_9_21_1.jpg?raw=true)

其中有四个很重要的方法：
> * public void set(T value)设置当前线程的线程局部变量的值。
> * public T get()该方法返回当前线程所对应的线程局部变量。
> * public void remove()将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK 5.0新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。
> * protected T initialValue()返回该线程局部变量的初始值，该方法是一个protected的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次。ThreadLocal中的缺省实现直接返回一个null。

#### java.lang.ThreadLocal<T>的具体实现
* set()
```
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```  

在这个方法内部我们看到，首先通过getMap(Thread t)方法获取一个和当前线程相关的ThreadLocalMap，然后将变量的值设置到这个ThreadLocalMap对象中，当然如果获取到的ThreadLocalMap对象为空，就通过createMap方法创建。

线程隔离的秘密，就在于ThreadLocalMap(android中是Values这个内部类)这个类。ThreadLocalMap是ThreadLocal类的一个静态内部类，它实现了键值对的设置和获取（对比Map对象来理解），每个线程中都有一个独立的ThreadLocalMap副本，它所存储的值，只能被当前线程读取和修改。ThreadLocal类通过操作每一个线程特有的ThreadLocalMap副本，从而实现了变量访问在不同线程中的隔离。因为每个线程的变量都是自己特有的，完全不会有并发错误。还有一点就是，ThreadLocalMap存储的键值对中的键是this对象指向的ThreadLocal对象，而值就是你所设置的对象了。
```
    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
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
其实里面就是一个Entry，当前线程为键，存入的对象为值。

* get()
```
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```  

第一句是取得当前线程，然后通过getMap(t)方法获取到一个map，map的类型为ThreadLocalMap。然后接着下面获取到<key,value>键值对,<font color=#DC143C>注意这里获取键值对传进去的是this，而不是当前线程t</font>。
如果获取成功，则返回value值。
如果map为空，则调用setInitialValue方法返回value。
首先看一下getMap方法中做了什么：
```
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```  
在getMap中，是调用当期线程t，返回当前线程t中的一个成员变量threadLocals。
那么我们继续取Thread类中取看一下成员变量threadLocals是什么：
```
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
实际上就是一个ThreadLocalMap，这个类型是ThreadLocal类的一个内部类，我们继续去看ThreadLocalMap的实现：
```
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        ......
```
可以看到ThreadLocalMap的Entry继承了WeakReference，并且使用ThreadLocal作为键值。
然后再继续看setInitialValue方法的具体实现：
```
    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
很容易了解，就是如果map不为空，就设置键值对，为空，再创建Map，看一下createMap的实现：
```
    /**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the map
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
至此，可能大部分朋友已经明白了ThreadLocal是如何为每个线程创建变量的副本的：
首先，在每个线程Thread内部有一个ThreadLocal.ThreadLocalMap类型的成员变量threadLocals，这个threadLocals就是用来存储实际的变量副本的，键值为当前ThreadLocal变量，value为变量副本（即T类型的变量）。
初始时，在Thread里面，threadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的副本变量为value，存到threadLocals。  

然后在当前线程里面，如果要使用副本变量，就可以通过get方法在threadLocals里面查找。

```
public class Test {
	ThreadLocal<Long> longLocal = new ThreadLocal<Long>();
	ThreadLocal<String> stringLocal = new ThreadLocal<String>();

	public void set() {
		longLocal.set(Thread.currentThread().getId());
		stringLocal.set(Thread.currentThread().getName());
	}

	public long getLong() {
		return longLocal.get();
	}

	public String getString() {
		return stringLocal.get();
	}

	public static void main(String[] args) throws InterruptedException {
		final Test test = new Test();


		test.set();
		System.out.println(test.getLong());
		System.out.println(test.getString());


		Thread thread1 = new Thread(){
			public void run() {
				test.set();
				System.out.println(test.getLong());
				System.out.println(test.getString());
			};
		};
		thread1.start();
		thread1.join();

		System.out.println(test.getLong());
		System.out.println(test.getString());
	}
}
```
运行结果为:
![](https://github.com/AerialLadder/StudyNotes/blob/master/PIC/2017_9_21_2.png?raw=true)  
这个方法在主线程如果没有调用test.set()方法会报出一个空指针异常的的错误,经过分析,原因为MAP使用问题,请看我另一篇[MAP分析][MAP分析]

本文部分内容参考[Java并发编程：深入剖析ThreadLocal][Java并发编程：深入剖析ThreadLocal]


[MAP分析]:
[Java并发编程：深入剖析ThreadLocal]:http://www.cnblogs.com/dolphin0520/p/3920407.html
