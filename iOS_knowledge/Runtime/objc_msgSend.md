# objc_msgSend

http://www.jianshu.com/p/ecc681f09d59?utm_campaign=maleskine&utm_content=note&utm_medium=writer_share&utm_source=weibo

## 总结
Objective-C 中给一个对象发送消息会经过以下几个步骤：
1. 在对象类的dispatch table中尝试找到该消息。如果找到了，跳到对应的函数IMP去执行实现代码
2. 如果没有找到，Runtime会发送`+resolveInstanceMethod:`或者`+resolveClassMethod:`尝试去resolve这个消息
3. 如果resolve方法返回NO，Runtime就发送`-forwardingTargetForSelector:`允许你把这个消息转发给另一个对象
4. 如果没有新的目标对象返回，Runtime就是发送`-methodSignatureForSelector:`和`-forwardInvocation:`消息。你可以发送`-invokeWithTarget:`消息来手动转发消息或者发送`-doesNotRecognizeSelector:`抛出异常

## 参考文档

[iOS-RunLoop充满灵性的死循环](http://draveness.me/message/)
[](https://desgard.com/2016/08/07/objc_msgSend1/)
[Objective-C Runtime](http://tech.glowing.com/cn/objective-c-runtime/)

