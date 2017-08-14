---
layout: post
title:  "工厂方法和抽象工厂的区别"
date:   2017-08-13 18:43:00 +0800
categories: jekyll update
---
之前学习工厂方法和抽象工厂时，总觉得这两个区别不是很明显，网上很多帖子说的区别都是：工厂方法生产一种产品，抽象工厂生产多种产品。

如果仅仅是这样为什么要把两个单独拿出来说，完全可以直接说抽象工厂，然后工厂方法是抽象工厂的一个特殊的情况就可以了。

最近在重构自己模块的项目时，其中会用到对象创建，所以又学习了一下这两种模式而且又结合其他网友的看法，现在对这两种模式有了一个比之前要深入的理解。总结如下：

## 共同点
1. 都属于对象创建型。
2. 对象创建都延迟到子类执行。
3. 都有相似的类图，只是一个是创建单个对象，一个是创建多个对象。如下：

![factory_method](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/Factory_method.png)
![abstract_method](https://raw.githubusercontent.com/war3tiger/war3tiger.github.io/master/resources/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/Abstract_method.png)


## 不同点
1. **工厂方法**创建的是一个完整的产品，调用者可以直接使用。

	**抽象工厂**创建的是一系列产品零件，调用者需要自己拼接成一个完整的产品。
2. **工厂方法**重点是继承。**抽象工厂**重点是组合。
	
	在工厂方法(Creator)中创建的产品类(Product)一般是给自己使用的，不是给调用者使用的。在这里Creator本身就是Client，Creator类中除了有factoryMethod方法以外，还有使用Product的一系列方法。例如：
	
```
	@interface Creator : NSObject
	//@protected
	- (Product *)createProduct;
	//@private
	- (void)handleProduct;
	@end
	
	@implementation Creator
	- (Product *)createProduct
	{
		return [[Product alloc] init];
	}
	
	- (void)handleProduct
	{
		Product *product = [self createProduct];
		[product operation];
		......
	}
	@end
```

在上面的例子中`createProduct`其实是为`handleProduct`服务的，在这里基类提供了一个内部方法来处理`Product`，可是`Product`的不同的子类有不同的`operation`实现，这就需要`Creator`的子类重写`createProduct`方法，来创建不同的`Product`的子类。这里就是使用**工厂方法**。

在抽象工厂(AbstractFactory)中创建的是一系列产品类(ProductA, ProductB, ...)，是给调用者(Client)使用的。Client使用抽象工厂创建自己需要的产品，然后进行其他相关操作。例如：

```
@interface AbstractFactory : NSObject
- (ProductA *)createProductA;
- (ProductB *)createProductB;
@end

@implementation AbstractFactory
- (ProductA *)createProductA
{
	return [[ProductA alloc] init];
}

- (ProductB *)createProductB
{
	return [[ProductB alloc] init];
}
@end

@interface Client : NSObject
- (void)setFactoryManager:(AbstractFactory *)factory;
@end

@implementation Client
- (void)setFactoryManager:(AbstractFactory *)factory
{
	_factory = factory;
}

- (void)handleProducts
{
	ProductA *pa = [_factory createProductA];
	ProductB *pb = [_factory createProductB];
	
	[pa operation];
	[pb operation];
	
	......
}
@end
```

这里`Client`和`AbstractFactory`就是使用组合，这里使用的就是**抽象工厂**。

3. **工厂方法**不仅仅是对象的创建，它的更主要的是实现其他逻辑，对象的创建只是为那个更主要的逻辑服务的。

	**抽象工厂**其实就是一系列对象的创建。
	
## 相互转化
其实**工厂方法**和**抽象工厂**是在一定条件下可以相互转化的。如下：
### 工厂方法

```
@interface ObjectOPeration : NSObject

- (Product *)createProduct;
- (void)handleProduct;

@end

@implementation ObjectOPeration

- (Product *)createProduct
{
    return [[Product alloc] init];
}

- (void)handleProduct
{
    Product *product = [self createProduct];
    [product operation];
    ......
}
```

### 抽象工厂

```
@interface AbstractFactory : NSObject

- (Product *)createProduct;

@end

@implementation AbstractFactory

- (Product *)createProduct
{
    return [[Product alloc] init];
}

@end


@interface ObjectOPeration : NSObject

- (void)setFactory:(AbstractFactory *)factory;
- (void)handleProduct;

@end

@implementation ObjectOPeration

- (void)setFactory:(AbstractFactory *)factory
{
    _factory = factory;
}

- (void)handleProduct
{
    Product *product = [_factory createProduct];
    [product operation];
    ......
}

@end
```

## 总结
以上都是一些理论基础，在实际项目开发中要学会活学活用，不能一味的照搬，具体问题具体分析，只有在实践中才能对这两种模式有更深的体会，更深入的了解。


## 参考

<https://www.zhihu.com/question/20367734>

<https://stackoverflow.com/questions/5739611/differences-between-abstract-factory-pattern-and-factory-method>

<https://dzone.com/articles/factory-method-vs-abstract>