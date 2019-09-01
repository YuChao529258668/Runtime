
主要讲述 Messaging、Method、Type Encodings、Property 等概念，以及 NSObject class 和 OC 代码如何与运行时系统交互。


官方文档 [Objective-C Runtime Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048)。
关于运行时系统的消息转发，可以看我的文章 [《iOS runtime (三) 消息转发》](https://www.jianshu.com/writer#/notebooks/5011690/notes/52190727)。

<br>

# 一、Introduction
---

运行时系统可以让代码推迟到运行的时候，再决定消息如何解析和转发。运行时系统通过检查参数来动态决定加载新类和转发消息给其他对象，提供对象在运行时的信息。

有两个版本的Objective-C运行时：“Modern” 和 “Legacy”。现代版本是以objective-c 2.0引入的，包含了许多新特性。


<br>

# 二、Interacting with the Runtime
---

3 种方式：
* 通过 Objective-C Source Code
编写和编译Objective-C源代码就可以使用它。编译器会创建实现语言动态特性的数据结构和函数调用。
数据结构捕获类和类别定义以及协议声明中的信息；它们包括在用Objective-C编程语言定义类和协议时讨论的类和协议对象，以及方法选择器、实例变量模板、从源代码中提取的其他信息。

* 通过 NSObject Methods
比如 `class`、`isKindOfClass:`、`isMemberOfClass:`、`respondsToSelector:`、`conformsToProtocol:`、`methodForSelector:` 。

* 通过 Runtime Functions
运行时头文件在路径 /usr/include/objc，具体看  [Objective-C Runtime Reference](https://developer.apple.com/documentation/objectivec/objective_c_runtime)。


<br>

# 三、Messaging
---

本章节讲述如何将消息表达式转换为 objc_msgsend 函数调用，以及如何按名称引用方法。然后解释如何利用 objc_msgsend，以及如果需要如何绕过动态绑定。

<br>

### 3.1 The objc_msgSend Function

OC 会在运行时才决定调用哪个函数。
```
// OC 函数调用
[receiver message];

// 转换后的代码
objc_msgSend(receiver, selector);

// 或者是带参数的
objc_msgSend(receiver, selector, arg1, arg2, ...);
```

dynamic binding:
* 首先找到 selector 指向的函数实现(method implementation)；
* 然后调用函数实现，passing it the receiving object (a pointer to its data), 并且传入相关参数。
* 最后，把函数实现的返回值，作为自己的返回值进行传递。


类有个 isa 指针和分发表(dispatch table)。
分发表保存了类的 selector 和 函数实现的地址。

![Figure 3-1  Messaging Framework.png](https://upload-images.jianshu.io/upload_images/1235154-4e7f80185d7cc807.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

给对象发送消息时，先去 isa 指向的类的分发表中查找。如果找不到，就去 superclass 指向的父类的分发表中查找。

为了提高查找速度，会缓存调用过的 selectors and addresses of methods。查找分发表之前，会先在缓存里查找。

<br>

### 3.2 Using Hidden Arguments

objc_msgSend 找到并调用方法实现时，会传递两个隐藏参数：
* 消息接收者
* 方法的 selector

隐藏的意思是，这两个参数并没有在方法的定义中声明。

方法内部有两个参数，一个是接收消息的对象 self，另一个是方法自己的 selector _cmd。

```
- (id)strange
{
    // 通过其他函数获取 target 和 method
    id  target = getTheReceiver();
    SEL method = getTheMethod();
 
    if ( target == self || method == _cmd )
        return nil;
    return [target performSelector:method];
}
```

上面的代码，self 是接收 strange 消息的对象，_cmd 是方法 strange 的 selector。


<br>

### 3.3 Getting a Method Address

避免动态绑定的唯一方式是，直接获取方法的地址，然后像函数一样调用它。

NSObject 的 methodForSelector: 方法可以获取方法实现的指针，然后通过指针来调用方法实现。这个指针要进行类型转换，要考虑返回类型和参数类型。

```
- (IMP)methodForSelector:(SEL)aSelector;
```

举个栗子：
```
void (*setter)(id, SEL, BOOL);
int i;
 
setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
for ( i = 0 ; i < 1000 ; i++ )
    setter(targetList[i], @selector(setFilled:), YES);
```

方法 setFilled: 只有 1 个参数，指针 setter 却有 3 个参数。
上面有提到过，调用方法实现时，要给它传消息接收者和方法的 selector。
因此指针 setter 有 3 个参数，参数 id 就是方法 setFilled: 里面的隐藏参数 self，参数 SEL 就是方法 setFilled: 里面的隐藏参数 _cmd，参数 BOOL 是形参需要的值。

> The first two arguments passed to the procedure are the receiving object (`self`) and the method selector (`_cmd`). These arguments are hidden in method syntax but must be made explicit when the method is called as a function.
Note that methodForSelector: is provided by the Cocoa runtime system; it’s not a feature of the Objective-C language itself.


<br>

# 四、Dynamic Method Resolution
---

本章节讲述如何动态提供方法实现。
有些情形需要动态解析方法，比如使用 dynamic：
```
@dynamic propertyName;
```

dynamic 是告诉编译器，属性的方法会动态提供。

通过实现 resolveInstanceMethod: 和 resolveClassMethod: 方法，来动态提供方法实现。

通过 class_addMethod 方法来添加方法：
```
OBJC_EXPORT BOOL class_addMethod(
Class cls, SEL name, 
IMP imp,   const char * types);
```
添加方法需要 4 个参数，Class，SEL，IMP，方法类型。
方法类型是个字符串，下文有讲解。

Objective-C 的方法最少需要两个参数：self 和 _cmd。

先定义被添加的方法：
```
void dynamicMethodIMP(id self, SEL _cmd) {
    // implementation ....
}
```

然后，you can dynamically add it to a class as a method (called resolveThisMethodDynamically) using resolveInstanceMethod: like this:

```
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(resolveThisMethodDynamically)) {
          // 添加自定义方法
          class_addMethod([self class], aSEL, (IMP)dynamicMethodIMP, "v@:");
          return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
@end
```

意思是，给 MyClass 的对象发送 resolveThisMethodDynamically 消息时，在 resolveInstanceMethod 里面动态添该方法。

所谓的动态添加，并不是说被添加的方法是运行的时候定义的，而是事先写好的方法，只不过在运行的时候添加到类里面。

添加方法的意思是，给 Selector (SEL) 添加一个对应的方法实现 (IMP)。

重点看这句代码：
```
class_addMethod([self class], aSEL, (IMP)dynamicMethodIMP, "v@:");
```

给类添加方法，参数是类 Class，选择子 SEL，方法实现 IMP，方法类型 const char *。

这里的 IMP 是 (IMP)dynamicMethodIMP。dynamicMethodIMP 是 C语言的函数名，也是个函数指针，指向函数实现的地址。和 OC 的函数名是不一样的。

OC 函数的 IMP 可以通过下面的函数获取：
```
- (IMP)methodForSelector:(SEL)aSelector;
```


这里的方法类型是 "v@:"，v 表示 void，@ 表示一个对象，: 表示 selector。
函数定义```  void dynamicMethodIMP(id self, SEL _cmd) ``` ，按顺序对应就是  "v@:"。

类在消息转发之前，有机会动态添加方法。也就是先收到 resolveInstanceMethod: 或 resolveClassMethod: 消息，如果这两个函数返回 NO，运行时系统才会进行消息转发。

小结：
通过实现 resolveInstanceMethod: 或 resolveClassMethod: 函数，在里面调用 class_addMethod 函数来动态添加方法。
关于函数类型，后文的 Type Encodings 章节有解释。

<br>

# 五、Message Forwarding
---

<br>

### 5.1 Forwarding

如果给对象发送无法处理的消息，在抛出错误之前，运行时会给对象发送 forwardInvocation: 消息，参数是个 NSInvocation 对象，封装了原来的消息和参数。

如果实现了 forwardInvocation: 方法，可以避免由于错误引发的崩溃。

假设 A 类有个 negotiate 函数，想让 B 类也能处理 negotiate 消息，有3种方式：
1、B 类继承 A 类；
2、组合的方式，B 类也实现 negotiate 函数，并且调用 A 的；
3、实现 forwardInvocation: 方法，转发消息给 A。

```
// 方式 2
- (id)negotiate
{
    if ( [someOtherObject respondsTo:@selector(negotiate)] )
        return [someOtherObject negotiate];
    return self;
}
```

对于方式 1 和 2，有点笨重，要继承和声明方法，如果有多个方法就比较麻烦，而且不能在运行时决定，不够灵活。

看下方式 3：
```
// 方式 3
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([someOtherObject respondsToSelector:
            [anInvocation selector]])
        [anInvocation invokeWithTarget:someOtherObject];
    else
        [super forwardInvocation:anInvocation];
}
```

转发消息时，可以通过 invokeWithTarget: 方法来发送消息。

NSObject 的 forwardInvocation: 方法，默认只是调用 doesNotRecognizeSelector: 方法。

forwardInvocation: 方法可以成为一个派发中心，把无法识别的消息都转发给不同的对象，或者只是接收无法处理的消息，不让代码崩溃，也可以把几个消息合并为一个消息。

<br>

### 5.2 Forwarding and Multiple Inheritance

转发消息类似多继承。如下图所示，对象 w 通过 forwardInvocation
转发 negotiate 给对象 d 处理，看起来就像是 w 继承了 Diplomat 类。

![Figure 5-1  Forwarding.png](https://upload-images.jianshu.io/upload_images/1235154-a3f5c3739de7e021.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br>

### 5.3 Surrogate Objects

转发消息可以代替对象。假设创建一个大对象很耗费时间和资源，比如要读取大量的文件或复杂的图片，一开始会创建一个轻量级的代理对象来代替它。只有真的需要这个对象，或者系统有空闲时，才去真的创建这个对象。当代理对象第一次收到 forwardInvocation: 消息时，如果对象不存在，则创建它。所有发给该对象的消息，都会经过代理对象。因此，对程序而言，代理对象和该对象是一样的。

<br>

### 5.4 Forwarding and Inheritance

如果通过 forwardInvocation: 扩展类的职责，这个转发机制应该表现为类似于继承，因此可能要重写一些方法，比如：
* respondsToSelector:
* isKindOfClass:
* instancesRespondToSelector:
* conformsToProtocol:
* methodSignatureForSelector:

如果不重写这些方法，其他代码通过 respondsToSelector: 判断的话，可能就不会给这个对象发送消息了，也就没有接下来的转发步骤。

举个栗子，如果能够转发消息，重写如下：
```
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ( [super respondsToSelector:aSelector] )
        return YES;
    else {
        /* Here, test whether the aSelector message can     *
         * be forwarded to another object and whether that  *
         * object can respond to it. Return YES if it can.  */
    }
    return NO;
}
```

Similarly, if an object forwards any remote messages it receives, it should have a version of methodSignatureForSelector: that can return accurate descriptions of the methods that ultimately respond to the forwarded messages; for example, if an object is able to forward a message to its surrogate, you would implement methodSignatureForSelector: as follows:

```
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector
{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
       signature = [surrogate methodSignatureForSelector:selector];
    }
    return signature;
}
```

转发消息是一个高级技术，不能用来代替继承，只有在不得已的情况下才去用。

<br>


# 六、Type Encodings
---

这个章节的编码讲了 OC 的类型编码，比如 int 是 i，float 是 f，
还有 Property 的类型编码，比如 N 是 nonatomic，C 是 copy。

Property 的类型编码，包含在属性描述里，可以通过 property_getAttributes 函数获取。

OC 的类型编码可以通过 @encode(type) 获取类型，返回值是个字符串。

The type can be a basic type such as an int, a pointer, a tagged structure or union, or a class name—any type, in fact, that can be used as an argument to the C sizeof() operator.

举个栗子：
```
char *buf1 = @encode(int **); // ^^i
char *buf2 = @encode(int); // i
char *buf3 = @encode(double); // d
char *buf4 = @encode(void); // v
size_t size = sizeof(buf2); // 8
```

下面的表格是 Objective-C type encodings：

Code | Meaning
:-:|-
c | A char
i | An int
s | A short
l | A long，l is treated as a 32-bit quantity on 64-bit programs.
q | A long long
C | An unsigned char
I | An unsigned int
S | An unsigned short
L | An unsigned long
Q | An unsigned long long
f | A float
d | A double
B | A C++ bool or a C99 _Bool
v | A void
\* | A character string (char *)
@ | An object (whether statically typed or typed id)
\# | A class object (Class)
: | A method selector (SEL)
[array type] | An array
{name=type...} | A structure
(name=type...) | A union
bnum | A bit field of num bits
^type | A pointer to type
? | An unknown type (among other things, this code is used for function pointers)

>Important: Objective-C does not support the long double type. @encode(long double) returns d, which is the same encoding as for double.

<br>

# 七、Declared Properties
---

<br>

### 7.1 Property Type and Functions

```
获取属性列表
objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount)

获取属性名
const char *property_getName(objc_property_t property)

获取属性
objc_property_t class_getProperty(Class cls, const char *name)

获取属性的属性
const char *property_getAttributes(objc_property_t property)
```

demo：
```
@interface ViewController ()
@property (nonatomic, assign) float myFloat;
@end

- (void)test6 {
    Class class = self.class;
    unsigned int count;
    objc_property_t *ps = class_copyPropertyList(class, &count);
    
    for (int i = 0; i < count; i++) {
        objc_property_t p = ps[i]; 
        // myFloat
        const char *name = property_getName(p); 
        // Tf,N,V_myFloat
        const char *atttribute = property_getAttributes(p); 
        fprintf(stdout, "%s, %s \n", name, atttribute);
    }
}
```

属性的属性输出 "Tf,N,V_myFloat"，Tf 表示属性是 float 类型，N 表示 nonatomic，V_myFloat 是 V 和实例变量名 _myFloat 连起来。下文有解释。

<br>

### 7.2 Property Type String

通过 property_getAttributes 函数可以获取 Property 的属性，
以 T 开头，T 后面接着 @encode() 返回的类型，然后是逗号，最后以 V 和实例变量名拼接结束。

比如上个例子的 "Tf,N,V_myFloat"，T 开头，f 是 @encode(float) 的返回值，N 是 nonatomic，结尾是 V 和实例变量名 _myFloat 连起来。

下面表格是 Declared property type encodings。

Code | Meaning
:-:|-
R | The property is read-only (readonly).
C | The property is a copy of the value last assigned (copy).
& | The property is a reference to the value last assigned (retain).
N | The property is non-atomic (nonatomic).
G<name> | The property defines a custom getter selector name. The name follows the G (for example, GcustomGetter,).
S<name> | The property defines a custom setter selector name. The name follows the S (for example, ScustomSetter:,).
D | The property is dynamic (@dynamic).
W | The property is a weak reference (__weak).
P | The property is eligible for garbage collection. 
T<encoding> | Specifies the type using old-style encoding.


官方的例子先给出定义：
```
enum FooManChu { FOO, MAN, CHU };
struct YorkshireTeaStruct { int pot; char lady; };
typedef struct YorkshireTeaStruct YorkshireTeaStructType;
union MoneyUnion { float alone; double down; };
```
然后列出的表格，是以这些定义为 Property 时的属性，摘录一些有意义的，比如:

Property declaration | Property description
:-:|:-:
@property char charDefault; | Tc,VcharDefault
@property enum FooManChu enumDefault; | Ti,VenumDefault
@property struct YorkshireTeaStruct structDefault; | T{YorkshireTeaStruct="pot"i"lady"c},VstructDefault
@property union MoneyUnion unionDefault; | T(MoneyUnion="alone"f"down"d),VunionDefault
@property int (*functionPointerDefault)(char *); | T^?,VfunctionPointerDefault
@property int *intPointer; | T^i,VintPointer
@property(getter=intGetFoo, setter=intSetFoo:) int intSetterGetter; | Ti,GintGetFoo,SintSetFoo:,VintSetterGetter
@property(retain) id idRetain; | T@,&,VidRetain
@property(nonatomic, readonly, retain) id idReadonlyRetainNonatomic; | T@,R,&,VidReadonlyRetainNonatomic

官方例子给的表格很长，上文给出一部分，完整的点这里查看 [Property Attribute Description Examples]([https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101-SW1)
)。


<br>
<br>
<br>
<br>
<br>
<br>













