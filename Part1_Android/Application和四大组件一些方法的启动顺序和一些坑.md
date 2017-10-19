#### 本文总结自：[Android系统中的Application和四大组件一些方法的启动顺序和一些坑]

#### 总结：
1. Application构造方法比attachBaseContext方法优先执行；
2. ContentProvider的onCreate的方法比Application的onCreate的方法先执行（一定，静态注册）；
3. Activity、Service的onCreate方法以及BroadcastReceiver的onReceive方法，
   是在MainApplication的onCreate方法之后执行的；
4. 调用流程为：Application构造方法 ---> Application的attachBaseContext --->
   ContentProvider的onCreate----> Application的onCreate --->
   Activity、Service等的onCreate（Activity和Service不分先后）；
5. Application的onCreate方法 和 Provider的call方法 不是顺序执行，而是会同时执行。

#### “坑”
1. “坑”一：
    在Application的attachBaseContext方法中，使用了getApplicationContext方法。
    当我发现在attachBaseContext方法中使用getApplicationContext方法返回null时，内心是崩溃。
    所以，如果在attachBaseContext方法中要使用context的话，那么使用this吧，别再使用getApplicationContext()方法了。

2. “坑”二：
    这个其实不算很坑，也不会引起崩溃，但需要注意：
    在Application的attachBaseContext方法中，去调用自身的ContentProvider，
    那么这个ContentProvider会被初始化两次，也就是说这个ContentProvider会被两次
    调用到onCreate。如果你在ContentProvider的onCreate中有一些逻辑，那么一定要检
    查是否会有影响。




[Android系统中的Application和四大组件一些方法的启动顺序和一些坑]:http://blog.csdn.net/long117long/article/details/66477562