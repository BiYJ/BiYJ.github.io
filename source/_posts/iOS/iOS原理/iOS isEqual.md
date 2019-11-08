---
title: isEqual
categories: iOS原理
---

## 如何重写 hash 方法

一个合理的 hash 方法要尽量让 hash 表中的元素均匀分布，来保证较高的查询性能。

如果两个对象可以被视为同一个对象，那么他们的 hash 值要一样。

mattt 在文章 [Equality](https://nshipster.com/equality/) 中给出了一个普遍的算法:

```
- (NSUInteger)hash 
{
	// 假设对象有三个属性，那么对这三个属性分别算出 hash 值，然后进行异或运算
    return [self.property1 hash] ^ [self.property2 hash] ^ [self.property3 hash];
}
```

Instagram 在开源 IGListKit 的同时，鼓励这么写 hash 方法：

```
- (NSUInteger)hash
{
	NSUInteger subhashes[] = {[self.property1 hash], [self.property2 hash], [self.property3 hash]};
	
	NSUInteger result = subhashes[0];
	
	for (int ii = 1; ii < 3; ++ii) {
		unsigned long long base = (((unsigned long long)result) << 32 | subhashes[ii]);
		base = (~base) + (base << 18);
		base ^= (base >> 31);
		base *=  21;
		base ^= (base >> 11);
		base += (base << 6);
		base ^= (base >> 22);
		result = base;
  }
  return result;
}
```

示例：

```
@interface Person : NSObject
 
@property (nonatomic, copy) NSString * name;
@property (nonatomic, strong) NSDate * birthday;
 
@end


- (BOOL)isEqual:(id)object
{
    if (self == object) {
        return YES;
    }
 
    if (![object isKindOfClass:[Person class]]) {
        return NO;
    }
 
    return [self isEqualToPerson:(Person *)object];
}
 
- (BOOL)isEqualToPerson:(Person *)person {
    if (!person) {
        return NO;
    }
 
    BOOL haveEqualNames = (!self.name && !person.name) || [self.name isEqualToString:person.name];
    BOOL haveEqualBirthdays = (!self.birthday && !person.birthday) || [self.birthday isEqualToDate:person.birthday];
 
    return haveEqualNames && haveEqualBirthdays;
}
```

1. == 运算符判断是否是同一对象, 因为同一对象必然完全相同
2. 判断是否是同一类型, 这样不仅可以提高判等的效率, 还可以避免隐式类型转换带来的潜在风险
3. 通过封装的 isEqualToPerson: 方法, 提高代码复用性
4. 判断 person 是否是 nil，做参数有效性检查
5. 对各个属性分别使用默认判等方法进行判断
6. 返回所有属性判等的与结果


## 重写 isEqual

> 如何写一个合理高效的判等方法？

1. 首先对内存地址进行判断，地址相等 return YES;
2. 进行判空处理，self == nil || object == nil ，return NO;
3. 类型判断，![object isKindOfClass:[self class]] , return NO;
4. 对对象的其他属性进行判断

根据这四个步骤，我们可以发现，我们都是先判断时间开销最少的属性。所以对于第 4 个步骤，如果对象有很多属性，我们也要依照这个原则来！

比如 [self.array isEqual:other.array] && self.intVal == other.intVal 这种写法是不合理的，因为 array 的判等会去遍历元素，时间开销大。如果 intVal 不相等的话就可以直接 return NO了，没必要进行数组的判等。应该这么写：self.intVal == other.intVal && [self.array isEqual:other.array]

```
- (BOOL)isEqual:(PersonModel *)object
{
	if (self == object) {
		return YES;
	}
	else if (self == nil || object == nil || ![object isKindOfClass:[self class]]) {
		return NO;
	}
	
	return
    (_property1 == object->_property1 ? YES : [_property1 isEqual:object->_property1]) &&
    (_property2 == object->_property2 ? YES : [_property2 isEqual:object->_property2]) &&
    (_property3 == object->_property3 ? YES : [_property3 isEqual:object->_property3]);
}
```


## hash 与判等的关系

hash 方法主要是用于在 Hash Table 查询成员用的, 那么与 isEqual() 有什么关系呢?

为了优化判等的效率，基于 hash 的 NSSet 和 NSDictionary 在判断成员是否相等时，会这样做：

1. 集成成员的 hash 值是否和目标 hash 值相等, 如果相同进入下一步，如果不等，直接判断不相等；
2. hash 值相同的情况下，再进行对象判等，作为判等的结果

简单地说就是

> hash值是对象判等的必要非充分条件


## 文章

[Equality](https://nshipster.com/equality/)