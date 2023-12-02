---
title: iOS入火坑之:一文入门oc
date: 2023-4-27 20:25:00
theme: cyanosis
highlight: atom-one-dark-reasonable
tags:
 - iOS
 - objective-c
categories:
 - iOS
cover: https://img.fansqz.com/img/013ceb57b41c9b0000018c1b591e01.jpg
---

#  oc初识

## 介绍

Objective-Objective-C是C语言的严格超集－－任何C语言程序不经修改就可以直接通过Objective-C编译器，在Objective-C中使用C语言代码也是完全合法的。Objective-C被描述为盖在C语言上的薄薄一层，因为Objective-C的原意就是在C语言主体上加入面向对象的特性。

oc文件类型如下：

| 扩展名 | 内容类型                                                     |
| ------ | ------------------------------------------------------------ |
| .h     | 头文件。头文件包含类，类型，函数和常数的声明。               |
| .m     | 源代码文件。这是典型的源代码文件扩展名，可以包含 Objective-C 和 C 代码。 |
| .mm    | 源代码文件。带有这种扩展名的源代码文件，除了可以包含Objective-C和C代码以外还可以包含C++代码。仅在你的Objective-C代码中确实需要使用C++类或者特性的时候才用这种扩展名。 |

## 类的实现

1. 类的声明，还没有实现

~~~objective-c
// Fanser.h文件
// 继承NSObject类
@interface  Fanser : NSObject {
  //成员变量
  int age;
}
// 属性，方法声明
@property (nonatomic, copy) NSString* name;
- (void) sayHello;
@end
~~~

2. 类的具体实现

~~~objective-c
// Fanser.m文件
#import "FansBoy.h"
@implementation Fanser
- (void) sayHello {
    NSLog(@"My name is %@", self.name);
}
- (void) private_method {
    // 只有当前区快能使用的本方法
}
@end

~~~

一个类就是由一个interface和一个implement组成的。

- 对于interface中声明的方法，可以理解为public方法

- 对于在implement中实现了而interface中没有声明的可以理解为private方法。只有implement的区块内可以调用到这个方法。

interface声明可以写在.m文件，也可以写在.h文件。implement文件只能写在.m文件中。

- interface写在.h文件中，其他.m文件可以通过#import的方式引入这个.h文件来使用这个类。
- interface写在.m文件中，则其他类无法导入该类。

## 类的创建

~~~objective-c
// main.m
#import "Fanser.h"
int main(int argc, const char * argv[]) {
    // 可以使用[Fanser new]，等价于[[Fanser alloc] init]
    Fanser *fanser = [[Fanser alloc] init];
    [fanser sayHello];
}
~~~

- 通过alloc来给对象分配空间，通过init方法来初始化对象。

对于如果需要实现自定义的构造方法，只需要在implement中重写父类的构造方法即可。

~~~objective-c
@implementation Fanser
- (void) sayHello {
    NSLog(@"My name is %@", self.name);
}
- (void) private_method {
    // 只有当前区快能使用的本方法
}
- (instancetype) init {
    self = [super init];
    if (self) {
        self.name = @"fzw";
    }
    NSLog(@"重写构造方法");
    return self;
}
@end
~~~



## 函数方法

1. 方法定义

~~~objective-c
// + 代表类方法 - 代表实例方法
// 这里的methodNameWith, paraLable为方法名标签，oc可以给每一个参数进行解释
+(int)methodNameWith:(int)para1 paraLabel:(int)paral
~~~

2. 方法的调用

~~~objective-c
Fanser fanser = [[Fanser new]];
// 类方法调用
int answer = [Fanser methodNameWith:1 paramLable:2];
// 实例方法调用
int answer = [fanser mehtodNameWith:1 paramLable:2];
// 对于没有任何参数的方法可以直接使用.进行调用
int answer = fanser.answer; //相当于[fanswer answer]
~~~

