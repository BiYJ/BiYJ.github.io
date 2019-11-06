---
title: iOS 类簇
categories: iOS原理
---

## 一、类簇

类簇是 Foundation 框架广泛使用的设计模式。<font color=#cc0000>类簇在公共抽象超类下对多个私有的具体子类进行分组</font>。以这种方式对类进行分组<font color=#cc0000>简化了面向对象框架的公共可见体系结构</font>，而不会降低其功能丰富度。类簇是基于抽象工厂设计模式的。


## 二、抽象工厂

抽象工厂模式是指当有多个抽象角色时，使用的一种工厂模式。抽象工厂模式可以向客户端提供一个接口，使客户端在不必指定产品的具体的情况下，创建多个产品族中的产品对象。很多人会混淆抽象工厂模式和工厂模式。实际上，两种的差别还是比较明显的，如下表。

<center>

|抽象工厂模式|工厂模式|
|:----:|:------:|
|通过对象组合创建抽象产品|通过类继承创建抽象产品|
|创建多系列产品|创建一种产品|
|必须修改父类的接口才能支持新的产品|子类化创建者并重载工厂方法以创建新产品|

</center>

点击[查看子类的名称](https://gist.github.com/Catfish-Man/bc4a9987d4d7219043afdf8ee536beb2)。


## 三、NSArray

《Effective Objective-C 2.0》中有一段话：

>In the case of NSArray, when an instance is allocated, it’s an instance of another class that’s allocated (during a call to alloc), known as a placeholder array. This placeholder array is then converted to an instance of another class, which is a concrete subclass of NSArray.
>
>在使用了 NSArray 的 alloc 方法来获取实例时，该方法首先会分类一个属于某类的实例，此实例充当“占位数组”。该数组稍后会转为另一个类的实例，而那个类则是 NSArray 的实体子类。

不管创建的是可变还是不可变的数组，在 alloc 之后得到的类都是 `__NSPlaceholderArray`。而当我们 init 一个不可变的空数组之后，得到的是 `__NSArray0`；如果有且只有一个元素，那就是 `__NSSingleObjectArrayI`；有多个元素的，叫做 `__NSArrayI`；init 出来一个可变数组的话，都是 `__NSArrayM`。

注意：当使用 + array.. 方法创建数组对象时，只有一个元素也是 `__NSArrayI`。

这里 \_\_NSSingleObjectArrayI，需要说明它的用意。


## 四、__NSSingleObjectArrayI

作为对比，__NSArrayI 必须要实现

* - count
* - objectAtIndex:

两个方法，但是我们可以非常显而易见的看出来，当数组只有一个数字的时候，是完全不需要这两个方法的。
再深入一点的说明一下，\_\_NSSingleObjectArrayI 是不需要去记录字符串长度的。它会比 \_\_NSArrayI <font color=#cc0000>少 8 个字节的长度</font>。苹果可能是为了优化性能考虑，从而在 iOS8 之后推出这个新的子类。

另外需要说明的是，实际上，\_\_NSArrayM 本身只有 7 个方法，分别是：

* - count
* - objectAtIndex:
* - insertObject:atIndex:
* - removeObjectAtIndex:
* - addObject:
* - removeLastObject
* - replaceObjectAtIndex:withObject:

所有其它高等级的抽象建立在它们的基础之上。例如 `- removeAllObjects` 方法简单地往回迭代，一个个地调用 `- removeObjectAtIndex:`。


## 五、NSDictionary

NSDictionary 与 NSArray 类似，不管创建的是可变还是不可变的字典，在 alloc 之后得到的类都是 `__NSPlaceholderDictionary`。而 init 一个不可变的空数组之后，得到的是 `__NSDictionary0`；如果有且只有一个元素，那就是 `__NSSingleEntryDictionaryI`；有多个元素的，叫做 `__NSDictionaryI`；init 出来一个可变数组的话，都是 `__NSDictionaryM`。还有一个子类，`__NSFrozenDictionaryM`。

```
NSDictionary * dict = [[NSDictionary alloc] initWithObjectsAndKeys:@"Tom", @"name", nil];
NSDictionary * copyDict = dict.copy;
NSLog(@"copyDict       : %@   %p   %@", copyDict, copyDict, [copyDict class]);

copyMDict       : {
    name = Tom;
}   0x60000363cb60   __NSFrozenDictionaryM
```

这个子类没什么特殊的作用，它仍然会被视为不可变字典。也就是说，对它进行改变的操作，会导致程序崩溃。崩溃信息如下：

```
-[__NSFrozenDictionaryM setObject:forKey:]: unrecognized selector sent to instance 0x600000490860
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[__NSFrozenDictionaryM setObject:forKey:]: unrecognized selector sent to instance 0x600000490860'
```

其实在在 NSArray 中也有个对应的 `__NSFrozenArrayM`。


## 六、NSSet

NSSet 的子类不过是 NSDictionary 换了个名字而已，不做细讲。

这里说明一下，`__NSSingleObjectSetI` 不需要打扰实际的哈希表，因为只有一个对象需要担心。类似的方法 containsObject: 不需要遍历任何东西或查找任何东西，它可以简单地将参数与 set/array/dictionary 表示的单个对象进行比较。

## 七、NSString

当测试创建 NSString 对象的时候，通过创建 NSString 不同的对象，并利用 object_getClassName 方法打印对象。

```
NSString * str1 = @"1234567890";
NSLog(@"str1: %@", [str1 class]);  // str1: __NSCFConstantString
    
NSString * str2 = @"123456789";
NSLog(@"str2: %@", [str2 class]);  //str2: __NSCFConstantString
    
NSString * str3 = [NSString stringWithFormat:@"123456789"];
NSLog(@"str3: %@", [str3 class]);  // str3: NSTaggedPointerString
    
NSString * str4 = [NSString stringWithFormat:@"1234567890"];
NSLog(@"str4: %@", [str4 class]);  // str4: __NSCFString
```

这里出现了三个子类：

* __NSCFConstantString
* __NSCFString
* NSTaggedPointerString

1. __NSCFConstantString

	它是一个字符串常量。它的引用计数非常大，是 4294967295。它的意思是：这个属性，怎么都不会被释放。相同的对象，内存地址是相同的，可以直接使用 == 方法（但是，这个对象的指针的地址依然不同，还是两个不同的对象）。它在编译时就决定的，不能在运行时创建。

	更详细的可以查看[这篇博客](https://blog.cnbluebox.com/blog/2014/04/16/nsstringte-xing-fen-xi-xue-xi/)。

2. __NSCFString

	这个就是可变的 NSString 所属的子类。

3. NSTaggedPointerString

	要从 iPhone5s 开始说起，iPhone5s 开始采用了 64 位处理器。在 32 位时代，一个指针大小是 32 位（4字节），而在 64 位时代翻倍，一个指针的大小变成了 64 位（8字节）。这样子，在处理某些小一点，短一点的 NSString、NSNumber、NSDate 对象的时候，会显得过于浪费效率。这个时候，苹果推出了 Tagged Pointer 技术。

	苹果将一个对象的指针拆分成了两部分，一部分直接保存数据，另一部分作为特殊标记（tag），表示这个是一个特别的指针。这样呢，就会将节省很多的时间，因为它不再需要正常创建对象的申请和创建空间，处理引用计数，以及直接读取（在 objc_msgSend 当中，Tagged Pointer 会被识别出来，直接从指针中读取）。

	苹果之前说过：
	
	> 使用 Tagged Pointer 技术之后，在内存上读取的速度快了 3 倍，创建时的速度比以前快了 106 倍。

	当然，这么做其实也是会有问题的，因为它并不是一个真正的对象，当你想要想其他普通的对象一样获取指针的时候，编译器直接就会报错（因为它也是在编译时创建的，而且压根没有 isa 指针）。

	编译器会告诉你正确的方法：改为使用 `object_getClass()`。


## 八、总结

类簇的优点：

1. 可以将抽象基类背后的复杂细节隐藏起来
2. 程序员不会需要记住各种创建对象的具体类实现，简化了开发成本，提高了开发效率
3. 便于进行封装和组件化
4. 减少了 if-else 这样缺乏扩展性的代码
5. 增加新功能支持不影响其他代码

类簇的缺点：

1. 已有的类簇非常不好扩展

我们了解类簇的好处：

* 出现 bug 时，可以通过崩溃报告中的类簇关键字，快速定位 bug 位置。
* 在实现一些固定且并不需要经常修改的事物时，可以高效的选择类簇去实现。

	举个例子：针对不同版本，不同机型往往需要不同的设置，这时可以选择使用类簇；

* app 的设置页面这种并不需要经常修改的页面，可以使用类簇去创建大量重复的布局代码。


## 九、文章

[伯陽](https://www.jianshu.com/u/471ab65a3123) - [iOS中类簇的使用](https://www.jianshu.com/p/68956f300fc2)
[Friday Q&A 2015-07-31: Tagged Pointer Strings](https://www.mikeash.com/pyblog/friday-qa-2015-07-31-tagged-pointer-strings.html)
[NSString特性分析学习](https://blog.cnbluebox.com/blog/2014/04/16/nsstringte-xing-fen-xi-xue-xi/)