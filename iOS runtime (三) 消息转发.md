讲述消息转发的步骤简述、Dynamic Method Resolution 、Replacement Receiver 、Message Forwarding、相关函数总结。每个步骤都有例子，包括 NSInvocation 的使用。


如果给对象发送无法处理的消息，在抛出错误之前，运行时系统会给对象一个动态方法解析和转发消息的机会。

<br>

# 步骤简述
---

消息转发的步骤：添加方法、转发对象、转发消息。

第一步，运行时系统会询问该类是否添加方法，即调用下面的函数：

```
// 动态解析实例方法
+(BOOL)resolveInstanceMethod:(SEL)sel ;

// 动态解析类方法
+(BOOL)resolveClassMethod:(SEL)sel ;
```

如果重写了上面的函数，在函数里面成功添加方法并且返回 YES，运行时系统会重新给该类发送之前无法处理的消息。

第二步，如果返回 NO，系统会询问该类是否有其他对象可以处理消息，即调用下面的方法：
```
- (id)forwardingTargetForSelector:(SEL)aSelector
```

如果不返回 self 或 nil，系统会把消息转发给返回的对象。
比如对象 A 持有对象 B、C、D，通过 ``` respondsToSelector ``` 函数判断谁可以处理该消息，就返回该对象。这样外界看起来就像是对象 A亲自处理了该消息。

第三步，如果返回 self 或者 nil，系统会进行真正的消息转发：
```
- (void)forwardInvocation:(NSInvocation *)invocation;
```

系统把选择子 SEL、目标 target、参数封装到 NSInvocation 里面，然后调用该类上面的函数。可以重写上面的函数来修改 NSInvocation 的属性，比如 SEL、target、参数等。封装 NSInvocation 对象，系统会调用该类的 ```methodSignatureForSelector``` 函数获取 NSMethodSignature 对象。

如果子类不处理，要调用父类的同名方法，而 NSObject 的方法默认是调用 ``` doesNotRecognizeSelector ``` 方法抛出异常。

越到后面，转发消息的代价就越大，所以最好在前面的步骤就处理掉消息。如果在第一步处理掉，系统还会缓存添加的方法，再次收到相同的消息就不走消息转发的流程了。如果第三步只是修改 NSInvocation 的 target，还不如在第二步就处理掉，否则还得创建 NSInvocation 对象。

下面是消息转发的流程图和各个步骤的详解。

