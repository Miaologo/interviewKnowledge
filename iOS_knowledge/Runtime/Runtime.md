# Runtime

头文件 \<obj/runtime.h>

## 基本概念

1. RunTime简称运行时,就是系统在运行的时候的一些机制，其中最主要的是消息机制。
2. 对于C语言，函数的调用在编译的时候会决定调用哪个函数，编译完成之后直接顺序执行，无任何二义性。
3. OC的函数调用成为消息发送。属于动态调用过程。在编译的时候并不能决定真正调用哪个函数（事实证明，在编 译阶段，OC可以调用任何函数，即使这个函数并未实现，只要申明过就不会报错。而C语言在编译阶段就会报错）。
4. 只有在真正运行的时候才会根据函数的名称找 到对应的函数来调用。

关于传统Runtime和现代版Runtime的区别

> In the legacy runtime, if you change the layout of instance variables in a class, you must recompile classes that inherit from it.
> In the modern runtime, if you change the layout of instance variables in a class, you do not have to recompile classes that inherit from it.
In addition, the modern runtime supports instance variable synthesis for declared properties (see Declared Properties in The Objective-C Programming Language).

## 基本方法

* 获取某个类的类方法
`Method class_getClassMethod(Class cls , SEL name)`

* 获取某个类的实例对象方法
`Method class_getInstanceMethod(Class cls , SEL name)`

* 交换两个方法的实现
`void method_exchangeImplementations(Method m1, Method m2)`

* 获取类cls的实例方法`@selector(name)`的实现`IMP`
`IMP class_getMethodImplementation(Class cls, SEL name)`

* 把类cls中的`@selector(name)`方法的实现替换为`imp`，`types`为表述此方法的参数类型的字符串
`IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types)`

* 获取一个方法的参数描述字符串
`const char *method_getTypeEncoding(Method m)`

* 向一个类中添加一个新方法，添加成功，返回`YES`，否则`NO`

```
//贴此class_addMethod的一个官方注释，不翻译了...
//@return YES if the method was added successfully, otherwise NO 
//(for example, the class already contains a method implementation with that name).
//@note class_addMethod will add an override of a superclass's implementation, 
//but will not replace an existing implementation in this class. 
//To change an existing implementation, use method_setImplementation.
```
`BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *type)`

## 常见作用

* 动态的添加对象的成员变量和方法
* 动态交换两个方法的实现
* 实现分类添加属性
* 实现NSCoding的自动归档和解档
* 实现字典模型的自动转换

### 动态变量控制

example：动态修改变量值

```
    unsigned int count = 0;
    Ivar *ivar = class_copyIvarList([self.xiaoMing class], &count);
    for (int i = 0; i<count; i++) {
        Ivar var = ivar[i];
        const char *varName = ivar_getName(var);
        NSString *name = [NSString stringWithUTF8String:varName];

        if ([name isEqualToString:@"_englishName"]) {
            object_setIvar(self.xiaoMing, var, @"Minggo");
            break;
        }
    }
    NSLog(@"XiaoMing first answer is %@",self.xiaoMing.englishName);
    self.nameTf.text = self.xiaoMing.englishName;
```

### 动态方法交换

```
    Method m1 = class_getInstanceMethod([self.xiaoMing class], @selector(firstSay));
    Method m2 = class_getInstanceMethod([self.xiaoMing class], @selector(secondSay));

    method_exchangeImplementations(m1, m2);
    NSString *secondName = [self.xiaoMing firstSay];

    self.nameTf.text = secondName;
    NSLog(@"XiaoMing:My name is %@",secondName);
```

### 动态添加方法

1、动态的给类中添加方法
`class_addMethod([self.exampleClass class], @selector(guess), (IMP)guessAnswer, "v@:");
`
> 这里参数地方说明一下：
(IMP)guessAnswer 意思是guessAnswer的地址指针;
> "v@:" 意思是，v代表无返回值void，如果是i则代表int；@代表 id sel; : 代表 SEL _cmd;
> “v@:@@” 意思是，两个参数的没有返回值。

2、 调用guess方法相应事件
`[self.exampleClass performSelector:@selector(guess)];`

3、 编写guessAnswer实现

```
void guessAnswer(id self,SEL _cmd){
    NSLog(@"He is from GuangTong");   
}
//这个有两个地方留意一下：
//1.void的前面没有+、-号，因为只是C的代码。
//2.必须有两个指定参数(id self,SEL _cmd)
```

```
-(void)answer{
    class_addMethod([self.xiaoMing class], @selector(guess), (IMP)guessAnswer, "v@:");
    if ([self.xiaoMing respondsToSelector:@selector(guess)]) {

        [self.xiaoMing performSelector:@selector(guess)];

    } else{
        NSLog(@"Sorry,I don't know");
    }
    self.cityTf.text = @"GuangTong";
}

void guessAnswer(id self,SEL _cmd){

    NSLog(@"He is from GuangTong");

}
```

