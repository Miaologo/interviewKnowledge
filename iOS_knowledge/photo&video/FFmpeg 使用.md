# FFmpeg 使用

## FFmpeg 简介

FFMPEG的代码主要包含两个部分：
(1)library，
	library里大部分都是api，直接调用api来操作视频，需要写成c和c++
(2)tool
	tool就是把命令行转换为api的操作
	
## Mac OS 使用
（1）直接使用brew安装  `brew install ffmpeg`

## iOS 使用
[](http://www.jianshu.com/p/dfc708bbacd5)
将ffmpeg编译出相应的静态库或者动态库
按照appstore的需求，编译出来的包还必须支持arm64，["一键编译"的脚本](https://github.com/kewlbear/FFmpeg-iOS-build-script)
只有一个build-ffmpeg.sh脚本文件。在终端中转至脚本的目录，执行命令：
`./build-ffmpeg.sh`
FFmpeg-iOS是编译出来的库，里面有我们需要的.a静态库，一共有7个。
执行命令：
`lipo -info libavcodec.a`