![《Effective Objective-C 2.0  编写高质量iOS与OS X代码的52个有效方法》](https://upload-images.jianshu.io/upload_images/1235154-d257874b7bc75b87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



<br>





# 一、Dynamic Method Resolution
---

动态添加方法，需要 ```#import <objc/runtime.h>```，
关键是调用 ```class_addMethod``` 函数：

```
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);
```

添加成功会返回 YES。如果类已经有同名的函数会添加失败。
YES if the method was added successfully, otherwise NO (for example, the class already contains a method implementation with that name).

cls 是被添加方法的类或元类。实例方法添加到类，类方法添加到元类。如果 self 是类，使用 self 或 [self class] 都能获取类，调用 ```object_getClass(self)``` 可以获取元类。如果 self 是对象不是类，```object_getClass(self)``` 获取的是类。

IMP 是函数实现地址，可以是 C语言的函数名。如果是 OC 定义的方法，可以通过方法 ```class_getMethodImplementation(Class cls, SEL sel)``` 获取。

const char *types 是函数类型，比如 "v@:"，v 是返回类型 void，@ 表示对象，：表示选择子 SEL。具体看我的文章 [《iOS runtime (二) 编程指导》](https://www.jianshu.com/writer#/notebooks/5011690/notes/10405022) 的 Type Encodings 章节。

需要注意的是，OC 的方法最少有两个参数，接收消息的对象 self 和选择子 _cmd。运行时系统调用函数时，会把这两个参数传给函数实现。也就是 OC 定义的无参数的函数，转成 C语言函数后，会有两个形参：

```
// OC 定义的函数
- (void)myMethodIMP {

}

// 转换后的 C语言函数
void myMethodIMP(id self, SEL _cmd)
{
// implementation ....
}
```

举个动态添加方法的例子，外部给 ClassA 的对象发送消息 
```[obj performSelector:@selector(funA)]```，
然后在 ClassA 添加对应的方法 funOfClassA：

```
// 要 #import <objc/runtime.h>

@implementation ClassA

// 要添加的方法
- (void)funOfClassA {
// 注意，这里输出的是 "ClassA funA"，而不是 "ClassA funOfClassA"。
// 因为外面是通过 [obj performSelector:@selector(funA)] 来调用的
// 如果是直接调用 funOfClassA 才会输出 "ClassA funOfClassA"。
// self 是消息的 target，_cmd 是消息的 SEL。
NSLog(@"%@ %@", self.class, NSStringFromSelector(_cmd));
}

// 动态添加方法
+ (BOOL)resolveInstanceMethod:(SEL)sel {
// 如果外面调用了 funA 才动态添加方法 funOfClassA。
// 添加是指把 SEL funA 和 funOfClassA 的方法实现地址放到
// 一个 Method 里面，然后添加到类的方法列表。
if ([NSStringFromSelector(sel) isEqualToString:@"funA"]) {
// 获取 SEL 和 IMP
SEL selToAdd = @selector(funOfClassA); 
IMP imp = class_getMethodImplementation(self, selToAdd);

// 创建 Method 并添加到方法列表
BOOL success = class_addMethod(self, sel, imp, "v@:"); 
NSLog(@"%@ 添加方法%@", self, success?@"成功":@"失败");
return success;
} else {
return [super resolveInstanceMethod:sel];
}
}

@end
```

测试代码：
```
- (void)viewDidLoad {
[super viewDidLoad];

ClassA *obj = [ClassA new];
[obj performSelector:@selector(funA)];
}
```

结果输出：
```
ClassA 添加方法成功
ClassA funA
```

测试的流程和结果，代码的注释已经写清楚了。

上面被添加的函数是用 OC 的方式写的，也可以用 C语言的方式写：
```
// 要添加的方法
void myMethodIMP(id self, SEL _cmd)
{
NSLog(@"%@ %@", [self class], NSStringFromSelector(_cmd));
}
```

添加方法的代码改成这样：
```
// C语言函数名就是 IMP，加 (void *) 是去掉警告
BOOL success = class_addMethod(self, sel, (void *)myMethodIMP, "v@:"); 
```



<br>

# 二、Replacement Receiver
---

备用接收者，运行时系统会调用下面的方法获取要转发消息的对象：
```
- (id)forwardingTargetForSelector:(SEL)aSelector
```

如果返回 self 会死循环。
如果返回 non-nil 对象，运行时系统会转发消息给它。
适用于只是简单的转发消息给另一个对象，花费的代价比真正的消息转发过程小很多很多。
不适用于捕获 NSInvocation、修改参数、修改返回值的情景。

This method gives an object a chance to redirect an unknown message sent to it before the much more expensive ```forwardInvocation``` machinery takes over.
This is useful when you simply want to redirect messages to another object and can be an order of magnitude faster than regular forwarding. 
It is not useful where the goal of the forwarding is to capture the NSInvocation, or manipulate the arguments or return value during the forwarding.


举个栗子，外部给 ClassA 对象发送 
```[obj performSelector:@selector(funB)]``` 
消息，然后转发给 ClassB 的对象处理。

ClassA 的 forwardingTargetForSelector 方法：

```
- (id)forwardingTargetForSelector:(SEL)aSelector {
id target = nil;

if ([self.objb respondsToSelector:aSelector]) {
target = self.objb;
} else if ([self.objc respondsToSelector:aSelector]) {
target = self.objc;
}

if (target) {
NSString *cmd = NSStringFromSelector(_cmd);
NSString *sel = NSStringFromSelector(aSelector);
NSLog(@"[%@  %@], 消息的SEL: %@, 转发的target: %@ ", self.class, cmd, sel, [target class]);
return target;
} else {
return [super forwardingTargetForSelector:aSelector];
}
}
```

ClassB 的 funB 方法：
```
- (void)funB {
NSLog(@"%@ %@", self.class, NSStringFromSelector(_cmd));
}
```

测试代码：
```
- (void)viewDidLoad {
[super viewDidLoad];

ClassA *obj = [ClassA new];
//    [obj performSelector:@selector(funA)];
[obj performSelector:@selector(funB)];
}
```

输出结果：
```
[ClassA  forwardingTargetForSelector:], 消息的SEL: funB, 转发的target: ClassB 
ClassB funB
```

在函数 forwardingTargetForSelector: 里面，通过 respondsToSelector 来判断能响应的对象。如果没有合适的对象，就调用 super 的同名方法。

外部可能通过 respondsToSelector 判断 ClassA 是否响应 funB 函数，因此要重写：
```
// ClassA.m
// 1、通过函数名判断
- (BOOL)respondsToSelector:(SEL)aSelector {
if ([NSStringFromSelector(aSelector) isEqualToString:@"funB"]) {
return YES;
} else {
return [super respondsToSelector:aSelector];
}
}

// 2、或者通过持有的对象判断
- (BOOL)respondsToSelector:(SEL)aSelector {
if ([self.objb respondsToSelector:aSelector]) {
return YES;
} else if ([self.objc respondsToSelector:aSelector]) {
return YES;
} else {
return [super respondsToSelector:aSelector];
}
}
```

通过此方案，可以用“组合”来模拟出“多重继承”的某些特性。在一个对象内部，可能还有一系列其他对象，该对象可以经由此方法将能够处理某选择子的相关内部对象返回，这样的话， 在外界看来好像是该对象亲自处理了这些消息。

<br>

# 三、Message Forwarding
---

运行时系统转发消息的步骤：
1. 系统调用 methodSignatureForSelector 获取方法签名；
2. 如果方法签名返回 nil，终止消息转发，程序崩溃；
3. 使用方法签名创建 NSInvocation 对象；
4. 调用 forwardInvocation 方法，并且把 NSInvocation 传进来；

用户的步骤：
1. 重写 methodSignatureForSelector 方法；
2. 在 forwardInvocation 方法里面修改 NSInvocation 的 target、参数等属性；
3. 通过 NSInvocation 的 invoke 或 invokeWithTarget 方法发送消息；
4. 发送消息后，可以通过 NSInvocation 获取和修改函数的返回值。

注意事项：
1. 如果 methodSignatureForSelector 返回 nil （NSObject 默认返回 nil），就不会调用 forwardInvocation 方法。可以通过
[NSMethodSignature signatureWithObjCTypes:"v@:"] 
函数创建方法签名。
2. 注意 methodSignatureForSelector 返回的方法签名，要和消息最终的 SEL 相对应。比如修改消息的 SEL 为 sel2，那么返回的方法签名要和 sel2 相对应。如果方法签名错误，可能会在其他地方崩溃。
3. 系统创建的 NSInvocation 对象，包含了消息的 target、SEL、参数、方法签名等信息，可以直接修改它的 target、SEL，可以通过它的 setArgument:atIndex: 方法修改参数（index 0 和 1 是隐藏参数 self 和 _cmd），通过 getReturnValue 或 setReturnValue 方法捕获和修改返回值，通过 invoke 或 invokeWithTarget 方法发送消息。

下面是主要用到的几个函数：
```
// 运行时系统调用，或用户主动调用，获取方法签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector

// 创建方法签名
+ (nullable NSMethodSignature *)signatureWithObjCTypes:(const char *)types;

// 子类重写，修改 NSInvocation 并且转发消息
- (void)forwardInvocation:(NSInvocation *)anInvocation
```

<br>

举两个栗子：
1. 外部给 ClassA 对象 a 发送 funC 消息，a 把消息的 SEL 修改为 funOfClassC，然后转发给 ClassC 对象处理。
2. 外部给 ClassA 对象 a 发送 printNumber: 消息，a 把消息转发给 ClassC 对象 c 处理，并且修改消息的参数和返回值、捕获函数调用的结果。

先看测试代码：
```
- (void)viewDidLoad {
[super viewDidLoad];

ClassA *obj = [ClassA new];
//    [obj performSelector:@selector(funA)];
//    [obj performSelector:@selector(funB)];

//    if ([obj respondsToSelector:@selector(funB)]) {
//        [obj performSelector:@selector(funB)];
//    }

// 测试修改消息的 SEL 和 target
if ([obj respondsToSelector:@selector(funC)]) {
[obj performSelector:@selector(funC)];
}

// 测试修改消息的参数和返回值
if ([obj respondsToSelector:@selector(printNumber:)]) {
id result = [obj performSelector:@selector(printNumber:) withObject:@(222)];
// 外部获取的返回值 resutl = 666
NSLog(@"外部获取的返回值 resutl = %@", result);
}
```

由于 ClassA 没有实现 funC 和 printNumber: 方法，因此要重写 respondsToSelector 方法：

```
// ClassA.m
- (BOOL)respondsToSelector:(SEL)aSelector {
if ([NSStringFromSelector(aSelector) isEqualToString:@"funA"]) {
return YES;
} else if ([self.objb respondsToSelector:aSelector]) { // funB
return YES;
} else if ([self.objc respondsToSelector:aSelector]) { // printNumber:
return YES;
} else if ([NSStringFromSelector(aSelector) isEqualToString:@"funC"]) {
return YES;
} else {
return [super respondsToSelector:aSelector];
}
}
```

ClassC 的 funOfClassC 方法：
```
// 外部调用的 funC 会修改为 funOfClassC
- (void)funOfClassC {
NSLog(@"%@ %@", self.class, NSStringFromSelector(_cmd));
}
```

ClassC 的 printNumber 方法：
```
// 打印并返回传入的 NSNumber
- (NSNumber *)printNumber:(NSNumber *)num {
NSLog(@"[%@ %@], number = %@", self.class, NSStringFromSelector(_cmd), num);
return num;
}
```

系统会先调用 ClassA 的 methodSignatureForSelector 方法获取方法签名：
```
// 系统创建 NSInvocation 对象时调用，比如消息转发
// 返回 nil 不会调用 forwardInvocation ，然后 unrecognized selector 崩溃
// NSMethodSignature 的函数类型不对的话，可能会崩溃
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
if ([NSStringFromSelector(aSelector) isEqualToString:@"funC"]) {
// 自己创建函数签名
// funC 会替换为 funOfClassC，因此要返回后者的方法签名
// 函数声明 - (void)funOfClassC
// "v@:" 表示返回值 是 void，形参分别是对象、SEL
NSMethodSignature *ms = [NSMethodSignature signatureWithObjCTypes:"v@:"];
return ms;

} else if ([NSStringFromSelector(aSelector) isEqualToString:@"printNumber:"]) {
// 可以通过能响应 aSelector 的对象获取
NSMethodSignature *signature = [self.objc methodSignatureForSelector:aSelector];
if (signature) {
return signature;
} else {
// 也可以自己创建
// 函数声明 - (NSNumber *)printNumber:(NSNumber *)num
// "@@:@" 表示返回值是对象，形参分别是对象、SEL、对象
NSMethodSignature *ms = [NSMethodSignature signatureWithObjCTypes:"@@:@"];
return ms;
}
} else {
return [super methodSignatureForSelector:aSelector];
}
}
```

ClassA 的 forwardInvocation 方法：
```
// 如果 methodSignatureForSelector: 返回 nil，系统不会调用本函数
// 这里如果什么都不做也不会崩溃，因为重写了父类的同名方法。
// NSObject 默认实现是调用 doesNotRecognizeSelector 抛出异常。
- (void)forwardInvocation:(NSInvocation *)anInvocation {

//  1、演示修改 target 和 SEL
// 这里是把 SEL funC 替换为 funOfClassC
// 因此是通过字符串来判断，而不是 respondsToSelector
if ([NSStringFromSelector(anInvocation.selector) isEqualToString:@"funC"]) {
// 修改消息的 SEL
anInvocation.selector = @selector(funOfClassC);
// 修改消息的 target
anInvocation.target = self.objc;
// 发送消息
[anInvocation invoke];
// 也可以直接发送消息，不用修改 target
//        [anInvocation invokeWithTarget:self.objc];

} else if ([NSStringFromSelector(anInvocation.selector) isEqualToString:@"printNumber:"]) {
// 2、演示修改参数和捕获返回值
// 函数声明 - (NSNumber *)printNumber:(NSNumber *)num
// 外部调用 [obj performSelector:@selector(printNumber:) withObject:@(222)];

// 查看参数
// index 0 和 1 是隐藏参数 self 和 _cmd
//        id obj = nil; // 会闪退
//        void *obj = nil; // 不会闪退
__autoreleasing id obj = nil; // 不会闪退
[anInvocation getArgument:&obj atIndex:0];        
SEL sel = 0; // printNumber:
[anInvocation getArgument:&sel atIndex:1];
__autoreleasing NSNumber *argument = nil; // 外部传入 222
[anInvocation getArgument:&argument atIndex:2];

// 先调用一次
[anInvocation invokeWithTarget:self.objc];
// 获取返回值
__autoreleasing NSNumber *result = nil;
[anInvocation getReturnValue:&result];
NSLog(@"ClassA result1 = %@", result); // 222

// 修改传进来的参数
__autoreleasing NSNumber *number = @444;
[anInvocation setArgument:&number atIndex:2];

// 再调用一次
[anInvocation invokeWithTarget:self.objc]; 
// 获取返回值
[anInvocation getReturnValue:&result];
NSLog(@"ClassA result2 = %@", result); // 444

// 修改返回值，外部获取的返回值就是 666
result = @666;
[anInvocation setReturnValue:&result];

} else {
[super forwardInvocation:anInvocation];
}
}
```


输出结果：
```
// 修改消息的 target 和 SEL
// 消息 [a funC] 修改为 [c funOfClassC]
ClassC funOfClassC

// 第一次转发消息
[ClassC printNumber:], number = 222
// 捕获的返回值
ClassA result1 = 222

// 修改参数后，第二次转发消息
[ClassC printNumber:], number = 444
// 修改参数后捕获的返回值
ClassA result2 = 444

// ClassA 修改消息的返回值为 666
// 外部获取修改过的返回值
外部获取的返回值 resutl = 666
```

外部给 a 发送 printNumber 消息，a 在消息转发过程中，先给 c 发送一次消息并获取函数返回值是 222，然后修改参数为 444，再发送一次消息并获取返回值是 444，最后修改返回值为 666，外部获取的函数返回值就是 666。

消息转发可以做很多事情，比如修改消息的 target、SEL、参数，修改和捕获函数的返回值，可以 invoke 修改后的消息无数次。

说一下上面提到的闪退问题：
```
id obj = nil; // 会闪退
// void *obj = nil; // 不会闪退
// __autoreleasing id obj = nil; // 不会闪退
[anInvocation getArgument:&obj atIndex:0];        
obj = nil; // obj 原来指向的对象，引用数会 -1
```
定义 id obj，默认修饰符是 __strong，通过 getArgument: 函数把 obj 指向对象 target，但由于不是 obj = target 这种方式赋值，编译器不会增加 target 的引用数，当 obj = nil 时，target 的引用数就会 -1，如果 == 0 了，target 就会废弃。这时候再给 target 发送消息，或者外部再次释放 target，就会崩溃，可以开启僵尸对象调试。

使用 __autoreleasing 修饰就不会崩溃，因为会把 obj 指向的对象注册到自动释放池，引用数 +1。释放池废弃的时候，引用数 -1。


<br>

# 四、相关函数总结
---

第一阶段：
```
// 动态解析实例方法
+(BOOL)resolveInstanceMethod:(SEL)sel ;

// 动态解析类方法
+(BOOL)resolveClassMethod:(SEL)sel ;

// 添加方法
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);

// 获取方法实现
IMP class_getMethodImplementation(Class _Nullable cls, SEL name) 
```

第二阶段：
```
// 返回消息转发的target
- (id)forwardingTargetForSelector:(SEL)aSelector
```

第三阶段：
```
// 转发消息
- (void)forwardInvocation:(NSInvocation *)invocation;

// 获取方法签名
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector

// 创建方法签名
+ (nullable NSMethodSignature *)signatureWithObjCTypes:(const char *)types;
```

NSInvocation 方法：
```
// 消息的 target 和 SEL，可以直接修改
@property (nullable, assign) id target;
@property SEL selector;

// 获取和修改返回值
- (void)getReturnValue:(void *)retLoc;
- (void)setReturnValue:(void *)retLoc;

// 获取和修改参数，注意 index 0 和 1 是 self 和 _cmd
- (void)getArgument:(void *)argumentLocation atIndex:(NSInteger)idx;
- (void)setArgument:(void *)argumentLocation atIndex:(NSInteger)idx;

// 发送消息
- (void)invoke;
- (void)invokeWithTarget:(id)target;
```


<br>

参考资料：
* 《Effective Objective-C 2.0  编写高质量iOS与OS X代码的52个有效方法》
* [Objective-C Runtime Programming Guide]([https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtForwarding.html#//apple_ref/doc/uid/TP40008048-CH105-SW1](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtForwarding.html#//apple_ref/doc/uid/TP40008048-CH105-SW1)
)



<br>