## category设置属性

众所周知，分类中是无法设置属性的，如果在分类的声明中写@property 只能为其生成get和set方法的声明。无法生成成员变量，即使使用点语法可以调用，程序执行也会crash

* set方法： 将值value跟对象object关联起来（将值value存储在对象的 object中）
	参数object：给对象object设置属性
	参数key：一个属性对应一个key，也可以通过key获取这个存储的值，key可以是任何类型，double、int等，建议使用char，可以节省字节
	参数value：给属性设置的值
	参数policy：存储策略（assign、copy、retain（strong））
	
	`void obj_setAssociatedObject(id object, const void *key, id value, obj_AssociationPolicy policy)`
	
```
#import "MyClass.h"

@interface MyClass (Category1)

@property(nonatomic,copy) NSString *name;

@end
```
```
#import "MyClass+Category1.h"
#import <objc/runtime.h>

@implementation MyClass (Category1)

+ (void)load
{
    NSLog(@"%@",@"load in Category1");
}

- (void)setName:(NSString *)name
{
    objc_setAssociatedObject(self,
                             "name",
                             name,
                             OBJC_ASSOCIATION_COPY);
}

- (NSString*)name
{
    NSString *nameObject = objc_getAssociatedObject(self, "name");
    return nameObject;
}

@end
```

## 获取类的成员变量

* 获取类的所有成员变量
参数 cls：对应的类
参数 outCount： 存放属性个数的地址
返回值： 存放所有获取到的属性的一个列表
`Ivar *class_copyIvarList(Class cls, unsigned int *outCount)`

* 获取成员变量的名字
`const char *ivar_getName(Ivar v)`

* 获取成员变量的类型
`const char *ivar_getTypeEndcoding(Ivar v)`

__实例：获取某个类中的所有成员变量的名字和类型__

```
unsigned int outCount = 0;
Ivar *ivars = class_copyIvarList([Person class], &outCount);

// 遍历所有成员变量
for (int i = 0; i < outCount; i++) {
    // 取出i位置对应的成员变量
    Ivar ivar = ivars[i];
    const char *name = ivar_getName(ivar);
    const char *type = ivar_getTypeEncoding(ivar);
    NSLog(@"成员变量名：%s 成员变量类型：%s",name,type);
}
// 注意释放内存！
free(ivars);
```
__实例：利用runtime获取所有属性来重写归档解档方法__

```
// 设置不需要归解档的属性
- (NSArray *)ignoredNames {
    return @[@"_aaa",@"_bbb",@"_ccc"];
}

// 解档方法
- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super initWithCoder:aDecoder]) {
        // 获取所有成员变量
        unsigned int outCount = 0;
        Ivar *ivars = class_copyIvarList([self class], &outCount);

        for (int i = 0; i < outCount; i++) {
            Ivar ivar = ivars[i];
            // 将每个成员变量名转换为NSString对象类型
            NSString *key = [NSString stringWithUTF8String:ivar_getName(ivar)];

            // 忽略不需要解档的属性
            if ([[self ignoredNames] containsObject:key]) {
                continue;
            }

            // 根据变量名解档取值，无论是什么类型
            id value = [aDecoder decodeObjectForKey:key];
            // 取出的值再设置给属性
            [self setValue:value forKey:key];
            // 这两步就相当于以前的 self.age = [aDecoder decodeObjectForKey:@"_age"];
        }
        free(ivars);
    }
    return self;
}

// 归档调用方法
- (void)encodeWithCoder:(NSCoder *)aCoder {
     // 获取所有成员变量
    unsigned int outCount = 0;
    Ivar *ivars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i++) {
        Ivar ivar = ivars[i];
        // 将每个成员变量名转换为NSString对象类型
        NSString *key = [NSString stringWithUTF8String:ivar_getName(ivar)];

        // 忽略不需要归档的属性
        if ([[self ignoredNames] containsObject:key]) {
            continue;
        }

        // 通过成员变量名，取出成员变量的值
        id value = [self valueForKeyPath:key];
        // 再将值归档
        [aCoder encodeObject:value forKey:key];
        // 这两步就相当于 [aCoder encodeObject:@(self.age) forKey:@"_age"];
    }
    free(ivars);
}
```

封装到NSObject类中

