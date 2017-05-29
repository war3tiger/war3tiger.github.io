---
layout: post
title:  "arc下使用weak clang失败"
date:   2017-05-29 17:00:00 +0800
categories: jekyll update
---

在arc模式下，使用weak修饰符时clang -rewrite-objc xxx.m时会提示编译错误：

```
cannot create __weak reference because the current deployment target does
      not support weak references
    __attribute__((objc_ownership(weak))) NSObject *obj2 = obj;
```

解决方法：

```
clang -rewrite-objc -fobjc-arc -stdlib=libc++ -mmacosx-version-min=10.7 -fobjc-runtime=macosx-10.7 -Wno-deprecated-declarations main.m
```