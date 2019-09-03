---
title: Category
categories: iOS原理
---


> 分类可以拓展类的属性、方法、协议等信息

## 一、底层结构

在 objc-4 的源码中，搜索 category\_t 可以看到:

```objc
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```

category\_t 就是一个分类的结构体，而我们所创建的的一个分类其实就是一个 category\_t 的结构体，category\_t 里面的结构跟类对象的结构很相似，包含了 name（名称，类名），instanceMethods（对象方法）、classMethods（类方法）、protocols（协议）、属性等。

在编译的时候，分类的属性、方法、协议等会先存储在这个结构体里面，在运行的时候，使用 runtime 动态的把分类里面的方法、属性、协议等添加到类对象（元类对象）中，具体源码可以查看。源码解读顺序： 

#### objc-os.mm

*    \_objc\_init()
*    map\_images()
*    map\_images\_nolock()

#### objc-runtime-new.mm

*    \_read\_images()
*    remethodizeClass()
*    attachCategories()
*    attachLists()
*    realloc、memmove、memcpy

最终可以找到这个方法 attachCategories

```objc
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);
    // 判断是否元类
    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations
    /* 方法数组 @[ @[method_t, method_t], @[method_t .....] ]  */
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    /* 属性数组 @[ @[property_t, property_t], @[property_t .....] ]  */
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    /* 协议数组 @[ @[peotocol_t, peotocol_t], @[peotocol_t .....] ]  */
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[I];

        // 将所有分类的对象方法，附加到类对象列表中
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        // 将所有分类的属性，附加到类属性列表中
        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        // 将所有分类的协议，附加到类协议列表中
        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```

这里，取出所有分类的方法、属性、协议，并将他们各自添加到一个二维数组里，最后再通过 attachLists 将他们添加到类对象中。  


## 二、Category和Class Extension 的区别

Class Extension:

```objc
@interface Person ()
@property (nonatomic, assign) int sex;
- (void)isBig;
@end
```

将属性、方法等封装在 .m 文件里面，类似 private 的应用。 

区别：Class Extension 在编译的时候，数据就已经包含类信息里了；Category 是在运行时，通过 runtime 将数据合并到类信息中。  


## 三、+ Load 和 +initialize 方法的区别

Load：在 runtime 加载类、分类的时候根据函数地址直接调用，程序初始化就会调用，在 Category 中，先调用类的 load（根据编译顺序），再调用分类的 load（根据编译顺序）。 

initialize：在类第一次接收到消息时调用，给类发送消息（objc\_msgSend）才会调用，优先调用父类的 initialize，再调用子类的 initialize，且只会调用一次（父类的 initialize 可能会调用多次）  

## 四、objc\_msgSend() 方法实现

在 objc4 源码中搜索 objc\_msgSend 发现这个方法是由汇编实现的

```objc
/********************************************************************
 *
 * id objc_msgSend(id self, SEL _cmd, ...);
 * IMP objc_msgLookup(id self, SEL _cmd, ...);
 * 
 * objc_msgLookup ABI:
 * IMP returned in x17
 * x16 reserved for our use but not used
 *
 ********************************************************************/

    .data
    .align 3
    .globl _objc_debug_taggedpointer_classes
_objc_debug_taggedpointer_classes:
    .fill 16, 8, 0
    .globl _objc_debug_taggedpointer_ext_classes
_objc_debug_taggedpointer_ext_classes:
    .fill 256, 8, 0

    ENTRY _objc_msgSend
    UNWIND _objc_msgSend, NoFrame
    MESSENGER_START

    cmp x0, #0          // nil check and tagged pointer check
    b.le    LNilOrTagged        //  (MSB tagged pointer looks negative)
    ldr x13, [x0]       // x13 = isa
    and x16, x13, #ISA_MASK // x16 = class  
```

但是可以大概猜出它的实现思路:

1.  由于 initialize 是第一次接受到消息调用，所以 initialize 的调用是在 objc\_msgSend 方法里，所以它的调用顺序应该是在最前面，而且是只调用一次的判断；
2.  通过 isa 寻找类/元类对象，寻找方法调用；
3.  如果 isa 没有寻找到对应的方法，则通过 superClass 寻找父类是否有这个方法，调用。


## 五、文章

[宁夏灼雪__](https://www.jianshu.com/u/edda0ce4a193) & [iOS底层day4 - 探索Category的实现](https://www.jianshu.com/p/1589bc808921)