# HashMap
本文源码除特殊说明，默认指JDK1.8  
## 源码解析可以参考：[HashMap 源码详细分析(JDK1.8)](https://segmentfault.com/a/1190000012926722)
## 1. 实现方式 
JDK1.7  **哈希表 + 链表**  
JDK1.8  **哈希表 + 链表/红黑树**
## 2. 基本参数
```
/**
 * 默认的初始化长度 - 必须是2的n次方.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * 最大容量 1<<30 2^30=1073741824
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 负载因子：默认值为`0.75`。 当元素的总个数>当前数组的长度 * 负载因子。
 * 数组会进行扩容，扩容为原来的两倍
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * 链表树化阙值： 默认值为 `8` 。表示在一个node（Table）节点下的值的个数大于8时候，
 * 会将链表转换成为红黑树。
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * 默认值为 `6` 。 表示在进行扩容期间，单个Node节点下的红黑树节点的个数小于6时候，
 * 会将红黑树转化成为链表。
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * 最小树化阈值，当Table所有元素超过改值，才会进行树化（为了防止前期阶段频繁扩容和树化过程冲突）
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```
## 3. 基本方法-hash()
```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
在JDK 1.8中hash方法计算的是做一次16位右位移异或混合。网上也有说将hashcode高低十六位异或，那应该是JDK 1.7的实现实现形式。不同版本有不同的方式。  
这部分可以参考一下：[JDK 源码中 HashMap 的 hash 方法原理是什么？](https://www.zhihu.com/question/20733617)
## 3. 基本方法-get()
```
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        //1. 首先根据key的hash后的值获取在table中的位置，方式为hash的值
        // 与当前table的长度进行 & 运算，高效
        (first = tab[(n - 1) & hash]) != null) {
        //2. 先检查第一个节点，检查方式为先对比hash后的值再比较key的内存地址或者equals比较
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 3. 判断第一个节点是不是TreeNode（红黑树），是的话就从红黑树中查找
            // 红黑树的查找就不分析了，感兴趣的可以深入研究
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 4. 不是红黑树就是链表，循环查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```
### 3.1 如何比较两个key？(重点)
`先对比hash后的值再比较key的内存地址或者equals比较`

## 4. 基本方法-put()
```
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1. 如果table为空或者table是一个空数组的话，会对table进行扩容操作，
    // 扩容的讲解放在了 第 5 小节
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 2. 根据key hash后的值与当前table的长度进行 & 运算后获取要存放的桶位置
    // 如果当前位置没有数据则创建一个节点放到当前位置
    // 否则进行下一步
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 3. 如果key与当前节点比较相同，则用 e 指向当前节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 4. 否则如果第一个节点是红黑树，则向红黑树中插入或者替换节点值。
        // 如果是插入，会返回一个null，如果是替换，就会返回老的节点，
        // 并由e指向它
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 5. 否则就是链表结构
        else {
            for (int binCount = 0; ; ++binCount) {
                // 6. 如果遍历到了末尾，则插入一个该K-V生成的节点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 7. 插入后，如果当前大小大于等于8个，就转化成红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 8. 如果当前节点的key就是要插入的key值，找到了这个点并且由 e 指向了它
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                //
                p = e;
            }
        }
        // 9. e此时指向的就是要操作的节点
        // 不过有的时候是新增节点的场景，有的时候是要修改节点的场景
        // 如果是新增节点的场景，e会是null
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // 10. onlyIfAbsent 表示是否仅在 oldValue 为 null 的情况下更新键值对的值
            // 默认为 false
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 这是一个空方法，可以重写
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // HashMap修改的次数
    ++modCount;
    // 10. 如果走到这里，证明e是新增节点的场景，会将当前size + 1
    // 如果插入后的size大于阈值，进行resize操作
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    // 11. 新增节点的场景返回null
    return null;
}
```
## 5. 基本方法-resize()
```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 1. 判断table是否已经初始化过
    if (oldCap > 0) {
        // 2. 容量达到最大值，不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 3. 如果容量X2后小于最大容量并且老的容量大于默认容量，阈值X2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 4. 如果没有初始化，并且老的阈值大于0，直接用老的阈值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 5. 如果没有初始化，老的阈值不大于0，设置新的容量为默认值，
    // 新的阈值为 默认大小 * 默认负载因子
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 6. 新的阈值为0的话
    if (newThr == 0) {
        // 7. 计算一个阈值 新的容量 * 当前设置的负载因子（没有设置的话是默认值）
        float ft = (float)newCap * loadFactor;
        // 8. 如果新的容量小于HashMap支持的最大值，
        // 并且计算出来的阈值小于HashMap支持的最大值
        // 新的阈值为计算出来的阈值，否则为Int的最大值
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 7. 设置阈值为经过一系列计算出来的阈值
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 8. 如果旧的table不为空，进行遍历
        // 并且将旧的table没一位的数据置空
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 9. 如果没有链表和树的结构，重新计算hash值并且重新计算在桶的位置，
                // 放在相应位置上
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 10. 如果是树的结构，对树进行拆分，在这个过程中，如果树小于6是会转成链表的
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                // 11. 链表结构,重新根据hash值和桶大小计算位置，
                // 并将要移动位置的节点和不移动位置的节点分别放在两个链表中
                // 最后将两个链表分别放在桶的相应位置上
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

关于resize这部分，可以深入看看，推荐：  
[深入理解HashMap(四): 关键源码逐行分析之resize扩容](https://segmentfault.com/a/1190000015812438)

## 6. 常见问题汇总
写到这里，最重要的三个方法就分析完了，剩下的几个方法可以参考推荐文章再细看吧。


### 6.1 如何让HasnMap有序？
使用 LinkedHashMap ，他的实现原理为节点同时保存了一个向前的节点的引用，节点既有前面节点的引用，又有后面节点的引用，是一个链表结构了，具体的可以参考：https://blog.csdn.net/justloveyou_/article/details/71713781

### 6.2 如何让HashMap线程安全？
1. 使用HashTable，不过性能差，不推荐使用。
2. 使用Collections.synchronizedMap(map)包装map，返回 SynchronizedMap 其实就是内部持有一个Map，然后对Map的各方法利用synchronized包装。性能也不是很好，不推荐使用。
3. 使用ConcurrentHashMap。
#### 6.2.1 ConcurrentHashMap
可以参考：[HashMap? ConcurrentHashMap? 相信看完这篇没人能难住你！](https://crossoverjie.top/2018/07/23/java-senior/ConcurrentHashMap/)  

`JDK 1.7的实现`  
由Segment数组、HashEntry组成。  
理解起来就是Segment就是一个改造版的HashMap。  
put数据的时候先根据key->hash, 根据计算出来的下标找到在Segment中的位置，在往Segment中put数据。  
往Segment中put数据跟往HashMap中put数据流程一样。这样相当于把桶上以前的链表放到了Segment中，Segment在put数据的时候会加锁。  
这样把锁的力度控制在了以前table的每个节点的范围，缩小了范围，提高了效率。  
get数据的时候不需要加锁。  
`JDK 1.8的实现` 
抛弃了Segment结构，它的结构跟HashMap的一样。  
不过在put数据的时候，会先查找该key是否有对应节点。  
如果没有，利用CAS的方式插入节点。  
如果有，利用 synchronized 加锁，在插入数据。
### 6.3 HashMap允许空键空值么
HashMap最多只允许一个键为Null(多条会覆盖)，但允许多个值为Null
### 6.4 HashMap的key一般怎么选？用可变类当HashMap的key有什么问题?
**一般用Integer、String这种不可变类当HashMap当key，而且String最为常用。**
-   (1)因为字符串是不可变的，所以在它创建的时候hashcode就被缓存了，不需要重新计算。这就使得字符串很适合作为Map中的键，字符串的处理速度要快过其它的键对象。这就是HashMap中的键往往都使用字符串。
-   (2)因为获取对象的时候要用到equals()和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的,这些类已经很规范的覆写了hashCode()以及equals()方法。
**用可变类当HashMap的key,该对象的hashcode可能发生改变，导致put进去的值，无法get出。**

### 6.5 如果让你实现一个自定义的class作为HashMap的key该如何实现？

此题考察两个知识点

-   重写hashcode和equals方法注意什么?
-   如何设计一个不变类

**针对问题一，记住下面四个原则即可**

(1)两个对象相等，hashcode一定相等 (2)两个对象不等，hashcode不一定不等 (3)hashcode相等，两个对象不一定相等 (4)hashcode不等，两个对象一定不等

**针对问题二，记住如何写一个不可变类**

(1)类添加final修饰符，保证类不被继承。 如果类可以被继承会破坏类的不可变性机制，只要继承类覆盖父类的方法并且继承类可以改变成员变量值，那么一旦子类以父类的形式出现时，不能保证当前类是否可变。

(2)保证所有成员变量必须私有，并且加上final修饰 通过这种方式保证成员变量不可改变。但只做到这一步还不够，因为如果是对象成员变量有可能再外部改变其值。所以第4点弥补这个不足。

(3)不提供改变成员变量的方法，包括setter 避免通过其他接口改变成员变量的值，破坏不可变特性。

(4)通过构造器初始化所有成员，进行深拷贝(deep copy) 如果构造器传入的对象直接赋值给成员变量，还是可以通过对传入对象的修改进而导致改变内部变量的值。例如：

```
public final class ImmutableDemo {  
    private final int[] myArray;  
    public ImmutableDemo(int[] array) {  
        this.myArray = array; // wrong  
    }  
}
```

这种方式不能保证不可变性，myArray和array指向同一块内存地址，用户可以在ImmutableDemo之外通过修改array对象的值来改变myArray内部的值。 为了保证内部的值不被修改，可以采用深度copy来创建一个新内存保存传入的值。正确做法：

```
public final class MyImmutableDemo {  
    private final int[] myArray;  
    public MyImmutableDemo(int[] array) {  
        this.myArray = array.clone();   
    }   
}
```

(5)在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝 这种做法也是防止对象外泄，防止通过getter获得内部可变成员对象后对成员变量直接操作，导致成员变量发生改变。

### 6.6 重写equals()方法，为什么也需要重写hashcode()方法?跟HashMap有关系吗?为什么？

<https://www.huaweicloud.com/articles/1a14ad369ac1a5b104c865cf1b8bb702.html>

Java集合类大量使用了hashcode和equals，如果不同时重写在使用集合类的时候会出现问题。   

跟HashMap有关系，或者说因为HashMap中用到了对象的hashcode方法所以会有关系，因为我们如果在设计两个对象相等的逻辑时，如果只重写Equals方法，那么一个类有两个对象A1，A2，他们的A1.equals(A2)为true，A1.hashcode和A2.hashcode不一样，当将A1和A2都作为HashMap的key时，HashMap会认为它两不相等，因为
HashMap在判断key值相不相等时会判断key的hashcode是不是一样，hashcode一样才进行equals判断，所以会有问题。
### 6.7 JDK 1.8中做了哪些优化优化？
-   `数组+链表改成了数组+链表或红黑树`
-   `链表的插入方式从头插法改成了尾插法`
-   `扩容的时候1.7需要对原数组中的元素进行重新hash定位在新数组的位置，1.8采用更简单的判断逻辑，位置不变或索引+旧容量大小；`
-   在插入时，1.7先判断是否需要扩容，再插入，1.8先进行插入，插入完成再判断是否需要扩容；
### 6.8 为什么HashMap的底层数组长度为何总是2的n次方
- 第一：当length为2的N次方的时候，h & (length-1) = h % length
为什么&效率更高呢？因为位运算直接对内存数据进行操作，不需要转成十进制，所以位运算要比取模运算的效率更高
- 第二：当length为2的N次方的时候，数据分布均匀，减少冲突
### 6.9 rehash的解释：
在创建hashMAP的时候可以设置来个参数，一般默认
初始化容量：创建hash表时桶的数量
负载因子：负载因子=map的size/初始化容量
当hash表中的负载因子达到负载极限的时候，hash表会自动成倍的增加容量（桶的数量），并将原有的对象重新的分配并加入新的桶内，这称为rehash。这个过程是十分好性能的，一般建议设置比较大的初始化容量，防止rehash，但是也不能设置过大，初始化容量过大浪费空间。

