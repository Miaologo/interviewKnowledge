# RAC中宏定义weakify和strongify

在Objective-C中使用RAC时，使用Block时为了保证不造成变量的循环引用使用到了两个宏定义`weakify`和`strongify`。你是否有疑问这个宏定义为什么要加个`@`，它们又是具体怎么实现的呢？
接下来我们将具体剖析这两个宏定义
[YYKit](https://github.com/ibireme/YYKit)也定义了这两个宏，但是它们稍微的简单点。

两个宏定义`weakify`和`strongify`中均适用到相同的几个宏定义，再次先剖析下作为预备知识

## rac_keywordify
查看在最新的RAC4.1.0中已经更换掉这个宏，具体从哪个版本开始做更换未做详细查看。

```
#if DEBUG
#define rac_keywordify autoreleasepool {}
#else
#define rac_keywordify try {} @catch (...) {}
#endif
```
宏`rac_keywordify`中定义了异常捕获语句，这段宏定义的代码开头少了个`@`，因此`weakify`和`strongify`前面必须加上`@`

## metamacro_foreach_cxt

```
#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
        metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(__VA_ARGS__))(MACRO, SEP, CONTEXT, __VA_ARGS__)
```
`metamacro_foreach_cxt`中是由几个宏定义组成，最外层宏`metamacro_concat`，和两个参数成员`metamacro_foreach_cxt`与`metamacro_argcount`组成

1、先来看看`metamacro_concat`

```
#define metamacro_concat(A, B) \
        metamacro_concat_(A, B)
        
#define metamacro_concat_(A, B) A ## B
```
`metamacro_concat`是用来连接两个参数的宏定义，解析这个宏之后上面的宏定义为：

```
#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
        metamacro_foreach_cxt##metamacro_argcount(__VA_ARGS__)(MACRO, SEP, CONTEXT, __VA_ARGS__)
```
2、接下来看看宏`metamacro_argcount`

```
#define metamacro_argcount(...) \
        metamacro_at(20, __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
```
`metamacro_argcount`宏是用来获取参数的个数。具体实现看下面分析。

```
#define metamacro_at(N, ...) \
        metamacro_concat(metamacro_at, N)(__VA_ARGS__)
```
`metamacro_at`是用来连接参数的，拼接metamacro_at##N(传入的第一个值，这里是20)(VA_ARGS) 也就是，解析这个宏之后，宏`metamacro_argcount`为

```
#define metamacro_argcount(...) \
        metamacro_at20( __VA_ARGS__, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1)
```
`metamacro_at20`的具体定义为：

```
#define metamacro_at20(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, ...) metamacro_head(__VA_ARGS__)
```
`metamacro_at20`会截取前面的20个参数，剩下的作为`metamacro_head`的参数。`__VA_ARGS__`中含有`N`个参数，剩下的参数就为最后的`N,N-1,...2,1`
因此解析宏`metamacro_at20`之后，宏`metamacro_argcount`为

```
#define metamacro_argcount(...) \
        metamacro_head( N,... 2, 1)
```
宏`metamacro_head`定义为：

```
#define metamacro_head(...) \
        metamacro_head_(__VA_ARGS__, 0)

#define metamacro_head_(FIRST, ...) FIRST
```
示例：

```
metamacro_argcount(NSObject, version)

NSObject对应_0
version对应_1
20对应_2
...
3对应_19

还剩(2,1)就是metamacro_head(__VA_ARGS__)的参数
所以
metamacro_at20(NSObject, version, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1);

metamacro_at20(_0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19, ...) metamacro_head(2,1)

```
3、最后`metamacro_foreach_cxt`

```
#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
        metamacro_foreach_cxt##metamacro_argcount(__VA_ARGS__)(MACRO, SEP, CONTEXT, __VA_ARGS__)
//变成了，其中`N`为参数个数
#define metamacro_foreach_cxt(MACRO, SEP, CONTEXT, ...) \
        metamacro_foreach_cxtN(MACRO, SEP, CONTEXT, __VA_ARGS__)
```

示例：@weakify(self)
则最终得到的语句为`metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, self)`
`metamacro_foreach_cxt1`宏定义如下：

```
#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)
得到
rac_weakify_(0, __weak, self)
```
当N＝20的时候怎么实现

```
#define metamacro_foreach_cxt20(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18, _19) \
    metamacro_foreach_cxt19(MACRO, SEP, CONTEXT, _0, _1, _2, _3, _4, _5, _6, _7, _8, _9, _10, _11, _12, _13, _14, _15, _16, _17, _18) \
    SEP \
    MACRO(19, CONTEXT, _19)
```
从这里看有点类似递归，先实现`MACRO(19, CONTEXT, _19)`，剩下前面19个参数，将它们传入`metamacro_foreach_cxt19`，而`metamacro_foreach_cxt19`会先实现`MACRO(18, CONTEXT, _18)`，再把剩下的18个参数传给`metamacro_foreach_cxt18`，如果此类推，...到`metamacro_foreach_cxt1`，最后到`metamacro_foreach_cxt0`，而`metamacro_foreach_cxt0`是保护机制，看看其定义如下：

```
#define metamacro_foreach_cxt0(MACRO, SEP, CONTEXT)
```
`metamacro_foreach_cxt0`不做任何操作

## weakify

```
RAC2.5
#define weakify(...) \
    rac_keywordify \
    metamacro_foreach_cxt(rac_weakify_,, __weak, __VA_ARGS__)

RAC4.1
#define weakify(...) \
    try {} @finally {} \
    metamacro_foreach_cxt(mtl_weakify_,, __weak, __VA_ARGS__)
```
RAC2.5中宏`weakify`中包括两个宏定义，`rac_keywordify`和`metamacro_foreach_cxt`


`weakify(...)`的初始定义中`metamacro_foreach_cxt(rac_weakify_,, __weak, __VA_ARGS__)`传了对应的参数。
因此得到

```
metamacro_foreach_cxt1(rac_weakify_,, __weak,self) rac_weakify_(0, __weak, self)
```
`rac_weakify_`定义如下，

```
#define rac_weakify_(INDEX, CONTEXT, VAR) \
    CONTEXT __typeof__(VAR) metamacro_concat(VAR, _weak_) = (VAR);
```
INDEX是第几个的标记，在这里没用。以`@weakify(self)`为例，

```
rac_weakify_(0, __weak, self)
变成了
__weak __typeof_(self) self_weak_ = self;
```

## strongify

```
#define strongify(...) \
    rac_keywordify \
    _Pragma("clang diagnostic push") \
    _Pragma("clang diagnostic ignored \"-Wshadow\"") \
    metamacro_foreach(rac_strongify_,, __VA_ARGS__) \
    _Pragma("clang diagnostic pop")
    
#define strongify(...) \
    try {} @finally {} \
    _Pragma("clang diagnostic push") \
    _Pragma("clang diagnostic ignored \"-Wshadow\"") \
    metamacro_foreach(mtl_strongify_,, __VA_ARGS__) \
    _Pragma("clang diagnostic pop")
```
### _Pragma语句
`_Pragma`就是`#pragma xxx xxx`，是几个clang语句，其作用为忽略当一个局部变量或类型声明遮盖另一个变量的警告
具体详细信息可以参考[fuckingclangwarnings](http://fuckingclangwarnings.com/)

```
_Pragma("clang diagnostic push")
_Pragma("clang diagnostic ignored \"-Wshadow\"")
_Pragma("clang diagnostic pop")
转换过来就是
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wshadow"
#pragma clang diagnostic pop
```
### metamacro_foreach

```
/**
 * Identical to #metamacro_foreach_cxt, except that no CONTEXT argument is
 * given. Only the index and current argument will thus be passed to MACRO.
 */
#define metamacro_foreach(MACRO, SEP, ...) \
        metamacro_foreach_cxt(metamacro_foreach_iter, SEP, MACRO, __VA_ARGS__)
```
以`@strongif(self)`为例，`metamacro_foreach(rac_strongify_,, self)`转换成

```
metamacro_foreach_cxt(metamacro_foreach_iter,, rac_strongify_, self)
>>
metamacro_concat(metamacro_foreach_cxt, metamacro_argcount(__VA_ARGS__))(MACRO, SEP, CONTEXT, __VA_ARGS__)
>>
metamacro_foreach_cxt1(metamacro_foreach_iter,, rac_strongify_, self)
其中`metamacro_foreach_cxt1`定义为`#define metamacro_foreach_cxt1(MACRO, SEP, CONTEXT, _0) MACRO(0, CONTEXT, _0)`
>>
metamacro_foreach_iter(0, rac_strongify_, self)
其中`metamacro_foreach_iter`的定义为`#define metamacro_foreach_iter(INDEX, MACRO, ARG) MACRO(INDEX, ARG)`
>>
rac_strongify_(0, self)
```
`rac_strongify_`宏定为如下

```
#define rac_strongify_(INDEX, VAR) \
    __strong __typeof__(VAR) VAR = metamacro_concat(VAR, _weak_);
因此`@strongify(self)`得到
__strong __typeof__(self) self = self_weak_;
```

## 总结
`@weakify(self)`和`@strongify(self)`等效于
__weak __typeof__ (self) self_weak_ = self;
__strong __typeof__(self) self = self_weak_;



## 参考文档

[剖析@weakify 和 @strongify](http://ios.jobbole.com/85019/)
[iOS中一个可以获得参数个数的宏](http://blog.campusapp.cn/2016/03/03/iOS%E4%B8%AD%E4%B8%80%E4%B8%AA%E5%8F%AF%E4%BB%A5%E8%8E%B7%E5%BE%97%E5%8F%82%E6%95%B0%E4%B8%AA%E6%95%B0%E7%9A%84%E5%AE%8F/)

