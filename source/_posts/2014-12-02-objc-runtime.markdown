---
layout: post
title: "做题理解Object-C的runtime问题"
date: 2014-12-02 18:19:36 +0800
comments: true
categories: object-c
---

![object-c](/images/custom_images/objc-800.jpg)
<!-- more -->

####问题一：下面的代码输出什么？

	@implementation Son : Father
	- (id)init
	{
	    self = [super init];
	    if (self)
	    {
	        NSLog(@"%@", NSStringFromClass([self class]));
	        NSLog(@"%@", NSStringFromClass([super class]));
	    }
	    return self;
	}
	@end
上面的问题其实考察的就是对`self`和`super`的理解程度。首先明确`self是类的隐藏参数，指向当前调用方法的这个类的实例`。super本质上是一个编译器指示符(**查看runtime源码有self和superclass的定义，并没有找到super的定义，可见super非实有，编译器会对其进行解释**)，super和self指向的同一个消息接受者。上面的例子，[self class]和[super class]，接受消息的对象都是当前的Son *xxx这个对象。不同之处，是super告诉编译器，调用class去父类的方法。

说的直白一点就是，self告诉编译器查找方法先从本类方法列表找，再去父类找；使用super时，则从父类的方法列表中找，以此类推。

所以上题中[self class]输出就是：`Son`，这里需要我们把这个顺序捋一捋就清楚了，首先Son没有实现class，所以会去Father类中找，也没有，接着就到了NSObject，在NSObject中有这个方法，看下面的定义，返回的就是self的类别。

	- (Class)class {
	    return object_getClass(self);
	}

再看[super class],在Father中没有，再去NSObject类中，看了上面的class的定义，是不是发现了，还是返回self的类别，所以[super class]的输出也是：`Son`。

以上的理解参考了下面两篇文章,你可以去全面的了解一下。

