![Runtime](https://upload-images.jianshu.io/upload_images/1235154-6e5299b67db5d4e3.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


Runtime API 讲解，附加 demo。

官方文档 [Objective-C Runtime]([https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc#see-also](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc#see-also)
)
类型编码 [Objective-C Runtime Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048) > [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)。


<br>

##  Working with Classes

---


获取类名：
```
const char * class_getName(Class cls);

const char *className = class_getName(Student.class);
NSLog(@"%s", className); // Student
```


是否元类：
```
BOOL class_isMetaClass(Class cls);

BOOL isMetaClass = class_isMetaClass(Student.class); // NO
```


获取实例变量(成员变量)：
```
Ivar class_getInstanceVariable(Class cls, const char *name)

Ivar aVar = class_getInstanceVariable(Student.class, "anInstanceVar");
const char *varName = ivar_getName(aVar); // anInstanceVar
```

获取类变量：
```
// 其实没有类变量
// 它只是返回 class_getInstanceVariable(cls->ISA(), name)
Ivar class_getClassVariable(Class cls, const char *name)

// nil
Ivar classVar = class_getClassVariable(Student.class, "anInstanceVar"); 
```

获取实例变量列表：
```
// 获取实例变量列表，不包括父类的。
// 注意调用 free 释放数组。
Ivar * class_copyIvarList(Class cls, unsigned int * outCount)


@interface Student : Person 
{
    NSString *anInstanceVar;
}
@property (nonatomic, assign) float score; // 分数
@property (nonatomic, strong) NSString *number; // 学号
@end


unsigned int outCount = 0;
Ivar *varList = class_copyIvarList(Student.class, &outCount);
for (int i = 0; i < outCount; i++) {
    Ivar var = varList[i];
    const char *varName = ivar_getName(var);
    // 输出 anInstanceVar, _score, _number
    NSLog(@"%s", varName);
}
free(varList);
```


获取属性：
```
objc_property_t class_getProperty(Class cls, const char * name)

objc_property_t aProperty = class_getProperty(Student.class, "score");
const char *pName = property_getName(aProperty); // score
```

获取属性列表：
```
// 获取属性列表，不包括父类的。
// 注意调用 free 释放数组。
objc_property_t * class_copyPropertyList(Class cls, unsigned int * outCount)


@property (nonatomic, assign) float score; // 分数
@property (nonatomic, strong) NSString *number; // 学号


unsigned int pCount = 0;
objc_property_t *pList = class_copyPropertyList(Student.class, &pCount);
for (int i = 0; i < pCount; i++) {
    objc_property_t property = pList[i];
    const char *pName = property_getName(property);
    NSLog(@"%s", pName); // score, number
}
free(pList);
```


添加方法：
```
// 添加成功返回 YES
// cls 是类或者元类。
// 元类可以通过 objc_getMetaClass("Student") 获取
// 元类可以通过 object_getClass([Student class]) 获取
// imp 是函数实现，对应的函数至少有两个参数(self and _cmd)
// imp 可以是 C语言函数名
// imp 也可以通过 class_getMethodImplementation(class, SEL) 获取
// types 是函数类型，详情看 Objective-C Runtime Programming Guide 的 Type Encodings 章节
// types 至少包含 "@:"，因为 OC 函数最少有两个参数。
// types "v@:" 表示返回类型 void，两个形参分别是对象和 SEL 类型。
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types);


SEL selToAdd = @selector(unKnowSelector);
IMP impToAdd = (void *)myMethodIMP;
Class classToAdd = Student.class; // 类
classToAdd = objc_getMetaClass("Student"); // 元类
classToAdd = object_getClass([Student class]); // 元类
// YES
BOOL addSuccess = class_addMethod(classToAdd, selToAdd, impToAdd, "v@:"); 


// OC 函数转换为 C语言函数，至少有两个参数
void myMethodIMP(id self, SEL _cmd)
{
    // implementation ....
}
```

获取实例方法：
```
// 会搜索父类
Method class_getInstanceMethod(Class cls, SEL name)

SEL instanceSel = @selector(studentInstanceMethod);
Method instanceMethod = class_getInstanceMethod(Student.class, instanceSel);
```


获取类方法：
```
// 会搜索父类
Method class_getClassMethod(Class cls, SEL name);

SEL classSel = @selector(studentClassMethod);
Method classMethod = class_getClassMethod(Student.class, classSel);
```

获取方法列表：
```
// 传类只获取实例方法列表，不包括类方法，不搜索父类
// 传元类获取类方法列表
// 元类可以通过 objc_getMetaClass("Student") 获取
// 元类可以通过 object_getClass([Student class]) 获取
// 注意调用 free() 释放数组
Method  * class_copyMethodList(Class cls, unsigned int *outCount);


@interface Student : Person
@property (nonatomic, assign) float score; // 分数
@property (nonatomic, strong) NSString *number; // 学号
+ (void)studentClassMethod;
- (void)studentInstanceMethod;
@end


unsigned int methodCount = 0;
Class targetClass = Student.class; // 类
targetClass = object_getClass(Student.class); // 元类
Method *methodList = class_copyMethodList(targetClass, &methodCount);
for (int i = 0; i < methodCount; i++) {
    Method method = methodList[i];
    SEL name = method_getName(method);
    NSLog(@"%@", NSStringFromSelector(name));
}
free(methodList);

// 类方法列表输出
// studentClassMethod

// 实例方法列表输出
// studentInstanceMethod
// number
// setNumber:
// setScore:
// score
// .cxx_destruct
```

替换方法：
```
// 如果要替换的方法不存在，就会添加方法
// 如果存在，就把 name 对应的方法的 imp 替换掉
// 参数可以参考 class_addMethod 方法
IMP class_replaceMethod(Class cls, SEL name, IMP imp, const char *types);
```

获取方法实现：
```
// 可能比 method_getImplementation() 快
IMP class_getMethodImplementation(Class cls, SEL name);

IMP impp = class_getMethodImplementation(NSString.class, @selector(compare));
```


获取协议列表：
```
// 不包括父类、其他协议遵循的协议
// Any protocols adopted by superclasses or other protocols are not included.
// 注意调用 free() 释放数组
Protocol ** class_copyProtocolList(Class cls, unsigned int *outCount);


@interface Student : Person <UITableViewDelegate, UITableViewDataSource>
@end


unsigned int protocolCount = 0;
__unsafe_unretained Protocol **plist = class_copyProtocolList(Student.class, &protocolCount);
for (int i = 0; i < protocolCount; i++) {
    Protocol *p = plist[i];
    const char *name = protocol_getName(p);
    // 输出 UITableViewDelegate、UITableViewDataSource
    NSLog(@"%s", name);
}
free(plist);
```

<br>

##  Adding Classes
---

创建类的步骤：
1. 调用 objc_allocateClassPair 创建类和元类；
2. 调用 class_addMethod 、class_addIvar 等方法修改类；
3. 调用 objc_registerClassPair 注册类，然后就可以正常使用了。
4. 类方法要添加到元类。实例方法和实例变量要新建类自己添加给自己。

创建类：
```
// superclass 可以是 nil
// name 是类名，比如 "Student"
// extraBytes 一般是 0
// 失败返回 Nil，例如同名的类已经存在
Class objc_allocateClassPair(Class superclass, const char *name, size_t extraBytes);
```

消除类：
```
// 消除类和相关元类
// cls 必须是通过 objc_allocateClassPair 创建的
void objc_disposeClassPair(Class cls);
```

注册类：
```
// cls 必须是通过 objc_allocateClassPair 创建的
// 创建的类必须注册后才能使用
void objc_registerClassPair(Class cls);
```

<br>


##  Working With Instances
---

修改实例变量的值：
```
// 通过字符串获取变量并且修改值，ARC 环境不能使用
// name 变量名
// value 实例变量的新值
// 返回值是要修改的 Ivar
Ivar object_setInstanceVariable(id obj, const char *name, void *value);

Student *stu = [Student new];
Ivar bVar = object_setInstanceVariable(stu, "_name", @"南瓜");
```

修改实例变量的值：
```
// 修改 Ivar 的值
void object_setIvar(id obj, Ivar ivar, id value);

// @property (nonatomic, strong) NSString *name; // 姓名
Student *stu = [Student new];
Ivar aVar = class_getInstanceVariable(Student.class, "_name");
object_setIvar(stu, aVar, @"西瓜");
NSLog(@"%@", stu.name); // 西瓜
```

获取实例变量的值：
```
id object_getIvar(id obj, Ivar ivar);

// @property (nonatomic, strong) NSString *name; // 姓名
Student *stu = [Student new];
stu.name = @"西瓜";
Ivar aVar = class_getInstanceVariable(Student.class, "_name");
id value = object_getIvar(stu, aVar);
NSLog(@"%@", value); // 西瓜
```

获取实例变量的值：
```
// 通过字符串获取实例变量和变量的值，ARC 环境不可用
// 注意区分 class_getInstanceVariable (Class cls, const char *name) 是获取变量 Ivar
Ivar object_getInstanceVariable(id obj, const char *name, void **outValue);
```

获取类名：
```
// 这个也可以 const char * class_getName(Class cls);
const char * object_getClassName(id obj);

// Student
const char *name = object_getClassName(stu);
```

设置类：
```
// 设置一个对象的类，返回对象原来的类
Class object_setClass(id obj, Class cls);
```

获取类：
```
// 获取对象的类。传入 nil 返回 Nil
// 由于类也是对象，如果传入的参数是类对象，则返回元类
Class object_getClass(id obj);

// 方式 1
Class class = objc_getClass("Student"); // 类
BOOL isMeta = class_isMetaClass(class); // NO
class = objc_getMetaClass("Student"); // 元类
isMeta = class_isMetaClass(class); // YES

// 方式 2
class = object_getClass([Student new]); // 类
isMeta = class_isMetaClass(class); // NO
class = object_getClass([Student class]); // 元类
isMeta = class_isMetaClass(class); // YES
```

<br>


##  Obtaining Class Definitions
---

获取类：
```
id objc_getClass(const char *name);

Class class = objc_getClass("Student"); // 类
```

获取元类：
```
id objc_getMetaClass(const char *name);

Class class = objc_getMetaClass("Student"); // 元类
```

<br>

##  Working with Instance Variables
---

获取实例变量名：
```
const char * ivar_getName(Ivar v);

Ivar aVar = class_getInstanceVariable(Student.class, "anInstanceVar");
const char *varName = ivar_getName(aVar); // anInstanceVar
```

获取实例变量类型：
```
const char * ivar_getTypeEncoding(Ivar v);

unsigned int outCount = 0;
Ivar *varList = class_copyIvarList(self.class, &outCount);
for (int i = 0; i < outCount; i++) {
    Ivar aVar = varList[i];
    const char *varName = ivar_getName(aVar);
    const char *varType = ivar_getTypeEncoding(aVar);
    printf("varName = %12s, \t varType = %s \n", varName, varType);
}
free(varList);

// 属性定义
@interface ViewController : UIViewController
@property (nonatomic, assign) int aInt;
@property (nonatomic, assign) NSInteger aNSInteger;
@property (nonatomic, assign) long aLong;
@property (nonatomic, assign) float aFloat;
@property (nonatomic, assign) double aDouble;
@property (nonatomic, assign) char aChar;
@property (nonatomic, assign) void *aPointer;

@property (nonatomic, copy) void (^aBlock) (void);
@property (nonatomic, strong) id aid;
@property (nonatomic, strong) NSArray *anArray;
@property (nonatomic, strong) NSDictionary *aDic;
@property (nonatomic, strong) Student *aStudent;
@end

// 控制台输出
//   varName =        _aInt,      varType = i
//   varName =  _aNSInteger,      varType = q
//   varName =       _aLong,      varType = q
//   varName =      _aFloat,      varType = f
//   varName =     _aDouble,      varType = d
//   varName =       _aChar,      varType = c
//   varName =    _aPointer,      varType = ^v

//   varName =      _aBlock,      varType = @?
//   varName =         _aid,      varType = @
//   varName =     _anArray,      varType = @"NSArray"
//   varName =        _aDic,      varType = @"NSDictionary"
//   varName =    _aStudent,      varType = @"Student"

i 表示 int，q 表示 long，^v 表示 void 类型指针，
@ 表示对象，id 是 @，
@"NSArray" 表示是 NSArray 类的对象，@"类名"。
```

编码详情 [Objective-C Runtime Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048) > [Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100).


<br>

##  Associative References
---

关联策略 objc_AssociationPolicy：
```
主要是 3 种：
assign(weak)、retain(strong)、copy，
以及 retain、copy 与 nonatomic 结合。

/**
 * Policies related to associative references.
 * These are options to objc_setAssociatedObject()
 */
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_ASSIGN = 0, // assign, weak

    /**< Specifies a strong reference to the associated object.
    *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, // retain, nonatomic

    /**< Specifies that the associated object is copied.
    *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3, // copy, nonatomic

    /**< Specifies a strong reference to the associated object.
    *   The association is made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401, // retain

    /**< Specifies that the associated object is copied.
    *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403 // copy
};

```

设置关联对象：
```
// 把 value 关联到对象 object
// value：如果是 nil，会移除之前关联的对象
// policy: 只要是 weak、strong、copy
void objc_setAssociatedObject(id object, const void *key, id value,
objc_AssociationPolicy policy);


static char associatedKey;
Student *stu = [Student new];
stu.name = @"stu1";
objc_setAssociatedObject(self, &associatedKey, stu, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
Student *stuuu = objc_getAssociatedObject(self, &associatedKey);

// 删除关联对象，传 nil
objc_setAssociatedObject(self, &associatedKey, nil, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

获取关联对象：
```
id objc_getAssociatedObject(id object, const void *key);

static char associatedKey;
Student *stu = [Student new];
stu.name = @"stu1";
objc_setAssociatedObject(self, &associatedKey, stu, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
Student *stuuu = objc_getAssociatedObject(self, &associatedKey);
```

删除所有关联对象：
```
// 删除目标所有关联的对象，最好不要这样操作，可能会误删系统添加的关联对象。
// 可以通过 objc_setAssociatedObject 传 nil 来删除某个 key 关联的对象
void objc_removeAssociatedObjects(id object);
```

<br>


##  Sending Messages
---

编译器会把函数调用转换为发送消息，使用下列中的一个发送：
``` 
objc_msgSend,       objc_msgSend_stret, 
objc_msgSendSuper,  objc_msgSendSuper_stret 
```
使用 super 关键字调用函数，会转换为使用 objc_msgSendSuper 发送消息，否则使用 objc_msgSend。
如果调用函数的返回值类型是 data structures，会转换为使用 objc_msgSendSuper_stret 或 objc_msgSend_stret 发送消息。


发送消息：

```
// self：接收消息的对象
// op：处理消息的方法的 SEL
// ...：方法的参数列表

// 方法的返回值是个对象
id objc_msgSend(id self, SEL op, ...);

// 方法的返回值是 floating-point value
long double objc_msgSend_fpret(id self, SEL op, ...);

// 方法的返回值是 data-structure value
void objc_msgSend_stret(id self, SEL op, ...);

// 消息发给父类，返回值类型是个对象
// 结构体 objc_super 有 id receiver、Class super_class 两个成员变量
id objc_msgSendSuper(struct objc_super *super, SEL op, ...);

// 消息发给父类，返回值类型是 data-structure value
void objc_msgSendSuper_stret(struct objc_super *super, SEL op, ...);



// - (NSString *)myStringMethod:(NSString *)st;
SEL sel = @selector(myStringMethod:);
// 直接调用会报错
// id result = objc_msgSend(self, sel, @"啦啦啦");
// 转换为函数指针 pointer，前两个是隐参 self、隐参 _cmd
NSString * (*pointer) (id, SEL, NSString *) 
 = (NSString *(*)(id, SEL, NSString *))objc_msgSend;
NSString *result = pointer(self, sel, @"啦啦啦");
NSLog(@"%@", result); // 啦啦啦
```


<br>


##  Working with Methods
---
需要 #import <objc/message.h>

调用方法：
```
// receiver： 方法所在的类的对象
// m： 要调用的方法
// ...： 方法的参数
// 更快：This is faster than calling method_getImplementation() 
// and method_getName().
// 转换：These functions must be cast to an appropriate 
// function pointer type before being called.
id method_invoke(id receiver, Method m, ...);


SEL sel = @selector(method_invokeTest:);
Method method = class_getInstanceMethod(self.class, sel);
// 直接调用会报错
// NSString *str = method_invoke(self, method, @"啦啦啦"); 
// 调用前，转换为函数指针 pointer
NSString * (*pointer) (id, Method, NSString *) 
 = (NSString * (*) (id, Method, NSString *))method_invoke;
NSString *str = pointer(self, method, @"啦啦啦");
NSLog(@"@%", str); // 啦啦啦
```

获取方法名：
```
// 返回值是 SEL
// 获取字符串可以继续调用 sel_getName(SEL)
SEL method_getName(Method m);


SEL sel = @selector(methodTest:);
Method method = class_getInstanceMethod(self.class, sel);
SEL name = method_getName(method);
const char *cname = sel_getName(name);
printf("%s", cname); // methodTest:
```

获取方法实现：
```
// 获取 IMP 后可以多次直接调用
IMP method_getImplementation(Method m);


// - (NSString *)myStringMethod:(NSString *)str;
SEL sel = @selector(myStringMethod:);
Method method = class_getInstanceMethod(self.class, sel);
IMP imp = method_getImplementation(method);
// 通过函数指针来调用方法实现
// OC 函数两个隐藏参数 self 和 _cmd，因此要声明 3 个参数
NSString * (*pointer) (id, SEL, NSString *) 
 = (NSString * (*) (id, SEL, NSString *))imp;
NSString *result = pointer(self, sel, @"啦啦啦");
NSLog(@"%@", result); // 啦啦啦
```

获取方法类型编码：
```
// 返回类型、参数类型
const char * method_getTypeEncoding(Method m);


// - (NSString *)myStringMethod:(NSString *)str;
SEL sel = @selector(myStringMethod:);
Method method = class_getInstanceMethod(self.class, sel);
const char *type = method_getTypeEncoding(method);
// @24@0:8@16，去掉数字 @@:@，第一个是返回值类型
// 表示返回值是对象，参数类型分别是对象、SEL、对象
// @@:@ 分别是 返回值、隐参 self、隐参 _cmd、参数 str
printf("%s", type); // @24@0:8@16
```

获取方法返回值类型：
```
void method_getReturnType(Method m, char *dst, size_t dst_len);

// - (NSString *)myStringMethod:(NSString *)str;
SEL sel = @selector(myStringMethod:);
Method method = class_getInstanceMethod(self.class, sel);
char type[10] = {};
method_getReturnType(method, type, sizeof(type));
printf("%s", type); // @，表示是个对象
```

获取方法参数类型、参数数量：
```
// 获取参数数量
unsigned int method_getNumberOfArguments(Method m);

// 获取参数类型
void method_getArgumentType(Method m, unsigned int index, char *dst, size_t dst_len);


// - (NSString *)myStringMethod:(NSString *)str;
SEL sel = @selector(myStringMethod:);
Method method = class_getInstanceMethod(self.class, sel);
unsigned int count = method_getNumberOfArguments(method);
for (int i = 0; i < count; i++) {
    char type[10] = {};
    method_getArgumentType(method, i, type, sizeof(type));
    // 0, @     1, :     2, @
    // 对应 self，_cmd， str
    printf("%i, %s \t", i, type);
}
```

获取方法描述：
```
struct objc_method_description * method_getDescription(Method m);

/// Defines a method
struct objc_method_description {
    SEL name;               /**< The name of the method */
    char * types;           /**< The types of the method arguments */
};


// - (NSString *)myStringMethod:(NSString *)str;
SEL sel = @selector(myStringMethod:);
Method method = class_getInstanceMethod(self.class, sel);
struct objc_method_description *description = method_getDescription(method);
SEL name = description->name;
const char *types = description->types;
// name = myStringMethod:, types = @24@0:8@16
printf("name = %s, types = %s", sel_getName(name), types);
```

方法交换：
```
// 交换两个方法的方法实现
// 通过 class_getInstanceMethod 获取交换的方法
void method_exchangeImplementations(Method m1, Method m2);

// 是下面语句的原子版
IMP imp1 = method_getImplementation(m1);
IMP imp2 = method_getImplementation(m2);
method_setImplementation(m1, imp2);
method_setImplementation(m2, imp1);


// 看下 AFNetworking 的代码
void af_swizzleSelector(Class theClass, SEL originalSelector, SEL swizzledSelector) {
    Method originalMethod = class_getInstanceMethod(theClass, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(theClass, swizzledSelector);
    method_exchangeImplementations(originalMethod, swizzledMethod);
}
```

<br>


##  Working with Selectors
---

获取选择子名字：
```
const char * sel_getName(SEL sel);
```

注册选择子：
```
// 在添加方法到类之前，要把方法的 SEL 注册到运行时系统。
// Registers a method with the Objective-C runtime system, 
// maps the method name to a selector.
// You must register a method name with the Objective-C runtime 
// system to obtain the method’s selector before you can 
// add the method to a class definition.
SEL sel_registerName(const char *str);

// 通过 class_addMethod 添加方法，需要传一个 SEL 参数
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)

SEL selToAdd = sel_registerName("methodToAdd");
```


比较选择子：
```
// 判断是否相同，等价于 ==
BOOL sel_isEqual(SEL lhs, SEL rhs);
```

<br>

## Working with Properties
---

符号定义请看 [Declared Properties](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html#//apple_ref/doc/uid/TP40008048-CH101) in [Objective-C Runtime Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048)。

获取属性名字：
```
// 比如 school
const char * property_getName(objc_property_t property);

objc_property_t property = class_getProperty(Student.class, "school");
const char *name = property_getName(property);
printf("name = %s", name); // school
```

获取属性的属性字符串：
```
// 返回一个字符串，包含所有属性，属性名+属性值，各个属性用逗号隔开。
// 比如 T@"NSString",C,N,V_school
const char * property_getAttributes(objc_property_t property);


// 定义 @property (nonatomic,copy) NSString *school；
objc_property_t property = class_getProperty(Student.class, "school");
const char *attributes = property_getAttributes(property);
printf("%s", attributes);  // T@"NSString",C,N,V_school

T@"NSString",C,N,V_school
T@"NSString" 表示属性 T 的值是 @"NSString"，@表示是个对象，引号里面是对象的类名。
C 表示 copy，N 表示 nonatomic，值为空，
V 表示实例变量，值是 _school
```

获取属性的属性列表：
```
// 获取 property 的 attributes 数组
// 记得调用 free 释放数组
objc_property_attribute_t * 
property_copyAttributeList(objc_property_t property, unsigned int *outCount);

/// Defines a property attribute
typedef struct {
    const char * _Nonnull name; /**< The name of the attribute */
    const char * _Nonnull value; /**< The value of the attribute (usually empty) */
} objc_property_attribute_t;


// demo
objc_property_t property = class_getProperty(Student.class, "school");
const char *name = property_getName(property);
printf("property name = %s\n", name); // school

const char *attributes = property_getAttributes(property);
printf("attributes = %s\n", attributes); // T@"NSString",C,N,V_school

printf("attribute list ---------\n");
unsigned int count = 0;
objc_property_attribute_t *attList = property_copyAttributeList(property, &count);
for (int i = 0; i < count; i++) {
    objc_property_attribute_t proAtt = attList[i];
    const char *name = proAtt.name;
    const char *value = proAtt.value;
    printf("name = %s, value = %s \n", name, value);
}
free(attList);

// 定义 @property (nonatomic,copy) NSString *school；
// 控制台输出
property name = school
attributes = T@"NSString",C,N,V_school
attribute list ---------
name = T, value = @"NSString" 
name = C, value =  
name = N, value =  
name = V, value = _school 
```

获取属性的单个属性值：
```
// 注意 attributeName 不是 property name
// 获取的值和 objc_property_attribute_t 的成员变量 value 相同
// 记得调用 free 释放
char * property_copyAttributeValue(
objc_property_t property, const char *attributeName);


const char *attValue = property_copyAttributeValue(property, "T");
// attName = T, attValue = @"NSString"
printf("attName = T, attValue = %s", attValue);
free(attValue);
```

<br>

##  Using Objective-C Language Features
---

获取 weak 对象：
```
// 返回 weak 指针指向的对象
// 返回对象前，会 retaining and autoreleasing the object。
// 使用 __weak 变量的表达式都会调用本函数
// 比如 [weakObj doSomething];
id objc_loadWeak(id  _Nullable *location);


```

存储 weak 对象：
```
// location：weak 指针的地址
// obj：weak 指针指向的对象
// 返回值是参数 obj
// 赋值给 weak 指针的语句都会调用本函数。
// 比如 tableView.dataSource = self;
id objc_storeWeak(id  _Nullable *location, id obj);
```


<br>
