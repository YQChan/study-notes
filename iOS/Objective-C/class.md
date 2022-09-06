# 类与对象

## 对象初始化

创建一个简单的类

```objc
// Person.h
#import <Foundation/Foundation.h>

@interface Person : NSObject

@end

// Person.m
#import "Person.h"

@implementation Person

@end

// main.m
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *person = [[Person alloc] init];
    }
    return 0;
}
```

在objc源码中查看 alloc 的实现，可以看到alloc 调用栈，并在这几个方法中设置断点

```objc
// NSObject.mm
@implementation NSObject

+ (id)alloc {
    return _objc_rootAlloc(self);
}

@end

// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

// Call [cls alloc] or [cls allocWithZone:nil], with appropriate
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
#if __OBJC2__
    if (slowpath(checkNil && !cls)) return nil;
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif

    // No shortcuts available.
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));
}

// objc-runtime-new.mm
id
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone __unused)
{
    // allocWithZone under __OBJC2__ ignores the zone parameter
    return _class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC);
}
```

运行调试发现，方法调用顺序为：callAlloc -> alloc -> _objc_rootAlloc -> callAlloc -> _objc_rootAllocWithZone，callAlloc 方法调用了两次，
而第一次调用 callAlloc 是在 objc_alloc 方法，全局搜索这个方法，可以找到以下实现：

```objc
// objc-runtime-new.mm
/***********************************************************************
* fixupMessageRef
* Repairs an old vtable dispatch call site. 
* vtable dispatch itself is not supported.
**********************************************************************/
static void 
fixupMessageRef(message_ref_t *msg)
{    
    msg->sel = sel_registerName((const char *)msg->sel);

    if (msg->imp == &objc_msgSend_fixup) { 
        if (msg->sel == @selector(alloc)) {
            msg->imp = (IMP)&objc_alloc;
        } else if (msg->sel == @selector(allocWithZone:)) {
            msg->imp = (IMP)&objc_allocWithZone;
        } else if (msg->sel == @selector(retain)) {
            msg->imp = (IMP)&objc_retain;
        } else if (msg->sel == @selector(release)) {
            msg->imp = (IMP)&objc_release;
        } else if (msg->sel == @selector(autorelease)) {
            msg->imp = (IMP)&objc_autorelease;
        } else {
            msg->imp = &objc_msgSend_fixedup;
        }
    } 
    else if (msg->imp == &objc_msgSendSuper2_fixup) { 
        msg->imp = &objc_msgSendSuper2_fixedup;
    } 
    else if (msg->imp == &objc_msgSend_stret_fixup) { 
        msg->imp = &objc_msgSend_stret_fixedup;
    } 
    else if (msg->imp == &objc_msgSendSuper2_stret_fixup) { 
        msg->imp = &objc_msgSendSuper2_stret_fixedup;
    } 
#if defined(__i386__)  ||  defined(__x86_64__)
    else if (msg->imp == &objc_msgSend_fpret_fixup) { 
        msg->imp = &objc_msgSend_fpret_fixedup;
    } 
#endif
#if defined(__x86_64__)
    else if (msg->imp == &objc_msgSend_fp2ret_fixup) { 
        msg->imp = &objc_msgSend_fp2ret_fixedup;
    } 
#endif
}
```

由此得出 alloc 方法整个调用流程：alloc -> objc_alloc -> callAlloc -> objc_msgSend -> alloc -> _objc_rootAlloc -> callAlloc -> _objc_rootAllocWithZone -> _class_createInstanceFromZone

```objc
// objc-runtime-new.mm

/***********************************************************************
* class_createInstance
* fixme
* Locking: none
*
* Note: this function has been carefully written so that the fastpath
* takes no branch.
**********************************************************************/
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        obj = (id)calloc(1, size);
    }
    if (slowpath(!obj)) {
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) {
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

在 _class_createInstanceFromZone 可以看到对象初始化的流程：

1. cls->instanceSize：计算所需内存空间大小，进行内存对齐。一个 NSObject 至少占用16字节。

2. calloc：向系统申请开辟内存，返回地址指针。

3. objc->initInstanceIsa：isa 指针关联相应的类，并将新对象的引用计数设置为1。

说明在 alloc 方法已经完成了对象初始化过程。

在源码查看 init 方法的实现

```objc
// NSObject.mm
@implementation NSObject

- (id)init {
    return _objc_rootInit(self);
}

@end

id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```

init 方法只是返回了当前对象。init 方式的设计相当于一种工厂模式，在子类重写该方法，进行如一些变量初始化赋值。

## 对象的结构

打开终端，cd 进入 main.m 文件所在路径，使用 clang 把文件编译成 cpp 文件。

```shell
clang -rewrite-objc main.m -o main.cpp
```

打开 main.cpp 文件，并查找 Person 声明

```c
#ifndef _REWRITER_typedef_Person
#define _REWRITER_typedef_Person
typedef struct objc_object Person;
typedef struct {} _objc_exc_Person;
#endif

struct Person_IMPL {
    struct NSObject_IMPL NSObject_IVARS;
    NSString *name;
    int age;
};