1. [Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)
2. [刨根问底Objective－C Runtime（1）－ Self & Super](http://chun.tips/blog/2014/11/05/bao-gen-wen-di-objective%5Bnil%5Dc-runtime(1)%5Bnil%5D-self-and-super/)

####问题二：下面代码的运行结果是？

	@interface Sark : NSObject
	@end
	
	@implementation Sark
	@end
	
	int main(int argc, const char * argv[]) {
	    @autoreleasepool {
	        BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
	        BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
	
	        BOOL res3 = [(id)[Sark class] isKindOfClass:[Sark class]];
	        BOOL res4 = [(id)[Sark class] isMemberOfClass:[Sark class]];
	
	        NSLog(@"%d %d %d %d", res1, res2, res3, res4);
	    }
	    return 0;
	}

上面题目主要考察就是对`isKindOf`和`isMemberOf`方法的认识，以及对`class`和`meta class`的理解。下面就先来了解这几个方法和概念。

首先看一看class的实现，他返回类对象本身。

	+ (Class)class {
	    return self;
	}
所以上面的[NSObject class]返回的就是NSObject对象本身，[Sark class]返回的就是Sark对象了。

	- (BOOL)isKindOf:aClass
	{
	    Class cls;
	    for (cls = isa; cls; cls = cls->superclass) 
	        if (cls == (Class)aClass)
	            return YES;
	    return NO;
	}
从上面知道，isKindOf用于判断是否这个类或这个类的子类的实例。

	- (BOOL)isMemberOf:aClass
	{
	    return isa == (Class)aClass;
	}
isMemberOf只会判断是否这个类的实例。

对上面两个方法有了了解，还需要进一步了解`isa`和`class`以及`meta class`的关系。通过一张图片来认识一下。

**在Object-C的设计哲学中，一切都是对象，Class本身也是一个对象，而这个对象对应的类就是Meta Class。即Class结构体中的isa指向的就是它的Meta Class 。**

![image](http://106.186.113.24:8888/other/Class%26MetaClass.001.jpg)

通过上图我们知道，实例对象的`isa`是指向对象，而对象的isa则指向meta对象。了解了这几个变量的作用，我们就不难理解上面的答案应该是：

	 1 0 0 0

只有对象的实例使用`isMemberOf`才有可能返回`YES`，其他非实例对象返回的都是`NO`，而`isKindOf`也是针对实例方法而言的，特殊的例外就是`NSObject`对象了，因为`NSObject对象的isa指向的meta class的superclass就是NSObject对象`。


本文总结自@Chun发表于Chun Tips的[刨根问底Objective－C Runtime（2）－ Object & Class & Meta Class][1]


[1]: http://chun.tips/blog/2014/11/05/bao-gen-wen-di-objective%5Bnil%5Dc-runtime-(2)%5Bnil%5D-object-and-class-and-meta-class/    "刨根问底Objective－C Runtime（2）－ Object & Class & Meta Class"

####问题三、下面代码会？Compile Error / Runtime Crash / NSLog…?

	@interface NSObject (Sark)
	+ (void)foo;
	@end
	
	@implementation NSObject (Sark)
	
	- (void)foo
	{
	    NSLog(@"IMP: -[NSObject(Sark) foo]");
	}
	
	@end
	
	int main(int argc, const char * argv[]) {
	    @autoreleasepool {
	        [NSObject foo];
	        [[NSObject new] foo];
	    }
	    return 0;
	}

上面这个题目玩的也是奇技淫巧，要特别注意是NSObject的扩展，而不是别的类的扩展，一定要深刻了解前面的图示。那么这一道题要考察的就是对方法传递的机制的熟悉程度。

上面的两个方法，在编译的时候会转换成下面的方法调用：

	 ((void (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("foo"));
	 ((void (*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("new")), sel_registerName("foo"));
	 
具体的分析过程，请看这里[刨根问底Objective－C Runtime（3）－ 消息 和 Category][2] 。这里就只记录最终的一个分析过程。

**上面的代码在objc runtime加载完后，NSObject的Sark Category被加载，由于+(void)foo并没有实现，所以编译器进行静态检查时，会出现警告信息，提示我们方法没有实现，但在代码编译中，它会被注释掉。**

**实际上被加入到Class的method list的方法是- (void)foo，它是一个实例方法，所以加入到当前类对象NSObject的方法列表中，而不是NSObject Meta Class的方法列表中。**

当执行`[NSObject foo]`时，其过程是

1. objc_msgSend的第一个参数“(id)objc_getClass("NSObject")”，获得NSObject Class的对象；
2. 类方法在Meta Class的方法列表中找，我们在load Category方法时加入的是- (void)foo实例方法，所以并不在NSObject Meta Class的方法列表中；
3. 继续往super class中找，结合前面可知，NSObject Meta Class的super class就是NSObject本身，所以，这时候找到-(void)foo方法；
4. 所以这里输出结果：`IMP: -[NSObject(Sark) foo]`.

当执行`[[NSObject new] foo]`，其过程如下

1. [NSObject new]生成一个NSObject对象；
2. 直接在该对象的类(NSObject)的方法列表里找；
3. 可以找到，所以输出结果：`IMP: -[NSObject(Sark) foo]`。


[2]:http://chun.tips/blog/2014/11/06/bao-gen-wen-di-objective%5Bnil%5Dc-runtime(3)%5Bnil%5D-xiao-xi-he-category/



####问题四：下面代码会? Compile Error / Runtime Crash / NSLog…?

	@interface Sark : NSObject
	@property (nonatomic, copy) NSString *name;
	@end
	
	@implementation Sark
	
	- (void)speak
	{
	    NSLog(@"my name is %@", self.name);
	}
	
	@end
	
	@interface Test : NSObject
	@end
	
	@implementation Test
	
	- (instancetype)init
	{
	    self = [super init];
	    if (self) {
	        id cls = [Sark class];
	        void *obj = &cls;
	        [(__bridge id)obj speak];
	    }
	    return self;
	}
	
	@end
	
	int main(int argc, const char * argv[]) {
	    @autoreleasepool {
	        [[Test alloc] init];
	    }
	    return 0;
	}
具体的解答过程参考[刨根问底Objective－C Runtime（4）－ 成员变量与属性](http://chun.tips/blog/2014/11/08/bao-gen-wen-di-objective%5Bnil%5Dc-runtime\(4\)%5Bnil%5D-cheng-yuan-bian-liang-yu-shu-xing/)

上面例子的关键在以下代码：

	id cls = [Sark class];
	void *obj = &cls;
	[(__bridge id)obj speak];
通过前面可知，[Sark class]即指向Sark对象本身，转换成id对象，而id实际上就是struct objc_object类型，obj则是指向cls的指针（按chun的说法是，obj相当于一个Sark的实例对象，但又不同于[Sark new]产生的对象），那么在obj所指向的Sark Class的方法列表中可以找到speak方法。

按我的理解就是，obj的目的就是为了使其isa指向Sark Class，因为按照前面的分析，cls的isa实际上是指向Sark的Meta Class，采用[cls speak]在Meta Class的方法列表中是找不到方法的，会报错，[(id)obj speak]则是在Sark Class方法列表中查找到了speak方法。
