# +load VS +initialize

## +load
load方法会在每个类文件在第一次被连接的时候就会掉用，也就是说只要它存在就会被调用一次
load的执行顺序满足一下几条

* 执行子类的load之前，当superClass未加载时，先执行superClass的load方法
* 分类的load方法统一在最后执行
* 优先满足以上两条，在满足按照Compile Source的顺序执行load方法

注意，这里是（调用分类的`+load`方法也是如此）直接使用函数内存地址的方式`(*load_method)(cls, SEL_load)`; 对 +load 方法进行调用的，而不是使用发送消息`objc_msgSend`的方式。
这样的调用方式就使得`+load`方法拥有了一个非常有趣的特性，那就是子类、父类和分类中的`+load`方法的实现是被区别对待的。也就是说如果子类没有实现`+load`方法，那么当它被加载时 runtime 是不会去调用父类的`+load`方法的。同理，当一个类和它的分类都实现了`+load`方法时，两个方法都会被调用。

## +initialize
`+initialize`方法是在类或它的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用。也就是说`+initialize`方法是以懒加载的方式被调用的，如果程序一直没有给某个类或它的子类发送消息，那么这个类的`+initialize`方法是永远不会被调用的。那这样设计有什么好处呢？好处是显而易见的，那就是节省系统资源，避免浪费。
具体的代码实现在`objc-runtime-new.mm`中

` ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);`
runtime 使用了发送消息`objc_msgSend`的方式对 +initialize 方法进行调用。也就是说 `+initialize`方法的调用与普通方法的调用是一样的，走的都是发送消息的流程。换言之，如果子类没有实现`+initialize`方法，那么继承自父类的实现会被调用；如果一个类的分类实现了`+initialize`方法，那么就会对这个类中的实现造成覆盖。

因此，如果一个子类没有实现 +initialize 方法，那么父类的实现是会被执行多次的。有时候，这可能是你想要的；但如果我们想确保自己的 +initialize 方法只执行一次，避免多次执行可能带来的副作用时，具体可以这样实现
```
+ (void)initialize {
  if (self == [ClassName self]) {
    // ... do the initialization ...
  }
}
```

## 综述

| +load | +initialize
-----| -----| -----
调用时机 | 被添加到runtime时 | 收到第一条消息前，可能永远不掉用
调用顺序 | 父类->子类->分类 | 父类->子类
调用次数 | 1次 | 多次
是否需要显示掉用父类实现 | 否 | 否
是否继承父类的实现 | 否 | 是
分类中实现 | 类和分类的都会执行 | 覆盖类中的方法，只执行分类的实现
## 参考文档
[load 和 initialize 方法的执行顺序以及类和对象的关系](http://www.jianshu.com/p/9daec08ec370)
[iOS 程序 main 函数之前发生了什么](http://blog.sunnyxx.com/2014/08/30/objc-pre-main/)

