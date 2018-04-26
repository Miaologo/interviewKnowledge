ReactNative 踩坑集锦

# 具体问题
1.在podfile文件中 加入pod ‘React’相关文件前，要先检查 platform :ios, '6.0’这个，如果是6.0会发现编译不过，报的错误也不是相关的，不容易查找。把它改成7.0，react只支持ios7以上。
 
2.在podfile文件中   pod ‘React’   pod 'React/RCTText’ ，pod 'React/RCTWebSocket'加入这三个就可以了，官方文档里写的加的比较多。参考了其它文档说只需加前两个但是会编译不过，所以加这三个。
 
3.文档说这么创建文件。$ mkdir ReactComponent $ touch ReactComponent/index.ios.js。 index.ios.js这个文件在ReactComponent文件里，发现会编译不过。给让ReactComponent，index.ios.js在同一目录下。
 
4  这里的.AppRegistry.registerComponent('SimpleApp', () => SimpleApp)， 这里SimpleApp要记得RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
moduleName: @"SimpleApp" launchOptions:nil]; 与这里的moduleName名字相同。这个我试的时候忽略了，编译不过了。
 
5。文档中 RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
moduleName: @"SimpleApp" launchOptions:nil];这个方法已经不存在了，要换成    RCTRootView *rootView = [[RCTRootView alloc]initWithBundleURL:jsCodeLocation moduleName:@"Elephant" initialProperties:nil launchOptions:nil];这个。
 
6.(JS_DIR=`pwd`/ReactComponent; cd Pods/React; npm run start -- --root $JS_DIR)，执行这个的时候要记得在项目的根目录下，要不也会报错

# 参考文档
[](http://www.jianshu.com/p/582e3031aa0c?nomobile=yes)