```
#import <Foundation/Foundation.h>

@interface NSObject (Extension)
- (NSArray *)ignoredNames;
- (void)encode:(NSCoder *)aCoder;
- (void)decode:(NSCoder *)aDecoder;
@end

#import "NSObject+Extension.h"
#import <objc/runtime.h>

@implementation NSObject (Extension)

- (void)decode:(NSCoder *)aDecoder {
    // 一层层父类往上查找，对父类的属性执行归解档方法
    Class c = self.class;
    while (c &&c != [NSObject class]) {

        unsigned int outCount = 0;
        Ivar *ivars = class_copyIvarList(c, &outCount);
        for (int i = 0; i < outCount; i++) {
            Ivar ivar = ivars[i];
            NSString *key = [NSString stringWithUTF8String:ivar_getName(ivar)];

            // 如果有实现该方法再去调用
            if ([self respondsToSelector:@selector(ignoredNames)]) {
                if ([[self ignoredNames] containsObject:key]) continue;
            }

            id value = [aDecoder decodeObjectForKey:key];
            [self setValue:value forKey:key];
        }
        free(ivars);
        c = [c superclass];
    }

}

- (void)encode:(NSCoder *)aCoder {
    // 一层层父类往上查找，对父类的属性执行归解档方法
    Class c = self.class;
    while (c &&c != [NSObject class]) {

        unsigned int outCount = 0;
        Ivar *ivars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i++) {
            Ivar ivar = ivars[i];
            NSString *key = [NSString stringWithUTF8String:ivar_getName(ivar)];

            // 如果有实现该方法再去调用
            if ([self respondsToSelector:@selector(ignoredNames)]) {
                if ([[self ignoredNames] containsObject:key]) continue;
            }

            id value = [self valueForKeyPath:key];
            [aCoder encodeObject:value forKey:key];
        }
        free(ivars);
        c = [c superclass];
    }
}
@end
```

__实例：利用runtime获取所有属性进行字典转模型__

```
NSObject+JSONExtension.h 文件

// 返回数组中都是什么类型的模型对象
- (NSString *)arrayObjectClass ;

NSObject+JSONExtension.m 文件

#import "NSObject+JSONExtension.h"
#import <objc/runtime.h>

@implementation NSObject (JSONExtension)

- (void)setDict:(NSDictionary *)dict {

    Class c = self.class;
    while (c &&c != [NSObject class]) {

        unsigned int outCount = 0;
        Ivar *ivars = class_copyIvarList(c, &outCount);
        for (int i = 0; i < outCount; i++) {
            Ivar ivar = ivars[i];
            NSString *key = [NSString stringWithUTF8String:ivar_getName(ivar)];

            // 成员变量名转为属性名（去掉下划线 _ ）
            key = [key substringFromIndex:1];
            // 取出字典的值
            id value = dict[key];

            // 如果模型属性数量大于字典键值对数理，模型属性会被赋值为nil而报错
            if (value == nil) continue;

            // 获得成员变量的类型
            NSString *type = [NSString stringWithUTF8String:ivar_getTypeEncoding(ivar)];

            // 如果属性是对象类型
            NSRange range = [type rangeOfString:@"@"];
            if (range.location != NSNotFound) {
                // 那么截取对象的名字（比如@"Dog"，截取为Dog）
                type = [type substringWithRange:NSMakeRange(2, type.length - 3)];
                // 排除系统的对象类型
                if (![type hasPrefix:@"NS"]) {
                    // 将对象名转换为对象的类型，将新的对象字典转模型（递归）
                    Class class = NSClassFromString(type);
                    value = [class objectWithDict:value];

                }else if ([type isEqualToString:@"NSArray"]) {

                    // 如果是数组类型，将数组中的每个模型进行字典转模型，先创建一个临时数组存放模型
                    NSArray *array = (NSArray *)value;
                    NSMutableArray *mArray = [NSMutableArray array];

                    // 获取到每个模型的类型
                    id class ;
                    if ([self respondsToSelector:@selector(arrayObjectClass)]) {

                        NSString *classStr = [self arrayObjectClass];
                        class = NSClassFromString(classStr);
                    }
                    // 将数组中的所有模型进行字典转模型
                    for (int i = 0; i < array.count; i++) {
                        [mArray addObject:[class objectWithDict:value[i]]];
                    }

                    value = mArray;
                }
            }

            // 将字典中的值设置到模型上
            [self setValue:value forKeyPath:key];
        }
        free(ivars);
        c = [c superclass];
    }
}

+ (instancetype )objectWithDict:(NSDictionary *)dict {
    NSObject *obj = [[self alloc]init];
    [obj setDict:dict];
    return obj;
}

@end
```

## 参考文档
[OC最实用的runtime总结，面试、工作你看我就足够了！](http://www.jianshu.com/p/ab966e8a82e2)
[runtime官方文档翻译](http://www.jianshu.com/p/158c5d118937)
[谈Runtime机制和使用的整体化梳理](http://www.jianshu.com/p/8916ad5662a2)


