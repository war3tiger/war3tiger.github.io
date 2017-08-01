---
layout: post
title:  "Block之捕获OC类型的对象"
date:   2017-08-01 17:12:00 +0800
categories: jekyll update
---
> 该文章的分析都是基于ARC下的。

Block分析是基于以下源码：

```
int main(int argc, const char * argv[]) {
    
    NSRunLoop *currentLoop = [NSRunLoop currentRunLoop];
    
    void (^g_block)(void);
    {
        MyObject *obj = [[MyObject alloc] init];
        g_block = ^{
            [obj i_method1];
        };
    }
    g_block();
    g_block = nil;
    
    [currentLoop run];
    return 0;
}
```
通过`clang -rewrite-objc main.mm`可以将OC代码生成可读的C源码。如下，这里只贴出主要部分的代码而且做了简化操作，详细信息自己可以生成看下。

```
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  MyObject *obj;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, MyObject *_obj, int flags=0) : obj(_obj) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) 
{
 	MyObject *obj = __cself->obj; // bound by copy
 	[obj i_method1];
}
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) 
{
	_Block_object_assign((void*)&dst->obj, (void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static void __main_block_dispose_0(struct __main_block_impl_0*src) 
{
	_Block_object_dispose((void*)src->obj, 3/*BLOCK_FIELD_IS_OBJECT*/);
}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 
	0, 
	sizeof(struct __main_block_impl_0), 
	__main_block_copy_0, 
	__main_block_dispose_0
};
int main(int argc, const char * argv[]) {

    void (*g_block)(void);
    {
        MyObject *obj = [[MyObject alloc] init];
        g_block = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, obj, 570425344));
    }
    ((void (*)(__block_impl *))((__block_impl *)g_block)->FuncPtr)((__block_impl *)g_block);
    g_block = __null;

    return 0;
}
```
这里可以看出block上是个结构体，而且还生成了一个静态全局对象：`__main_block_desc_0_DATA `，该对象提供了一个copy和dispose方法，这两个方法的说明请往下看。

下面重点分析main函数中的代码。

## __main_block_impl_0 构造函数

当函数执行到`g_block = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, obj, 570425344));`时，开始执行`__main_block_impl_0 `的构造方法：

```
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, MyObject *_obj, int flags=0) : obj(_obj) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
```
注意这里会执行一段**obj(_obj)**函数，这里其实就等于`obj=_obj`，在执行这段代码的时候强大的编译器来了，会将赋值操作分解成下面两个函数:

![block_01_01](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/block/block_01_01.png)

这时这个MyObject对象相当于被block强引用一次。

然后继续执行该构造方法中的代码，这时会执行`impl.isa = &_NSConcreteStackBlock;`，这表示该block是在栈上创建的。
`impl.Flags = flags;(BLOCK_HAS_COPY_DISPOSE| BLOCK_USE_STRET)`这里的flags是如下类型：

```
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler
};
```
执行完该构造函数后`g_block`已经创建好了而且是在栈区，这时强大的编译器又出现了，会在执行完构造函数后面又会生成block copy的代码：`g_blcok = _Block_copy(g_block);`下面看下**_Block_copy**的执行逻辑。

## _Block_copy
```
void *_Block_copy(const void *arg) {
    return _Block_copy_internal(arg, WANTS_ONE);
}

static void *_Block_copy_internal(const void *arg, const int flags) {
    struct Block_layout *aBlock;
    const bool wantsOne = (WANTS_ONE & flags) == WANTS_ONE;
    
    if (!arg) return NULL;
    
    aBlock = (struct Block_layout *)arg;
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }
    struct Block_layout *result = malloc(aBlock->descriptor->size);
    if (!result) return (void *)0;
    memmove(result, aBlock, aBlock->descriptor->size);
    result->flags &= ~(BLOCK_REFCOUNT_MASK);
    result->flags |= BLOCK_NEEDS_FREE | 1;
    result->isa = _NSConcreteMallocBlock;
    if (result->flags & BLOCK_HAS_COPY_DISPOSE) {
        (*aBlock->descriptor->copy)(result, aBlock);
    }
    return result;
}
```
这里最后一个函数`(*aBlock->descriptor->copy)(result, aBlock);`就会用到`__main_block_desc_0_DATA `中的copy方法了。
执行完`g_blcok = _Block_copy(g_block);`后，g_block已经从栈上移到了堆上了。而且g_block捕获的oc对象也会被再次retain下。至于g_block捕获的oc对象是如何被retain的，请看下面的`__main_block_desc_0_DATA `中的**copy**方法。