oc的一大特色就是其消息模型。Objective-C里，与其说对象互相调用方法，不如说对象之间互相传递消息更为精确。此二种风格的主要差异在于调用方法/消息传递这个动作。C++里类别与方法的关系严格清楚，一个方法必定属于一个类别，而且在编译时（compile time）就已经紧密绑定，不可能调用一个不存在类别里的方法。但在Objective-C，类别与消息的关系比较松散，调用方法视为对对象发送消息，所有方法都被视为对消息的回应。所有消息处理直到运行时（runtime）才会动态决定，并交由类别自行决定如何处理收到的消息。也就是说，一个类别不保证一定会回应收到的消息，如果类别收到了一个无法处理的消息，程序只会抛出异常，不会出错或崩溃。

## 成员变量

oc不像其他语言一样，oc如果将成员变量在interface中进行声明，则可以被外部访问。如果不在interface中声明，而写在implement中，则为私有，不能被外部访问。

~~~objective-c
// Fanser.h
@interface  Fanser : NSObject {
  	// 公开变量
    @public int public_int_variable;
    @protected double protected_double_variable;
}
@end
  
// Fanser.m
@implementation Fanser {
  	// 私有成员变量
    int _private_int_variable;
    double _private_double_variable;
}
@end
~~~

~~~objective-c
//公开变量可以在外部进行访问
// main.m
#import "Fanser.h"
int main(int argc, const char * argv[]) {
    
    Fanser *fanser = [[Fanser alloc] init];
    fanser->public_int_variable = 111;
    NSLog(@"%d",fanser->public_int_variable);
}
~~~

事实上，对于面向对象语言来说，我们并不喜欢使用公开的变量。而是将成员变量私有化，然后通过get/set方法访问变量。下面是访问成员变量正确的打开方式

~~~objective-c
// Fanser.h
@interface  Fanser : NSObject
// get/set方法的声明
- (void)setName:(NSString *)name;
- (NSString *)name;
@end
  
  
// Fanser.m
@implementation Fanser {
    //私有变量
    NSString* _name;
}
// get/set方法的实现
- (void)setName:(NSString *)name {
    self->_name = name;
}
- (void)name {
    return _name;
}
~~~

## 属性

对于每个成员变量我们都需要去声明和实现它的get/set方法无疑是会增加程序员的开发成本，所以oc为我们引入的属性。

~~~objective-c
// fanser.h
@interface  Fanser : NSObject
// 属性的声明
@property (nonatomic, copy) NSString* name;
@end
  
// fanser.m
@implementation Fanser
@end
~~~

通过@property可以声明属性。以上代码oc会自动生成`_name`的私有变量，以及`name`,`setName`方法。

当然，属性不止是有这些用处，属性还可以设置很多特性。

- 访问原子性，默认：atomic
- 存取特性，默认：readwrite
- 内存管理，默认：strong
- 重写get/set方法：getter=getName，setter=assignName
- 是否可为null，默认：null_unspecified

## 协议

oc中的协议类似于其他语言等接口，可以声明一些协议，然后一些类可以去遵循这些协议。oc是显示声明。

1. 协议的声明

~~~objective-c
// FanserProtocol.h
// 协议的定义，<NSObject>意味着继承制自NSObject协议
@protocol  Fanser <NSObject>
// @required 要求要实现
@required
- (void) doCodeReview;
// optional 不一定要实现
@optional
- (void)writeDailyReportAt:(NSDate *)date;
@end
~~~

2. 协议的遵守

~~~objective-c
//声明遵守了Fanser，和SeniorLevel协议
// 声明遵守了某些协议以后，就会声明这些协议所含有的方法
@interface  Fanser : NSObject <Fanser,SeniorLevel>
@end
~~~

3. 协议类型

~~~objective-c
@interface System : NSObject
// id<FanserProtocol> 的类型是遵循FanserProtocol协议的对象。
//一个对象如果遵循了FanserProtocol协议，就可以给fanser赋值。
@property (weak, nonatomic) id<FanserProtocol> fanser;
@end
~~~

有的时候，一个类声明要遵守某个协议，但是却不提供其实现。在编译期间会警告，但是是可以编译成功的。如果在程序的运行过程中，调用了这个没有实现的方法，则可能会导致程序出错。

