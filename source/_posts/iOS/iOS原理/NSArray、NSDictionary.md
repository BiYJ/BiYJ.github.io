---
title: NSArray、NSDictionary
categories: iOS原理
---


## 一、NSDictionary

#### 1.1 使用自定义对象 key

> Dictionaries manage pairs of keys and values. A key-value pair within a dictionary is called an entry. Each entry consists of one object that represents the key, and a second object which is that key’s value. Within a dictionary, the keys are unique—that is, no two keys in a single dictionary are equal (as determined by isEqual:). A key can be any object that adopts the NSCopying protocol and implements the hash and isEqual: methods. 

字典管理着键值对。键是唯一的，同一个字典里面不会有相同的 key（由 isEqual: 确定），key 可以是遵守 `NSCoping` 协议并实现 `hash` 和 `isEqual:` 方法的任何对象。

```
- (instancetype)copyWithZone:(NSZone *)zone
{  
    return self;
}
```

<center>
![](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Art/Dictionary_2x.png)
</center>

如果字典很少更改或大量更改，则应使用不可变的字典；否则使用可变字典。

> Objects added as values to a dictionary are not copied (unless you pass YES as the argument to initWithDictionary:copyItems:). Rather, a strong reference to the object is added to the dictionary. 

作为 value 添加到字典的对象不会被复制（除非调用 `initWithDictionary:copyItems:` 方法，并且参数传递 YES），而是将对对象的强引用添加到字典中。[Copying Collections](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Articles/Copying.html#//apple_ref/doc/uid/TP40010162-SW1)

> You can also add entries from another dictionary using the addEntriesFromDictionary: instance method. If both dictionaries contain the same key, the receiver’s previous value object for that key is released and the new object takes its place.

使用 `addEntriesFromDictionary:` 从一个已有的字典添加键值对时，相同 key 对应的 value 会被替换。

> Keys must conform to the NSCopying protocol. Methods that add entries to dictionaries—whether as part of initialization (for all dictionaries) or during modification (for mutable dictionaries)— don’t add each key object to the dictionary directly. Instead, they copy each key argument and add the copy to the dictionary. After being copied into the dictionary, the dictionary-owned copies of the keys should not be modified.
> 
> Keys must implement the hash and isEqual: methods because a dictionary uses a hash table to organize its storage and to quickly access contained objects. In addition, performance in a dictionary is heavily dependent on the hash function used. With a bad hash function, the decrease in performance can be severe. For more information on the hash and isEqual: methods see [NSObject](https://developer.apple.com/library/archive/documentation/LegacyTechnologies/WebObjects/WebObjects_3.5/Reference/Frameworks/ObjC/Foundation/Protocols/NSObject/Description.html#//apple_ref/occ/intf/NSObject).

使用自定义对象 key 时，对象需要遵守 `NSCoping` 协议，存储数据时，字典会持有 key 的复制，复制到字典中后，不应修改字典拥有的键的副本。

键必须实现 hash 和 isEqual: 方法，因为字典使用<font color=#cc0000>哈希表</font>来组织其存储并快速访问所包含的对象。此外，字典的性能在很大程度上取决于所使用的哈希函数。如果哈希函数不正确，性能下降可能会很严重。

**注意**：不要使用一个很大的对象作为 key，例如 NSImage 对象，这样会导致性能下降。


#### 1.2 NSMapTable

NSMapTable 允许针对特定情况（例如，当您需要高级内存管理选项时或想要保留特定类型的指针时）定制其他的存储选项。如图中的映射表配置为保存对其值对象的<font color=#cc0000>弱引用</font>。

![](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Art/MapTable_2x.png)

还可以指定是否要复制输入到数组中的对象。

> The NSMapTable class also defines a number of convenience constructors for creating a map table with strong or weak references to its contents. For example, mapTableWithStrongToWeakObjects creates a map table that holds strong references to its keys and weak references to its values. These convenience constructors should only be used if you are storing objects.

使用 `mapTableWithStrongToWeakObjects` 可以强引用 key，弱引用 value。

为了能够使用任意的指针，请同时使用 `NSPointerFunctionsOpaqueMemory` 和 `NSPointerFunctionsOpaquePersonality` 值选项对其进行初始化。键和值的 options 不必相同。当使用映射表包含任意指针时，指针类型应该使用 c 语言的 void *。

```
NSPointerFunctionsOptions keyOptions   = NSPointerFunctionsStrongMemory | NSPointerFunctionsObjectPersonality | NSPointerFunctionsCopyIn;
NSPointerFunctionsOptions valueOptions = NSPointerFunctionsOpaqueMemory | NSPointerFunctionsOpaquePersonality;
 
NSMapTable * mapTable = [NSMapTable mapTableWithKeyOptions:keyOptions valueOptions:valueOptions];
 
int i = 1;
NSString * key = @"Key";
NSMapInsert(mapTable, key, &i);

NSLog(@"Key contains: %i", *(int *) NSMapGet(mapTable, key));
```


#### 1.3 底层原理

> Objective-C 中的字典 NSDictionary 底层其实是一个哈希表。

NSDictionary 使用 hash 表来实现 key 和 value 之间的映射和存储，hash 函数设计的好坏影响着数据的查找访问效率。数据在 hash 表中分布的越均匀，其访问效率越高。而在 Objective-C 中，通常都是利用 NSString 来作为键值，其内部使用的 hash 函数也是通过使用 NSString 对象作为键值来保证数据的各个节点在 hash 表中均匀分布。

字典每个条目存取是通过将字典的键(Key)计算出键的 hash 值，通过查 hash 表获取具体的 value。所以作为 NSDictionary 的键(Key)取值的时候，只要其 key 对象内容地址相同就可以取出相应的值。


## 二、数组

#### 2.1 NSPointArray

NSPointArray 类似 NSMutableArray，但它可以容纳 nil 值，同 count 值也包含 nil 的数量。它还允许你在定制特定的情况下设置额外的存储选项。

```
NSPointerArray * arr = [NSPointerArray pointerArrayWithOptions:NSPointerFunctionsStrongMemory];
[arr addPointer:nil];
[arr addPointer:@"sss"];

NSLog(@"%lu", arr.count);
NSLog(@"%@", [arr pointerAtIndex:0]);
NSLog(@"%@", [arr pointerAtIndex:1]);
    
arr.count = 1;
NSLog(@"%@", [arr pointerAtIndex:0]);
NSLog(@"%@", [arr pointerAtIndex:1]);  // 崩溃。-[NSConcretePointerArray pointerAtIndex:]: attempt to access pointer at index 1 beyond bounds 1'
```

![](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Art/PointerArray_2x.png)

当您想要使用弱引用的有序集合时，可以使用 NSPointerArray 对象。例如，假设您有一个包含一些对象的全局数组。因为从不收集全局对象，所以除非将其保持为弱状态，否则不能释放其所有内容。配置为弱保存对象的指针数组不拥有其内容。如果在此类指针数组中没有对对象的强引用，则可以释放这些对象。

为了使用任意的指针，可以同时使用 `NSPointerFunctionsOpaqueMemory` 和`NSPointerFunctionsOpaquePersonality` 选项对其进行初始化。


```
NSPointerFunctionsOptions options = (NSPointerFunctionsOpaqueMemory | NSPointerFunctionsOpaquePersonality);
    
NSPointerArray * ptrArray = [NSPointerArray pointerArrayWithOptions:options];
    
int i = 1;
[ptrArray addPointer:&i];
    
NSLog(@" Index 0 contains: %i", *(int *)[ptrArray pointerAtIndex:0]);
```

## 三、数据拷贝

> The normal copy is a shallow copy that produces a new collection that shares ownership of the objects with the original. Deep copies create new objects from the originals and add those to the new collection.

通常情况是浅拷贝，它会生成一个新集合，该集合与原始对象共享对象的所有权。深拷贝从原始对象创建新对象，并将其添加到新集合中。

![](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Art/CopyingCollections_2x.png)

>When you create a shallow copy, the objects in the original collection are sent a retain message and the pointers are copied to the new collection.

浅拷贝时，原始数据集合中的对象会发送 <font color=#cc0000>`retain`</font> 消息，并将指针拷贝到新数据集合中。

> You can use the collection’s equivalent of initWithArray:copyItems: with YES as the second parameter. If you create a deep copy of a collection in this way, each object in the collection is sent a copyWithZone: message. If the objects in the collection have adopted the NSCopying protocol, the objects are deeply copied to the new collection, which is then the sole owner of the copied objects. If the objects do not adopt the NSCopying protocol, attempting to copy them in such a way results in a runtime error. However, copyWithZone: produces a shallow copy. This kind of copy is only capable of producing a one-level-deep copy.

使用 copyItem:YES 方法时，会给集合中的每个对象发送 copyWithZone: 消息。如果对象实现了 NSCopying 协议，那么这些对象会被深拷贝到新的数据集合；如果没有实现协议，则会导致运行时错误。

然而 copyWithZone: 生成的是浅层的“深拷贝”，只是做了一层的深拷贝，如果数据是多层数组嵌套的话，则第二层、第三层...的数据没有进行深拷贝。

> If you need a true deep copy, such as when you have an array of arrays, you can archive and then unarchive the collection, provided the contents all conform to the NSCoding protocol.

如果想真正的深拷贝，只要内容全部符合 <font color=#cc0000>`NSCoding`</font> 协议，则可以存档然后取消存档该集合。

```
NSArray * trueDeepCopyArray = [NSKeyedUnarchiver unarchiveObjectWithData：
          [NSKeyedArchiver archivedDataWithRootObjectoldArray]];
```

复制集合时，该集合或其包含的对象的<font color=#cc0000>`可变性`</font>可能会受到影响。

* copyWithZone: makes the surface level immutable. All deeper levels have the mutability they previously had.  第一层是可变的，其余深层还是以前的可变性。

* initWithArray:copyItems: with NO as the second parameter gives the surface level the mutability of the class it is allocated as. All deeper levels have the mutability they previously had.  第一层保持分配时的可变性，其余深层还是以前的可变性

* initWithArray:copyItems: with YES as the second parameter gives the surface level the mutability of the class it is allocated as. The next level is immutable, and all deeper levels have the mutability they previously had.
Archiving and unarchiving the collection leaves the mutability of all levels as it was before.  第一层保持分配时的可变性，第二层是可变的，其余深层还是以前的可变性


## 四、文章

[Dictionaries: Collections of Keys and Values](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Collections/Articles/Dictionaries.html#//apple_ref/doc/uid/20000134-CJBCBGII)