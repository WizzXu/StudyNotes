Dart中空安全下的单例模式网上的文章好少，一搜一堆都是以前的非空安全的写法，今天我来写一个空安全下的单例。其实很简单，只是新手有可能想不到。  
老手请绕路。

```
class Single{
  static Single? _single;
  Single._();
  
  static Single getInstance(){
    return _single ??= Single._();
  }
}
```
测试代码：

```
void main() {
  print(Single.getInstance().hashCode);
  print(Single.getInstance().hashCode);
  print(Single._single.hashCode);
}
```
测试结果：

```
213875923 
213875923 
213875923
```

证明没问题

上个图片证明一下

![https://gitee.com/wizzxu/oss/raw/master/picraw/uPic/空安全下的Dart、Flutter单例模式要怎么写.png](https://gitee.com/wizzxu/oss/raw/master/picraw/uPic/空安全下的Dart、Flutter单例模式要怎么写.png)