## __main_block_desc_0_DATA
在上面生成的C源码中`__main_block_desc_0_DATA `的copy和dispose方法分别是：

```
_Block_object_assign((void*)&dst->obj, (void*)src->obj, 3);
_Block_object_dispose((void*)src->obj, 3);
```
这两个函数最后面的那个参数是如下类型：

```
enum {
    BLOCK_FIELD_IS_OBJECT   =  3,  // id, NSObject, __attribute__((NSObject)), block, ...
    BLOCK_FIELD_IS_BLOCK    =  7,  // a block variable
    BLOCK_FIELD_IS_BYREF    =  8,  // the on stack structure holding the __block variable
    BLOCK_FIELD_IS_WEAK     = 16,  // declared __weak, only used in byref copy helpers
    BLOCK_BYREF_CALLER      = 128, // called from __block (byref) copy/dispose support routines.
};

```
##### copy方法
下面看下`_Block_object_assign `的源码实现：

```
void _Block_object_assign(void *destAddr, const void *object, const int flags) {
    
    if ((flags & BLOCK_BYREF_CALLER) == BLOCK_BYREF_CALLER) {
        if ((flags & BLOCK_FIELD_IS_WEAK) == BLOCK_FIELD_IS_WEAK) {
            _Block_assign_weak(object, destAddr);
        }
        else {
            _Block_assign((void *)object, destAddr);
        }
    }
    else if ((flags & BLOCK_FIELD_IS_BYREF) == BLOCK_FIELD_IS_BYREF)  {
        _Block_byref_assign_copy(destAddr, object, flags);
    }
    else if ((flags & BLOCK_FIELD_IS_BLOCK) == BLOCK_FIELD_IS_BLOCK) {
        _Block_assign(_Block_copy_internal(object, flags), destAddr);
    }
    else if ((flags & BLOCK_FIELD_IS_OBJECT) == BLOCK_FIELD_IS_OBJECT) {
        objc_storeStrong(destAddr, object);
    }
}
```
##### release方法
`g_blcok = _Block_copy(g_block);`执行完这个函数后，就会调用`g_block();`方法执行block中的函数，最后调用`g_block = nil;`方法销毁block，当执行到该段代码时强大的编译器又出现了，它会插入如下代码来销毁block。如下图：

![block_01_02](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/block/block_01_02.png)

下面看下_Block_release源码实现：

```
void _Block_release(void *arg) {
    struct Block_layout *aBlock = (struct Block_layout *)arg;
    int32_t newCount;
    if (!aBlock) return;
    newCount = latching_decr_int(&aBlock->flags) & BLOCK_REFCOUNT_MASK;
    if (newCount > 0) return;
    // Hit zero
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        if (aBlock->flags & BLOCK_HAS_COPY_DISPOSE)(*aBlock->descriptor->dispose)(aBlock);
        _Block_deallocator(aBlock);
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        ;
    }
    else {
        printf("Block_release called upon a stack Block: %p, ignored\n", (void *)aBlock);
    }
}
```
这时就会调用`__main_block_desc_0_DATA `中的`dispose`方法。最终会执行到`_Block_object_dispose((void*)src->obj, 3);`该方法。源码如下：

```
void _Block_object_dispose(const void *object, const int flags) {
    if (flags & BLOCK_FIELD_IS_BYREF)  {
        _Block_byref_release(object);
    }
    else if ((flags & (BLOCK_FIELD_IS_BLOCK|BLOCK_BYREF_CALLER)) == BLOCK_FIELD_IS_BLOCK) {
        _Block_destroy(object);
    }
    else if ((flags & (BLOCK_FIELD_IS_WEAK|BLOCK_FIELD_IS_BLOCK|BLOCK_BYREF_CALLER)) == BLOCK_FIELD_IS_OBJECT) {
        objc_storeStrong(&object, nil);
    }
}

```
执行到`objc_storeStrong(&object, nil);`时，这里就会将block捕获的MyObject对象解引用下。
至此g_block解引用了它捕获到的全部对象，g_block对象也被销毁掉了。

### 备注：
1. `latching_incr_int`表示将block的引用计数加1。
2. `latching_decr_int `表示将block的引用计数减1，并返回retainCount值。

# 参考
1. [BlocksRuntime](http://llvm.org/svn/llvm-project/compiler-rt/trunk/lib/BlocksRuntime/runtime.c)
2. [Block_private.h](http://llvm.org/svn/llvm-project/compiler-rt/trunk/lib/BlocksRuntime/Block_private.h)
3. OC Runtime源码