struct NSObject_IMPL {
    Class isa;
};
```

可以看到，Person 对象本质都是一个 objc_object 的结构体，Person_IMPL 里面有 NSObject_IMPL 和成员变量的值，NSObject_IMPL 包含了一个 Class 类型的 isa 指针。

对象里面存储了一个isa指针 + 成员变量的值，isa指针是固定的，占8个字节，所以影响对象内存的只有成员变量（属性会自动生成带下划线的成员变量）。

### isa 结构

查看源码找到 isa

```objc
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    uintptr_t bits;

private:
    // Accessing the class requires custom ptrauth operations, so
    // force clients to go through setClass/getClass by making this
    // private.
    Class cls;

public:
#if defined(ISA_BITFIELD)
    struct {
         uintptr_t nonpointer        : 1;//->表示使用优化的isa指针
         uintptr_t has_assoc         : 1;//->是否包含关联对象
         uintptr_t has_cxx_dtor      : 1;//->是否设置了析构函数，如果没有，释放对象更快
         uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 ->类的指针
         uintptr_t magic             : 6;//->固定值,用于判断是否完成初始化
         uintptr_t weakly_referenced : 1;//->对象是否被弱引用
         uintptr_t deallocating      : 1;//->对象是否正在销毁
         uintptr_t has_sidetable_rc  : 1;//1->在extra_rc存储引用计数将要溢出的时候,借助Sidetable(散列表)存储引用计数,has_sidetable_rc设置成1
         uintptr_t extra_rc          : 19;  //->存储引用计数
    };

    bool isDeallocating() {
        return extra_rc == 0 && has_sidetable_rc == 0;
    }
    void setDeallocating() {
        extra_rc = 0;
        has_sidetable_rc = 0;
    }
#endif

    void setClass(Class cls, objc_object *obj);
    Class getClass(bool authenticated);
    Class getDecodedClass(bool authenticated);
}
```

可以看到 isa 本质是一个8字节联合体，而且使用了位域优化内存分配，里面存放了类对象的指针。nonPointerIsa是内存优化的一种手段。isa是一个Class类型的结构体指针，占8个字节，主要是用 来存内存地址的。但是8个字节意味着它就有8*8=64位。存储地址根本不需要这么多的内存空间。而且每个对象都有个isa指针，这样就浪费了内存。所以苹果就把和对象一些息息相关的东⻄，存在了这块内存空间里面。这种isa指针就叫nonPointerIsa。

## 类的结构

```objc
typedef struct objc_class *Class;

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
};

struct class_data_bits_t {

    // ...省略
public:
    class_rw_t* data() {
        return (class_rw_t *)(bits & FAST_DATA_MASK);
    }
    // ...省略
}

struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;
    // ...省略
}

struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    const method_list_t * baseMethods;
    const protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    const property_list_t *baseProperties;
};
```

objc_class继承了objc_object，结构如下：

isa：objc_object 指向类，objc_class 指向元类。

superclass：指向父类。

cache：存储用户消息转发优化的方法缓存和 vtable 。

bits：class_rw_t 和 class_ro_t ，保存了方法、协议、属性等列表和一些标志位。

__总结：__ 类也是对象，类对象有且只有一个。类对象本质为objc_class结构体。类对象里面存储了类的父 类、属性、实例方法、协议、成员变量、方法缓存等等。

### isa

isa 是一个 Class 类型的指针。每个实例对象有个isa的指针,他指向对象的类，而Class里也有个isa的指针, 指向metaClass(元类)。元类保存了类方法的列表。当类方法被调用时，先会从本身查找类方法的实现，如果没有，元类会向他父类查找该方法。同时注意的是：元类（metaClass）也是类，它也是对象。元类也有isa指针,它的isa指针最终指向的是一个根元类（NSObject metaClass）。根元类的isa指针指向本身，这样形成了一个封闭的内循环。

元类设计是为了复用一套消息发送机制（objc_msgSend(id self, SEL)）（消息接受者，方法名），符合单一职责的设计原则。如果没有元类，需要objc_msgSend增加参数区分类方法和实例方法，影响消息发送效率。

### class_rw_t 和 class_ro_t

class_rw_t是结构体类型，提供了获取属性列表，方法列表，协议列表的方法。

class_ro_t是结构体类型，有一个ivar_list_t * ivars变量。

ro属于clean memory，在编译时即确定的内存空间，只读，加载后不会发生改变的内存空间，包括类名称、方法、协议和实例变量的信息；

rw的数据空间属于dirty memory，rw是运行时的结构，可读可写，由于其动态性，可以往类中添加属性、方法、协议。在运行时会发生变更的内存。

rw数据什么时候初始化呢？—— realizeClassWithoutSwift，类初始化的时候！在类初始化流程中，从macho中获取的数据data()，强制装换为class_ro_t，同时初始化class_rw_t的空间，并复制一份ro的数据放入rw中。

class_rw_ext_t可以减少内存的消耗。苹果在wwdc2020里面说过，只有大约10%左右的类需要动 态修改。所以只有10%左右的类里面需要生成class_rw_ext_t这个结构体。这样的话，可以节约很大一部分内存。

class_rw_ext_t生成的条件：用过runtime的Api进行动态修改的时候；有分类的时候，且分类和本类都为非懒加载类的时候。实现了+load方法即为非懒加载类。
