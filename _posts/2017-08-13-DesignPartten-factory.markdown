---
layout: post
title:  "工厂方法和抽象工厂的区别"
date:   2017-08-13 18:43:00 +0800
categories: jekyll update
---
之前学习工厂方法和抽象工厂时，总觉得这两个区别不是很明显，网上很多帖子说的区别都是：工厂方法生产一种产品，抽象工厂生产多种产品。

如果仅仅是这样为什么要把两个单独拿出来说，完全可以直接说抽象工厂，然后工厂方法是抽象工厂的一个特殊的情况就可以了。

最近在重构自己模块的项目，其中会用到对象创建，所以又学习了一下这两种模式而且又结合网上其他网友的看法，现在对这两种模式有了一个比之前要深入的理解。总结如下：

## 共同点
1. 都属于对象创建型。
2. 对象创建都延迟到子类。
3. 都有相似的类图，只是一个是创建单个对象，一个是创建多个对象。如下：


## 参考

<https://www.zhihu.com/question/20367734>

<https://stackoverflow.com/questions/5739611/differences-between-abstract-factory-pattern-and-factory-method>

<https://dzone.com/articles/factory-method-vs-abstract>