# iOS UI性能卡顿检测

# 文档

[](http://www.tanhao.me/code/151113.html/)

##  输出 Swift 工程里，编译时间超过 100 ms 的函数信息的命令：

```
xcodebuild -workspace XXX.xcworkspace -scheme XXX clean build OTHER_SWIFT_FLAGS="-Xfrontend -debug-time-function-bodies" | grep '^\d\{3,\}[.]\{1\}'

```

