#### Map我们经常使用,但是你真的会用吗?
这篇文章,我不是为了说明Map的实现原理是什么,我就是为了记录一个开发中遇到的问题,以及如何解决.

先来写一个简单的小demo:
```
public class TestMap {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<String, Integer>();
        map.put("111",111);
        map.put("222",222);
        map.put("333",333);

        System.out.println("args = [" + map.toString() + "]");
    }
}
```
运行结果我就不截图了
```
args = [{111=111, 222=222, 333=333}]

Process finished with exit code 0
```
就是这样的,没有什么特殊的,下面我们在来看一个demo:
```
public class TestMap {
    public static void main(String[] args) {
        System.out.println("args = [" + getInteger("111") + "]");
    }

    public static int getInteger(String key){
        Map<String, Integer> map = new HashMap<String, Integer>();
        map.put("111",111);
        map.put("222",222);
        map.put("333",333);

        return map.get(key);
    }
}
```
运行结果为:
```
args = [111]

Process finished with exit code 0
```
也没有什么问题,但是刚我们调用getInteger(String key)
的时候传入的参数在Map里面没有呢?
看下面这个demo:
```
public class TestMap {
    public static void main(String[] args) {
        System.out.println("args = [" + getInteger("444") + "]");
    }

    public static int getInteger(String key){
        Map<String, Integer> map = new HashMap<String, Integer>();
        map.put("111",111);
        map.put("222",222);
        map.put("333",333);

        return map.get(key);
    }
}
```
运行结果:
```
Exception in thread "main" java.lang.NullPointerException
	at TestMap.getInteger(TestMap.java:15)
	at TestMap.main(TestMap.java:6)

Process finished with exit code 1
```
这时候会报一个空指针异常,为什么会有空指针异常呢?
这里涉及到一个JDK1.5之后的[自动装箱拆箱机制][自动装箱拆箱机制].
原因大概可以这么说,看一段代码
```
    public static void main(String[] args) {
        Integer integer = 99;
        int i = integer;
        System.out.println(i);
    }
```

结果为:
```
99

Process finished with exit code 0
```
但是当代码改为这样:
```
public static void main(String[] args) {
    Integer integer = null;
    int i = integer;
    System.out.println(i);
}
```
结果为:
```
Exception in thread "main" java.lang.NullPointerException
	at TestMap.main(TestMap.java:7)

Process finished with exit code 1
```
其实就是拆箱失败,所以在Map使用的时候,返回值为基本数据类型的情况下,最后不要直接get().
下面给出一段提倡的使用方法:
```
public class TestMap {
    public static void main(String[] args) {
        /*Integer integer = null;
        int i = integer;
        System.out.println(i);*/

        System.out.println("args = [" + getInteger("444",-1) + "]");
    }

    public static int getInteger(String key,int defaultt){
        Map<String, Integer> map = new HashMap<String, Integer>();
        map.put("111",111);
        map.put("222",222);
        map.put("333",333);
        if (map.containsKey(key)){
            return map.get(key);
        }

        return defaultt;
    }
}
```
返回结果:
```
args = [-1]

Process finished with exit code 0
```
这样就当key不存在的时候就不会报空指针异常了.

在JDK1.8中已经有这样的方法了:
```
map.getOrDefault(key,defaultt);
看一下源码:
    default V getOrDefault(Object key, V defaultValue) {
        V v;
        return (((v = get(key)) != null) || containsKey(key))
            ? v
            : defaultValue;
    }
```
实现原理差不多,次方法在JDK1.8中才有.


[自动装箱拆箱机制]:http://blog.csdn.net/hp910315/article/details/48654777
