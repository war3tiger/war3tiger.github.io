---
layout: post
title:  "MRC下block使用备忘"
date:   2018-09-07 18:39:00 +0800
categories: jekyll update
---

# 场景

```
__block typeof(self) weakSelf = self;
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
	typeof(self) *strongSelf = weakSelf;
	if (strongSelf) {
		[strongSelf doSomething];
	}
});
```
这是在MRC下使用block时很常用的一个解决循环引用的场景。
可是这么做是不对的！

**分析：**

这时如果在执行block之前，这个对象释放了，那么`strongSelf`已经是野指针了，对野指针进行操作必然会导致crash。

**解决方法如下：**

使用`malloc_zone_from_ptr(strongSelf)`判断该指针指是否已经成为了野指针。

代码如下：

```
__block typeof(self) weakSelf = self;
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(3.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
	typeof(self) *strongSelf = weakSelf;
	if (malloc_zone_from_ptr(strongSelf)) {
		[strongSelf doSomething]
	} else {
		NSLog(@"error: strongSelf is dealloc!!!");
	}
});
```

**备注：**
使用`malloc_zone_from_ptr`时需要包含头文件：`#import <malloc/malloc.h>`。否则编译失败。