# String
## 1. String 为什么设计成final类型的？
1. 缓存hashcode，String不可变，所以hashcode不变，这样缓存才有意义，不必重新计算。
2. 同一个字符串对象可以被多个线程共享，如果访问频繁的话，可以省略同步和锁等待的时间，从而提升性能。
3. 不可变对象，方便常量池缓存
特别要注意的是，**String类的所有方法都没有改变字符串本身的值，都是返回了一个新的对象**。

## 2. 常量池对象和新建对象

```
public static void main(String[] args) {
    String str1 = "HelloXWY";
    String str2 = "HelloXWY";
    String str3 = new String("HelloXWY");
    String str4 = "Hello";
    String str5 = "XWY";
    String str6 = "Hello" + "XWY";
    String str7 = str4 + str5;
    String str8 = new String("Hello") + new String("XWY");

    System.out.println("str1 == str2 result: " + (str1 == str2));

    System.out.println("str1 == str3 result: " + (str1 == str3));

    System.out.println("str1 == str6 result: " + (str1 == str6));

    System.out.println("str1 == str7 result: " + (str1 == str7));

    System.out.println("str1 == str7.intern() result: " + (str1 == str7.intern()));

    System.out.println("str3 == str3.intern() result: " + (str3 == str3.intern()));

    System.out.println("str1 == str8 result: " + (str1 == str8));

    System.out.println("str1 == str8.intern() result: " + (str1 == str8.intern()));
}

运行结果：
/Library/Java/JavaVirtualMachines/jdk1.8.0_271.jdk/Contents/Home/bin/java -javaagent:/Applications/IntelliJ IDEA 
str1 == str2 result: true
str1 == str3 result: false
str1 == str6 result: true
str1 == str7 result: false
str1 == str7.intern() result: true
str3 == str3.intern() result: false
str1 == str8 result: false
str1 == str8.intern() result: true
```
### 2.1 str1 == str2 result: true
```
String str1 = "HelloXWY";   
String str2 = "HelloXWY";
```

`""` 形式创建对象JVM会先查找常量池中是否存在该对象，有则返回常量池中该对象的引用。  
没有则会先在常量池中创建该数据的对象，在返回该对象的引用。  


当执行第一句时，JVM会先去常量池中创建对象并返回该对象的引用。  
当执行第二句时，直接返回常量池中对象的引用。

两者指向同一对象，结果为true
### 2.2 str1 == str3 result: false
```
String str3 = new String("HelloXWY");
```
`new String()`的形式创建对象会先在堆中创建一个String对象，再去常量池中找有没有要对应数据的对象。如果有，把池中对象的数据的引用传给堆中对象。如果没有则需要在常量池中先创建对应数据的对象，在进行引用传递。  
看下`new String()`源码就能理解了。  

```
/** The value is used for character storage. */
private final char value[];

/** Cache the hash code for the string */
private int hash; // Default to 0

public String(String original) {
    this.value = original.value;
    this.hash = original.hash;
}
```
str3指向了堆中对象，结果为false
### 2.3 str1 == str6 result: true 
```
String str6 = "Hello" + "XWY"; 
```
class文件反编译能看到
```
String str6 = "HelloXWY";
```
编译期自动优化了，所以情况跟2.2一样。结果为true
### 2.4 str1 == str7 result: false
```
String str7 = str4 + str5;
```
代码执行的时候，JVM会在堆（heap）中创建一个以str4为基础的一个StringBuilder对象，然后调用StringBuilder的append()方法完成与str5的合并，之后会调用toString()方法在堆（heap）中创建一个String对象，并把这个String对象的引用赋给str7。  
结果为false
### 2.5 其他
```
str1 == str7.intern() result: true
str3 == str3.intern() result: false
```
intern():  
判断这个常量是否存在于常量池。  
(1)如果存在  
(1.1)判断存在内容是引用还是常量   
(1.1.1)如果是引用，返回引用地址指向堆空间对象  
(1.1.2)如果是常量，直接返回常量池常量  
(2)如果不存在  
(2.1)将当前对象引用复制到常量池,并且返回的是当前对象的引用

### 小节

`""`直接操作常量池，有该对象就返回，没有就新建再返回  
`new Stirng()`堆上新建对象，内容先去常量池中找，有则取常量池中对象的内部数据引用。没有需要在常量池中创建对象，再取引用。
## 3. 字符串拼接
`+`底层采用StringBuilder进行了优化，但是在有些场景下比如for中会创建大量StringBuilder，能拼接任何类型数据。  
`StringBuilder`建议使用，内部有缓冲区。线程不安全。  
`StringBuffer`原理跟StringBuilder差不多。线程安全。存在效率问题。  
`concat`concat的计算效率要比+的效率高，比StringBuilder低，只能拼接String类型数据  

```
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true);
}
```

# 基本数据类型包装类的（Integer，Byte，Short，Long）的缓冲池
## Integer类范围 -128到127
```
/**
 * Cache to support the object identity semantics of autoboxing for values between
 * -128 and 127 (inclusive) as required by JLS.
 *
 * The cache is initialized on first usage.  The size of the cache
 * may be controlled by the {@code -XX:AutoBoxCacheMax=<size>} option.
 * During VM initialization, java.lang.Integer.IntegerCache.high property
 * may be set and saved in the private system properties in the
 * sun.misc.VM class.
 */

private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

## Long 类范围 -128到127
```
private static class LongCache {
    private LongCache(){}

    static final Long cache[] = new Long[-(-128) + 127 + 1];

    static {
        for(int i = 0; i < cache.length; i++)
            cache[i] = new Long(i - 128);
    }
}

/**
 * Returns a {@code Long} instance representing the specified
 * {@code long} value.
 * If a new {@code Long} instance is not required, this method
 * should generally be used in preference to the constructor
 * {@link #Long(long)}, as this method is likely to yield
 * significantly better space and time performance by caching
 * frequently requested values.
 *
 * Note that unlike the {@linkplain Integer#valueOf(int)
 * corresponding method} in the {@code Integer} class, this method
 * is <em>not</em> required to cache values within a particular
 * range.
 *
 * @param  l a long value.
 * @return a {@code Long} instance representing {@code l}.
 * @since  1.5
 */
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}
```

## Byte，Short 的缓存池范围默认都是: -128 到 127
Byte的所有值都在缓存区中，用它生成的相同值对象都是相等的。