我们可以使用respondsToSelector来判断某个对象是够实现了某个方法。

~~~objective-c
Myclass *myObject = [[MyClass alloc] init];
if ([myObject respondsToSelector:@selector(numberOfDatasFrom:)]) {
  int dataCount = [myObject numberOfDatasFrom:@"main"];
  NSLog(@"实现了该方法");
}
~~~

# Frameworks

## 框架的分层

![image-20230427195145838](https://img.fansqz.com/img/image-20230427195145838.png)

- Cocoa Touch（接触层）：提供了很多ui相关的框架，与界面相关
- Media（媒体层）：提供了很多媒体相关技术支持的框架
- Core Services（服务层）：提供了很多基础能力，核心服务
- Core OS（操作系统层）：苹果提供的最底层的框架

处于上层的框架会依赖与下层框架，下层框架不会依赖于上层

# Foundation

通过`#import <Foundation/Foundation.h>`即可引入Foundation框架

## NSObject

所有类的根类（root Class），含有大量oc语言方法。

1. 分配内存空间&初始化方法

~~~objective-c
@interface NSObject <NSObject>
// 为新对象分配内存空间
+(instancetype)alloc;
// 初始化对象
-(instancetype)init;
// 相当于[[NSObject alloc] init]
+(instancetype)new;
@end
~~~

2. 发送消息

~~~objective-c
@protocol NSObject
// 发送消息给对象，返回消息执行结果
- (id)performSelector:(SEL)aSelector;
// 发送一个带参数消息给对象，返回消息执行结果
- (id)performSelector:(SEL)aSelector withObject:(id)object;
// 判断对象是否可以调用指定方法
- (BOOL)respondsToSelector:(SEL)aSelector;
@end
~~~

3. 类关系判断

~~~objective-c
@interface NSObject <NSObject>
// 获取当前对象的类
- (Class)class;
// 获取当前对象的父类
- (Class)superclass;
// 判断对象是否是给定类或给定类的子类的实例
- (BOOL)isKindOfClass:(Class)aClass;
// 判断对象是否是给定类的实例
- (BOOL)isMemberOfClass:(Class)aClass;
// 判断对象是否遵循给定的协议
- (BOOL)conformsToProtocol:(Protocol *)aProtocol;
@end
~~~

## NSString

NSString是一个专门用于处理字符串的类。提供了很多实用的方法

1. NSString的创建

~~~objective-c
// 通过@""创建，常用
NSString *string = @"fzw";
// 通过c字符串创建NSString
NSSTring *string2 = [NSString stringWithUTF8Sring:"fzw"];

// 创建空字符串
NSString *string = [[NSString alloc] init];
// 通过一个NSString创建另一个NSString
NSString *string2 = [NSString stringWithString:@"hello"];
// 拼接字符串
NSString *string3 = [string2 stringByAppendingString:@"world"];

// 格式化创建
NSString *string1 = [[NSString alloc]initWithFormat:@"%d, %@", 123, @"456"];
~~~

2. NSString转为其他类型

~~~objective-c
NSString *numberStr = @"456";
BOOL boolValue = [numberStr boolValue];
int intValue = [numberStr intValue];
float floatValue = [numberStr floatValue];
double doubleValue = [numberStr doubleValue];
~~~

## NSArray

一个数组类，数组操作都习惯使用NSArray。

1. NSArray的创建

~~~objective-c
// 创建空数组
NSArray *arr1 = [NSArray array];
NSArray *arr2 = [[NSArray alloc] init];

// 创建数组
NSArray *arr3 = @[@"a", @"b", @"c"];
NSArray *arr4 = [NSArray arrayWithObjects:@"a", @"b", @"c", nil];

// Array无法存放基本数据类型，如果需要存放基本数据类型，可以先转化为NSNumber
NSNumber *number = [[NSNumber alloc] initWithInt:666];
NSArray *arr5 = [NSArray arrayWithObjects:@(123), number, nil];

// 限制存放的对象的类型
NSArray<NSNumber *> *numberArr = @[@(1), @(2), @(0.3)];
~~~

2. NSArray元素的访问

可以通过`arr[index]`的方式来访问元素，也可以通过`[arr objectAtIndex:index]`来访问元素，这两种写法效果一样。

~~~objective-c
// 在NSArray访问元素的时候，由于无法确定原始类型，所以可以加一些判断。
// 1. 错误写法
NSArray *arr = @[@"abc", @(1)];
for (int i = 0; i < arr.count; i++) {
  NSString *str = arr[i];
  NSLog(@"%@", str);
}

// 2. 正确写法
NSArray *arr = @[@"abc", @(1)];
for (int i = 0; i < arr.count; i++) {
  if ([arr[i] isKindOfClass:[NSString class]]) {
    NSString *str = arr[i];
    NSLog(@"%@", str);
  } else if ([arr[i] isKindOfClass:[NSNumber class]]) {
    NSNumber *num = arr[i];
    NSLog(@"%@", num);
  }
}
~~~

3. 快速枚举(for-in)

oc提供了快速枚举的方式来简化数组的枚举，

~~~objective-c
NSArray<NSString*> *arr = @[@"a", @"b", @"c"];
for (NSString *str in arr) {
  NSLog(@"%@", str);
}
~~~



***注：对于NSArray，是无法改变数组的元素的。如果需要改变数组的元素，则需要使用NSMutableArray。***

## NSMutableArray

从名字上看，是一个可变数组。可以添加删除元素。

常用方法

~~~objective-c
//创建一个数组，指定容量为size
+(id)arrayWithCapacity:size
// 初始化一个新分配的数组，指定容量为size
-(id)initWithCapacity:size
//将对象obj添加到数组末尾
-(void)addObject:obj
//将对象 obj 插入到索引为 i 的位置
-(void)insertObject:obj atIndex:i	
//将数组中索引为 i 处的元素用obj 置换
-(void)replaceObject:obj atIndex:i
//从数组中删除所有是 obj 的对象
-(void)removeObject:obj	
//从数组中删除索引为 i 的对像
-(void)removeObjectAtIndex:i
//用 selector 只是的比较方法将数组排序
-(void)sortUsingSelector:(SEL)selector	
~~~

## NSDictionary

数据字典，主要用于存放key-value键值对。

1. map的创建

~~~objective-c
// 创建
NSDictionary *map = @{
  @"key1":@"value1",
  @"key2":@"value2"
};
NSDictionary *map2 = [[NSDictionary alloc] initWithObjects:@[@"value1", @"value2"] forKeys:@[@"key1",@"key2"]];
// 限制存放类型
NSDictionary<NSString*, NSNumber*> *map3 = @{
  @"key1":@(0),
  @"key2":@(1)
};
~~~

2. map的遍历

~~~objective-c
NSDictionary<NSString*, NSString*> *map = @{
  @"key1":@"value1",
  @"key2":@"value2"
};
NSArray<NSString*> *keys = [map allKeys];
for (int i = 0; i < keys.count; i++) {
  NSString *key = keys[i];
  NSString *value = map[key];
  NSLog(@"%@", value);
}
~~~

3. 快速枚举

~~~objective-c
NSDictionary<NSString*, NSString*> *map = @{
  @"key1":@"value1",
  @"key2":@"value2"
};
for (NSString *key in map) {
  NSString *value = map[key];
  NSLog(@"%@", value);
}
~~~

***注：对于NSDictionary，是不可变的。如果需要改变NSDictionary的元素，则需要使用NSMutableArray。***



## NSMutableDictionary

从名字上看，是一个可变字典。可以添加删除键值对。

常用方法：

~~~objective-c
// 根据初始容量创建字典
- (instancetype)initWithCapacity:(NSUInteger)numItems;
// 设置一个key-value
- (void)setObject:(ObjectType)anObject forKey:(id<NSCopying>)aKey;
// 根据key删除一个key-value
- (void)removeObjectForKey:(KeyType)aKey;
~~~



