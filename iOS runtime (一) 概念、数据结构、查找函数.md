![runtime](https://upload-images.jianshu.io/upload_images/1235154-d52f122a2f5f2cb2.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


概念 class、object，SEL、IMP、method，查找方法的过程。

源代码来自 [objc4-750.tar.gz](https://opensource.apple.com/tarballs/objc4/)。


<br>

## 一、Class 和 Object
---

关于 Class 和 object 的定义，可以看我的文章 [《iOS 类、元类、Block (基于 OC 2.0)》](https://www.jianshu.com/p/707fa6c33fae)。

这里列出简单定义：
```
// id
typedef struct objc_object *id;

// 对象
struct objc_object {
private:
    isa_t isa;
    // 省略部分成员变量以及方法...
}

// Class
typedef struct objc_class *Class;

// 类
struct objc_class : objc_object {
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    // class_rw_t 有类的方法和属性列表。
    class_rw_t *data() { 
        return bits.data();
    }
    // 省略部分成员变量以及方法...
}

// NSObject
@interface NSObject <NSObject> {
    Class isa;
}
```

类和对象都是结构体。
类、元类也是对象。
对象里面封装有个 isa 指针，指向它的类。
类的 isa 指针指向它的元类。
类比对象多封装了方法和属性列表。


<br>

## 二、SEL、IMP、Method
---

#### 2.1 SEL

SEL 看起来是个结构体指针：
```
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```

objc_selector 的定义没找到。

可以通过创建 SEL 的函数追踪：
```
SEL sel_registerName(const char *name) {
    return __sel_registerName(name, 1, 1);     // YES lock, YES copy
}
```

简化后的 __sel_registerName：
```
static SEL __sel_registerName(const char *name, bool shouldLock, bool copy) 
{
    SEL result = 0;

    // 这里省略部分代码

    if (!result) {
        result = sel_alloc(name, copy);
        // fixme choose a better container (hash not map for starters)
        NXMapInsert(namedSelectors, sel_getName(result), result);
    }

    return result;
}

```

调用 sel_alloc，上面传进来的 copy 是 YES： 
```
static SEL sel_alloc(const char *name, bool copy)
{
    selLock.assertLocked();
    return (SEL)(copy ? strdupIfMutable(name) : name);    
}
```
因此看 strdupIfMutable 定义：
```
// strdup that doesn't copy read-only memory
static inline char *
strdupIfMutable(const char *str)
{
    size_t size = strlen(str) + 1;
    if (_dyld_is_memory_immutable(str, size)) {
        return (char *)str;
    } else {
        return (char *)memdup(str, size);
    }
}
```
不管走的是 if 还是 else，可以看到 SEL 就是个 char *。

顺便看下 memdup 的定义：
```
static inline void *
memdup(const void *mem, size_t len)
{
    void *dup = malloc(len);
    memcpy(dup, mem, len);
    return dup;
}
```

大概是分配新的内存并且复制内容，也就是深拷贝一个字符串。

再看下获取 SEL 名字的函数 sel_getName：
```
const char *sel_getName(SEL sel) 
{
    if (!sel) return "<null selector>";
    return (const char *)(const void*)sel;
}
```
直接强制类型转换了。

能找到的定义：
```
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```
因此，SEL 到底是 char * 还是结构体指针 objc_selector * ？
网上说 iOS 和 Mac 是不同的，我觉得在 iOS 中，SEL 应该可以理解为字符串。


<br>

#### 2.2 IMP
IMP 是方法实现的指针，比如 C语言的函数名。
```
/// A pointer to the function of a method implementation. 
typedef void (*IMP)(void /* id, SEL, ... */ ); 
```

OC 函数可以通过 ``` - (IMP)methodForSelector:(SEL)aSelector ``` 获取。

获取 IMP 后，可以直接调用方法实现，举个栗子：
```
void (*setter)(id, SEL, BOOL);
setter = (void (*)(id, SEL, BOOL))[target
    methodForSelector:@selector(setFilled:)];
setter(target, @selector(setFilled:), YES);
```
保存 IMP 需要强制类型转换。
方法 setFilled: 只需要 1 个参数，指针 setter 有 3 个参数，是因为执行方法实现需要把 target 和 SEL 传给它。



<br>

#### 2.3 Method

Method 是结构体 method_t 指针：

```
typedef struct method_t *Method;
```

结构体 method_t ：
```
struct method_t {
    SEL name;
    const char *types;
    MethodListIMP imp; // MethodListIMP 是 IMP 的别名
    // 这里省略了一个 struct SortBySELAddress
};

// using 是定义别名，类似 typedef。
using MethodListIMP = IMP;
```

Method 有 SEL 函数名，char * 函数类型，IMP 方法实现。

举个例子，通过命令 
``` clang -rewrite-objc -fobjc-arc -fobjc-runtime=macosx-10.7 Test.m ```
把自己写的 Test.m 转换为 Test.cpp 文件。

转换的代码如下：
```
static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[6];
} _OBJC_$_INSTANCE_METHODS_Test __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	6,
	{(struct objc_selector *)"dealloc", "v16@0:8", (void *)_I_Test_dealloc},
	{(struct objc_selector *)"myName", "@16@0:8", (void *)_I_Test_myName},
	{(struct objc_selector *)"setMyName:", "v24@0:8@16", (void *)_I_Test_setMyName_}
};
```

上面的代码可以看成是 Test 类的方法列表，其中一个 Method：
```
{
(struct objc_selector *)"setMyName:", 
"v24@0:8@16",
(void *)_I_Test_setMyName_
}
```
前面是 SEL，中间是函数类型，后面是 IMP。

函数类型 "v24@0:8@16" ，v 是 void，@表示一个对象，：表示一个 SEL。数字的意思暂时不懂。

去掉数字就是 "v@:@"，第一个 @ 是接收消息的 target，也就是 Test 类的一个对象，: 表示 SEL也就是当前方法 _cmd，第二个 @ 是传进来的参数。对于类型编码，在我的文章 [《iOS runtime (二) 编程指导》](https://www.jianshu.com/writer#/notebooks/5011690/notes/10405022) 的 Type Encodings 章节有讲解。


<br>

#### 2.4 SEL、IMP、Method 的关系

SEL 包含了函数名，用于匹配 Method。
IMP 是个指针，指向方法实现。
Method 包含了 SEL、IMP、方法类型。
class 通过比较 SEL 是否相同来查找 Method，从而找到 IMP。

<br>

## 三、查找函数的过程
---

OC 用于查找方法实现的函数：
```
- (IMP)methodForSelector:(SEL)aSelector 
``` 

看下它的实现：
```
+ (IMP)methodForSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return object_getMethodImplementation((id)self, sel);
}

- (IMP)methodForSelector:(SEL)sel {
    if (!sel) [self doesNotRecognizeSelector:sel];
    return object_getMethodImplementation(self, sel);
}
```

都是通过 object_getMethodImplementation 查找：
```
IMP object_getMethodImplementation(id obj, SEL name)
{
    Class cls = (obj ? obj->getIsa() : nil);
    return class_getMethodImplementation(cls, name);
}
```

上面的 class_getMethodImplementation：
```
IMP class_getMethodImplementation(Class cls, SEL sel)
{
    IMP imp;
    if (!cls  ||  !sel) return nil;

    imp = lookUpImpOrNil(cls, sel, nil, 
           YES/*initialize*/, YES/*cache*/, YES/*resolver*/);

    if (!imp) {
        return _objc_msgForward;
    }
    return imp;
}
```

上面调用的 lookUpImpOrNil 方法，里面最终会跳转到 lookUpImpOrForward 方法：
```
// 简化后
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;

    // 省略从 cache 查找的过程
    // 省略可能要初始化 class 的过程

    // Try this class's method lists.
    {
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            goto done;
        }
    }

    // 省略从父类查找的过程
    // 省略 method resolver 的过程
    // 省略 method forwarding 的过程

 done:
    return imp;
}
```

上面的方法显示了查找函数实现的大致过程，包括从缓存查找、从当前类和父类查找、方法的 resolver 和 forwarding 过程。

根据 SEL 找到 Method 后，缓存方法并且返回 Method 的 IMP 属性。

看下根据 SEL 找到 Method 的过程。上面调用的 getMethodNoSuper_nolock 方法：
```
static method_t *
getMethodNoSuper_nolock(Class cls, SEL sel)
{
    for (auto mlists = cls->data()->methods.beginLists(), 
              end = cls->data()->methods.endLists(); 
         mlists != end;
         ++mlists)
    {
        method_t *m = search_method_list(*mlists, sel);
        if (m) return m;
    }
    return nil;
}
```

从 methods.beginLists() 遍历到 methods.endLists()。
methods 是一个 method_array_t 类，保存类的方法列表。

上面循环体调用的 search_method_list：
```
static method_t *search_method_list(const method_list_t *mlist, SEL sel)
{
    int methodListIsFixedUp = mlist->isFixedUp();
    int methodListHasExpectedSize = mlist->entsize() == sizeof(method_t);
    
    if (__builtin_expect(methodListIsFixedUp && methodListHasExpectedSize, 1)) {
        return findMethodInSortedMethodList(sel, mlist);
    } else {
        // Linear search of unsorted method list
        for (auto& meth : *mlist) {
            if (meth.name == sel) return &meth;
        }
    }
    return nil;
}
```

if 分支调用的 findMethodInSortedMethodList，内部和 else 分支一样，通过 '==' 运算符，比较传进来的 SEL 和 Method 的 name 属性是否相等。

总结：
Method 包含了 SEL、IMP、方法类型。
Class 通过 SEL 查找 IMP，先是根据 SEL 是否相等找到 Method，然后返回 Method 的 IMP 属性。
