# 消息发送与转发

## 方法调用本质

在 Person 类里新增 test 实例方法，并在 main 函数调用。

```objc
#import <Foundation/Foundation.h>

@interface Person : NSObject

- (void)test;

@end

// Person.m
#import "Person.h"

@implementation Person

- (void)test {
    NSLog(@"%s", __func__);
}

@end

// main.m
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *person = [[Person alloc] init];
        [person test];
    }
    return 0;
}
```

打开终端，cd 进入 main.m 文件所在路径，使用 clang 把文件编译成 cpp 文件。

```shell
clang -rewrite-objc main.m -o main.cpp
```

打开 main.cpp 文件，并查找 main 函数。

```c
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        Person *person = ((Person *(*)(id, SEL))(void *)objc_msgSend)((id)((Person *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("Person"), sel_registerName("alloc")), sel_registerName("init"));
        ((void (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("test"));
    }
    return 0;
}
```

在iOS中调用方法其实就是在给对象发送某条消息。消息的发送在编译的时候编译器就会把方法转换为objc_msgSend这个函数。objc_msgSend有俩个隐式的参数，消息的接收者和消息的方法名。objc_msgSend这个函数就能够通过这俩个隐式的参数去找到方法具体的实现。如果消息的接收者是实例对象，isa就指向类对象，后再通过第二个参数方法名，去类对象里面找对应的方法实 现。如果消息的接收者是类对象，isa就指向元类，就会去元类里面找对应的方法实现。

## 消息的快速查找

查看 objc 源码，找到 objc_msgSend 实现（objc_msgSend 源码使用汇编实现，目的是能快速查找方法）

```armasm
//进入objc_msgSend流程
 ENTRY _objc_msgSend
//流程开始，无需frame
 UNWIND _objc_msgSend, NoFrame

//判断p0(消息接受者)是否存在，不存在则重新开始执行objc_msgSend
 cmp p0, #0   // nil check and tagged pointer check

//如果支持小对象类型。返回小对象或空
#if SUPPORT_TAGGED_POINTERS
//b是进行跳转，b.le是小于判断，也就是小于的时候LNilOrTagged
 b.le LNilOrTagged  //  (MSB tagged pointer looks negative)
#else
//等于，如果不支持小对象，就LReturnZero
 b.eq LReturnZero
#endif
//通过p13取isa
 ldr p13, [x0]  // p13 = isa
//通过isa取class并保存到p16寄存器中
 GetClassFromIsa_p16 p13, 1, x0 // p16 = class
//LGetIsaDone是一个入口
LGetIsaDone:
 // calls imp or objc_msgSend_uncached
//进入到缓存查找或者没有缓存查找方法的流程
 CacheLookup NORMAL, _objc_msgSend, __objc_msgSend_uncached

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
// nil check判空处理，直接退出
 b.eq LReturnZero  // nil check
 GetTaggedClass
 b LGetIsaDone
// SUPPORT_TAGGED_POINTERS
#endif

LReturnZero:
 // x0 is already zero
 mov x1, #0
 movi d0, #0
 movi d1, #0
 movi d2, #0
 movi d3, #0
 ret

 END_ENTRY _objc_msgSend
```

消息快速查找的流程：

1. 判断 receiver（消息的接受者）是否存在
2. receiver -> isa -> class
3. class内存平移 -> cache
4. cache -> buckets
5. 遍历buckets -> bucket(sel,imp)对比sel
6. 如果bucket(sel,imp)对比sel 相等 -->cacheHit-->调用imp
7. 如果cache里面没有找到对应的sel --> _objc_msgSend_uncached

## 消息慢速查找

在 cache 找不到方法就会进入到消息慢速查找流程。

_objc_msgSend_uncached 的实现

```armasm
STATIC_ENTRY __objc_msgSend_uncached
UNWIND __objc_msgSend_uncached, FrameWithNoSaves

// THIS IS NOT A CALLABLE C FUNCTION
// Out-of-band p15 is the class to search
 
MethodTableLookup
TailCallFunctionPointer x17

END_ENTRY __objc_msgSend_uncached

.macro MethodTableLookup
 
 SAVE_REGS MSGSEND

 // lookUpImpOrForward(obj, sel, cls, LOOKUP_INITIALIZE | LOOKUP_RESOLVER)
 // receiver and selector already in x0 and x1
 mov x2, x16
 mov x3, #3
 bl _lookUpImpOrForward

 // IMP in x0
 mov x17, x0

 RESTORE_REGS MSGSEND

.endmacro
```

最后调用 lookUpImpOrForward 方法。

