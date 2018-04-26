WWDC2016-406 Optimizing App Startup Time

# 优化App的启动时间
 -- 主要为406的视频内容

理论：
* `main()`函数执行前发生了什么
* `Mach-O`格式
* 虚拟内存基础
* Mach-O二进制文件的加载和准备事项
实践：
* 测量App启动时间
* 优化启动时间

## Mach-O术语

文件类型：
* Executable： 应用的主要二进制文件
* Dylib: 动态链接库（或者是DSO or DLL）
* Bundle: 不能被链接的Dylib，只能在运行时使用`dlopen()`加载的，如 plug-ins

Image： 是一种Executable(或Dylib，Bundle)文件
Framework：包含resources，headers和Dylib的文件夹

## Mach-O Image File
Mach-O文件被划分成一些segements，segment的名字均是大写，每个segement又被划分成一些sections
每个segment的大小为page的整数倍，具体页的大小和硬件相关（arm64为16KB，其他的为4KB）
section的大小没有是page的整数倍大小的限制，但section之间不会有重叠区域
Mach-O文件基本都包含三个segments，分别对应`__TEXT`，`__DATA`，`__LINKEDIT`
* `_TEXT`包含Mach header，code和只读的一些常量，只读可执行（-rx）
* `_DATA`
* `_LINKEDIT`

# 参考文档
[优化 App 的启动时间](http://yulingtianxia.com/blog/2016/10/30/Optimizing-App-Startup-Time/)
[iOS App从点击到启动](http://www.jianshu.com/p/231b1cebf477)
http://ios.jobbole.com/90498/
http://blog.sunnyxx.com/2014/08/30/objc-pre-main/

[由App的启动说起](http://oncenote.com/2015/06/01/How-App-Launch/)
http://www.daizi.me/2016/01/05/iOS%20%E5%90%AF%E5%8A%A8%E6%97%B6%E4%BC%98%E5%8C%96%20(1)/
[](http://www.cnphp6.com/archives/760630)

[iOS Dynamic Framework 对 App 启动时间影响实测](http://ios.jobbole.com/90934/)
[iOS瘦身之删除无用的mach-O文件](http://mp.weixin.qq.com/s?__biz=MzA3ODg4MDk0Ng==&mid=2651112096&idx=1&sn=ce8fccce7d5f70e30c078e63e8ea0d15&scene=21#wechat_redirect)

[Macho文件浏览器---MachOView](http://www.jianshu.com/p/175925ab3355)
[dyld中mach-o文件加载的简单分析](http://turingh.github.io/2016/03/01/dyld%E4%B8%ADmacho%E5%8A%A0%E8%BD%BD%E7%9A%84%E7%AE%80%E5%8D%95%E5%88%86%E6%9E%90/)
[了解iOS上的可执行文件和Mach-O格式](http://www.cocoachina.com/mac/20150122/10988.html)
[The Nitty Gritty of "Hello World" on OS X](http://www.reinterpretcast.com/hello-world-mach-o)

[iOS 开发中的『库』(一)](http://www.jianshu.com/p/48aff237e8ff)
[Slow App Startup Times](http://useyourloaf.com/blog/slow-app-startup-times/)


[dylib浅析](http://makezl.github.io/2016/06/27/dylib/)
[iOS 程序 main 函数之前发生了什么](http://blog.sunnyxx.com/2014/08/30/objc-pre-main/)

[mach-o格式浅析(二)](http://www.cnblogs.com/tieyan/p/4462691.html)

[动态库的加载和移除添加监听回调](https://github.com/ddeville/ImageLogger/blob/master/Shared/LLImageLogger.m)

[Objective-C 深入理解 +load 和 +initialize](http://www.jianshu.com/p/872447c6dc3f)
[Objective-C +load vs +initialize](http://blog.leichunfeng.com/blog/2015/05/02/objective-c-plus-load-vs-plus-initialize/)

http://stackoverflow.com/questions/24052386/does-swift-compile-to-native-code
https://untitledkingdom.co/blog/obj-c-vs-swift/
http://arstechnica.com/apple/2014/10/os-x-10-10/22/

[iOS编译过程的原理和应用](http://www.kuqin.com/shuoit/20161216/353174.html)

