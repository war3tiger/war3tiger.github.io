---
layout: post
title:  "AutoreleasePool实现"
date:   2017-09-08 21:05:00 +0800
categories: jekyll update
---
>本文中所讲都是基于ARC下的。

在ARC下使用autorelease的场景一般是这样的：

**场景1**

```
@autoreleasepool {
	ObjectA *obj = [ObjectA objectA];
	[obj xxx];
}
```
经过编辑器处理后生成如下代码：

```
void *pool = objc_autoreleasePoolPush();
ObjectA *obj = [[ObjectA alloc] init];//这里是为了方便理解才这么写的
objc_autoreleaseReturnValue(obj);
objc_retainAutoreleasedReturnValue(obj);
[obj xxx];
objc_autoreleasePoolPop(pool);
```
如果嵌套使用autorelease时如下场景：

**场景2**

```
@autoreleasepool {
	ObjectA *obj = [ObjectA objectA];
	[obj xxx];
	@autoreleasepool {
		ObjectB *objB = [ObjectB objectB];
		[objB xxx];
	}
}
```
经过编译器处理如下：

```
void *pool = objc_autoreleasePoolPush();
ObjectA *obj = [[ObjectA alloc] init];//这里是为了方便理解才这么写的
objc_autoreleaseReturnValue(obj);
objc_retainAutoreleasedReturnValue(obj);
[obj xxx];

void *poolB = objc_autoreleasePoolPush();
ObjectB *objB = [[ObjectB alloc] init];//这里是为了方便理解才这么写的
objc_autoreleaseReturnValue(objB);
objc_retainAutoreleasedReturnValue(objB);
[objB xxx];
objc_autoreleasePoolPop(poolB);

objc_autoreleasePoolPop(pool);
```
现在看下`objc_autoreleasePoolPush`、`objc_autoreleasePoolPop`和`objc_autoreleaseReturnValue `的源码实现：

```
void *objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}

//这里做了简化处理 只是分析autoreleasepool的实现
id  objc_autoreleaseReturnValue(id obj)
{
	return AutoreleasePoolPage::autorelease(obj);
}

```
从以上三个函数的源码可见，autoreleasepool的实现是依赖于`AutoreleasePoolPage`。
下面就主要来看下`AutoreleasePoolPage `具体是做什么的。

## AutoreleasePoolPage
### 数据结构（简化版）
```
class AutoreleasePoolPage 
{
#   define EMPTY_POOL_PLACEHOLDER ((id*)1)
#   define POOL_BOUNDARY nil
    static pthread_key_t const key = AUTORELEASE_POOL_KEY;
    static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
    static size_t const SIZE = 4096;	//4k

    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
}
```
**EMPTY_POOL_PLACEHOLDER**这个说明如果当前没有page，那么将该变量设置为当前page。
### TLS(Thread Local Storage)：线程本地存储
`thread`这个变量说明**AutoreleasePoolPage**和**线程**是一一对应的，也就是说一个线程里面只有一个AutoreleasePoolPage，如果该线程中使用的autorelease对象很多超出了一个page的范围，那么`parent`和`child`就该上场了，下面会详细说明。

至于如何从当前线程中获得对应的page，这里就用到了`key`这个变量以及TLS技术。主要用到了以下函数：

```
pthread_key_create
pthread_get_specific
pthread_set_specific
```
详细使用说明可以自行查阅相关文档。

### 执行过程
这里会用图片的形式说明。

**场景1**

1. `AutoreleasePoolPage`初始内存图如下：

![autorelease_01](https://github.com/war3tiger/war3tiger.github.io/blob/master/resources/AutoreleasePoolPage/autorelease_01.png?raw=true)

2. `void *pool = objc_autoreleasePoolPush();`内存结构如下图：

![autorelease_02](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/AutoreleasePoolPage/autorelease_02.png)

3. `objc_autoreleaseReturnValue(obj);`内存结构如下图：

![autorelease_03](https://github.com/war3tiger/war3tiger.github.io/blob/master/resources/AutoreleasePoolPage/autorelease_03.png?raw=true)

执行完第3步之后，obj已经加到了`AutoreleasePoolPage`中了，如果还有其他autorelease对象也是通过这种方式加进去的，即：将当前对象放到next所指的区域，然后next++。

4. `objc_retainAutoreleasedReturnValue(obj);`就是简单的将对象retain。不过多解释。

5. `objc_autoreleasePoolPop(pool);`就是将next指针做自减操作，然后将对象释放，并将next所指的地址用`0xa3`填充，一直到`next == pool`为止。现在的内存结构图和第2步的是一样的了。

**场景2**

执行过程和**场景一**过程类似，只是多了一个嵌套。现在就分析下嵌套是怎么回事。看下图：

![autorelease_04](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/AutoreleasePoolPage/autorelease_04.png)

在上图中当嵌套的autorelease执行完之后，`next`指针就会指向`0xa1a2a3a4`的地址。

### parent child
当AutoreleasePoolPage满了以后，再添加autorelease对象时就需要重新创建一个AutoreleasePoolPage的对象了，并将新创建的parent指向现在的page，当前page的child指向新创建的page。然后pop时和上面的逻辑一样。

## 结束
该分析比较简单，只是把autorelease的大致流程介绍了下，希望可以通过该文章对autorelease有个比较浅显的认识和理解，至于更加深入的理解还是多看源码比较好。而且还可以从中学到一些好的算法和技术。比如：poolpage是如何管理parent和child中的poolpage对autorelease对象的添加和释放的。相信你看完之后会觉得解法很新奇。