```objc
IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)
{
    const IMP forward_imp = (IMP)_objc_msgForward_impcache;
    IMP imp = nil;
    Class curClass;

    runtimeLock.assertUnlocked();

    if (slowpath(!cls->isInitialized())) {
        behavior |= LOOKUP_NOCACHE;
    }

    runtimeLock.lock();

    checkIsKnownClass(cls);

    cls = realizeAndInitializeIfNeeded_locked(inst, cls, behavior & LOOKUP_INITIALIZE);
    // runtimeLock may have been dropped but is now locked again
    runtimeLock.assertLocked();
    curClass = cls;

    for (unsigned attempts = unreasonableClassCount();;) {
        if (curClass->cache.isConstantOptimizedCache(/* strict */true)) {
#if CONFIG_USE_PREOPT_CACHES
            imp = cache_getImp(curClass, sel);
            if (imp) goto done_unlock;
            curClass = curClass->cache.preoptFallbackClass();
#endif
        } else {
            // curClass method list.
            method_t *meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                imp = meth->imp(false);
                goto done;
            }

            if (slowpath((curClass = curClass->getSuperclass()) == nil)) {
                // No implementation found, and method resolver didn't help.
                // Use forwarding.
                imp = forward_imp;
                break;
            }
        }

        // Halt if there is a cycle in the superclass chain.
        if (slowpath(--attempts == 0)) {
            _objc_fatal("Memory corruption in class list.");
        }

        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (slowpath(imp == forward_imp)) {
            // Found a forward:: entry in a superclass.
            // Stop searching, but don't cache yet; call method
            // resolver for this class first.
            break;
        }
        if (fastpath(imp)) {
            // Found the method in a superclass. Cache it in this class.
            goto done;
        }
    }

    // No implementation found. Try method resolver once.

    if (slowpath(behavior & LOOKUP_RESOLVER)) {
        behavior ^= LOOKUP_RESOLVER;
        return resolveMethod_locked(inst, sel, cls, behavior);
    }

 done:
    if (fastpath((behavior & LOOKUP_NOCACHE) == 0)) {
#if CONFIG_USE_PREOPT_CACHES
        while (cls->cache.isConstantOptimizedCache(/* strict */true)) {
            cls = cls->cache.preoptFallbackClass();
        }
#endif
        log_and_fill_cache(cls, imp, sel, inst, curClass);
    }
 done_unlock:
    runtimeLock.unlock();
    if (slowpath((behavior & LOOKUP_NIL) && imp == forward_imp)) {
        return nil;
    }
    return imp;
}
```

消息慢速查找流程：

1. 先找当前类的 methodList
2. superClass 的 cache
3. superClass 的 methodList
4. 递归查找直到 superClas 为 nil

## 方法动态决议

在 lookUpImpOrFoward 方法里可以看到当找不到方法时会调用 resolveMethod_locked，这个方法实现了动态决议。

```objc
resolveMethod_locked(id inst, SEL sel, Class cls, int behavior)
{
    runtimeLock.assertLocked();
    ASSERT(cls->isRealized());

    runtimeLock.unlock();

    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        resolveInstanceMethod(inst, sel, cls);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        resolveClassMethod(inst, sel, cls);
        if (!lookUpImpOrNilTryCache(inst, sel, cls)) {
            resolveInstanceMethod(inst, sel, cls);
        }
    }

    // chances are that calling the resolver have populated the cache
    // so attempt using it
    return lookUpImpOrForwardTryCache(inst, sel, cls, behavior);
}
```

可以在类里面重写 resolveInstanceMethod（实例方法动态决议） 和 resolveClassMethod（类方法动态决议） 通过 Runtime 动态添加 sel 对应的方法实现 imp。

```objc
#import "Person.h"
#import <objc/runtime.h>

@implementation Person

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    IMP imp = class_getMethodImplementation(self.class, @selector(method));
    class_addMethod(self.class, sel, imp, "v@:");
    return YES;
}

- (void)method {
    NSLog(@"%s", __func__);
}

@end
```

## 消息快速转发

动态决议之后，进入到消息转发流程。

消息快速转发是把消息转发给另外一个对象。

```objc
-(id)forwardingTargetForSelector:(SEL)aSelector {
    return [NSObject new];
}
```

## 消息慢速转发

可以在NSObject分类实现这两个方法，防止程序因为找不到方法崩溃。

```objc
-(NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [NSMethodSignature signatureWithObjCTypes:"v@:"];
}

-(void)forwardInvocation:(NSInvocation *)anInvocation {
    
}
```
