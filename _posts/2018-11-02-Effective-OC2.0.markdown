---
layout: post
title:  "Effective OC2.0总结"
date:   2018-09-07 18:39:00 +0800
categories: jekyll update
---
[1. ARC下内存管理及编译优化](#catalog-1)

[2. 多用派发队列，少用同步锁](#catalog-2)

[3. GCD和NSOperation区别](#catalog-3)


# <a name="catalog-1"></a>1. ARC下内存管理及编译优化
## 内存管理
1. 若方法名以下列词语开头，则其返回的对象归调用者所有：**alloc**、**new**、**copy**、**mutableCopy**。及编译器不会自动执行`autorelease`操作。
2. 若方法名不以上述词语开头，则表示返回的对象不归调用者所有。这时返回的对象会自动调用`autorelease`操作。

例如：

```
@interface MyObject : NSObject
+ (instancetype)allocObject;
+ (instancetype)object;
@end

@implementation MyObject
- (void)dealloc {
    printf("MyObject dealloc:%p\n", self);
}

+ (instancetype)allocObject {
    return [[MyObject alloc] init];
}

+ (instancetype)object {
    return [[MyObject alloc] init];
}
@end


void test(void) {
    printf("start\n");
    __weak MyObject *obj1 = nil;
    __weak MyObject *obj2 = nil;
    {
        MyObject *obj = [MyObject allocObject];
        obj1 = obj;
        obj2 = [MyObject object];
        printf("<obj1:%p, obj2:%p>\n", obj1, obj2);
    }
    printf("**********\n");
    printf("obj1:%p, obj2:%p\n", obj1, obj2);
    printf("end\n");
}

```
结果：

```
start
<obj1:0x10fe294e0, obj2:0x10fe291e0>
MyObject dealloc:0x10fe294e0
**********
obj1:0x0, obj2:0x10fe291e0
end
MyObject dealloc:0x10fe291e0
```
## 编译优化
ARC在编译期间除了会自动插入`retain`、`release`、`autorelease`操作外，在插入时还做了优化。比如：在编译期间，ARC会把能够互相抵消的`retain`、`release`、`autorelease`操作简化。例如：

```
_myPerson = [EOCPerson personWithName:@"Bob Smith"];
```
调用该方法会返回新的**EOCPerson**对象，而此方法返回对象之前ARC调用了`autorelease`方法。由于`_myPerson`是个强引用，所以编译器还执行了一次`retain`操作。因此上面代码与如下MRC下代码等效：

```
MRC
tmp = [EOCPerson personWithName:@"Bob Smith"];
_myPerson = [tmp retain];
```
此时可以看出`autorelease`与`retain`都是多余的。这是ARC会优化成如下代码：

```
+ (EOCPerson *)personWithName:(NSString *)name {
	EOCPerson *person = [[EOCPerson alloc] init];
	person.name = name;
	objc_autoreleaseReturnValue(person);
}
EOCPerson *tmp = [EOCPerson personWithName:@"Bob Smith"];
_myPerson = objc_retainAutoreleasedReturnValue(tmp);
```
其中`objc_autoreleaseReturnValue `和`objc_retainAutoreleasedReturnValue `函数的执行效率要比调用`autorelease`、`retain`更快。伪代码如下：

```
id objc_autoreleaseReturnValue(id object) {
	if (caller will retain object) {
		setflag(object);
		return object;
	} else {
		return [object autorelease];
	}
}

id objc_retainAutoreleasedReturnValue(id object) {
	if (get_flag(object)) {
		clear_flag(object);
		return object;
	} else {
		return [object retain];
	}
}
```

# <a name="catalog-2"></a>2. 多用派发队列，少用同步锁
在开发中，如果有多个线程执行同一份代码，这时一般都会使用锁来实现同步机制。比如**@synchronized**、**NSLock**等，这种方式的效率较低。替代方案是使用GCD，它能以更简单、更高效的形式为代码加锁。

比如对属性的读写，如果是多线程同时对某一对象属性进行读写操作，那么使用加锁的代码如下：

```
- (NSString *)someString {
	@synchronized(self) {
		return _someString;
	}
}

- (void)setSomeString:(NSString *)someString {
    @synchronized (self) {
        _someString = someString;
    }
}
```
那么可以使用串行队列实现同步，优化为：

```
queue = dispatch_queue_create("com.effective.syncQueue", NULL);

- (NSString *)someString {
    __block NSString *str = nil;
    dispatch_sync(queue, ^{
        str = _someString;
    });
    return str;
}

- (void)setSomeString:(NSString *)someString {
    dispatch_sync(queue, ^{
        _someString = someString;
    });
}
```

这么做是吧读写都安排在队列里执行，由于是串行队列，所以对该属性的访问都是同步了。

**注意：**如果度操作可以并发执行，而写操作不能并发执行而且是异步执行。可以改为如下代码：

```
queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

- (NSString *)someString {
    __block NSString *str = nil;
    dispatch_sync(queue, ^{
        str = _someString;
    });
    return str;
}

- (void)setSomeString:(NSString *)someString {
    dispatch_barrier_async(queue, ^{
        _someString = someString;
    });
}
```

# <a name="catalog-3"></a>3. GCD和NSOperation区别
**1. 取消某个操作**

NSOperation可以取消未执行的任务，不能取消正在执行中的任务。GCD不管是未执行还是正在执行的都无法取消，只能通过应用程序自己来实现取消。

**2. 指定操作间的依赖关系**

**3. 通过键值观测机制监控NSOperation对象属性**

**4. 指定操作优先级**

**5. 重用NSOperation对象**