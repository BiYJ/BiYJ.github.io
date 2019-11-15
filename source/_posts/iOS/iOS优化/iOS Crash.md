---
title: iOS Crash
categories: iOS优化
---

[Understanding and Analyzing Application Crash Reports](https://developer.apple.com/library/archive/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-INTRODUCTION)



## 稳定性

APP 稳定性方面主要是减少异常及崩溃，针对这部分，可以从 Category、Method Swizzling 以及静态分析来入手。

#### 1.1 Category 分析

Category可以为现有的类添加方法，但是Category方法的不规范使用很容易引发问题，其中最容易出现的就是重名问题。重名问题分为两种情况，一种是APP内的不同业务Category重名，另一种是APP内的Category方法与系统API重名，包括系统私有API。

首先是不同业务的Category方法重名。如果是相同的逻辑，这种情况是方法重复，只保留一份可以减少冗余代码及安装包大小，减少rebase/binding time。如果是不同的逻辑，那么会导致只有一份逻辑生效，其他业务就会产生逻辑错误甚至导致崩溃。

如果是与系统API重名，那么可能影响系统逻辑，尤其是与私有API重名时，很难发现问题（因为document中搜不到）。之前曾发生过APP逻辑异常但一直找不到原因的情况，后来发现是Category方法与系统私有API重名导致。


比较规范的做法是Category名加前缀，但是一些开发人员可能由于各种原因没有这么做。为了及时发现重名情况，爱奇艺技术团队创建了一个工具进行监控，工具会定期将APP中所有Category方法汇总，分析有无重名情况。与系统API的对比需要先将系统API提取出来，公开的API可以通过解析系统库头文件来提取API，私有API可以通过class dump系统库，然后用结果中所有的API减去头文件中的提取的API，就是私有API。这样就得到一份iOS系统的所有API，用APP中的Category方法跟系统所有API求交集，就可以得到与系统API重名的Category方法。


#### 1.2 Method swizzling 分析

Method swizzling 可以解决很多问题，但也会引发一些问题。一般情况下，Method swizzling主要是为系统方法插入一些逻辑，但有时也会导致修改系统逻辑的情况出现。之前遇到过某些业务修改了系统实现，导致特殊情况下APP的崩溃，比如当NSArray的元素超过五万时，对象的一些方法会走系统优化后的方法，所以针对这些方法的swizzling都会出现问题。

为统计各业务Method swizzling的使用情况，团队针对使用了method_exchangeImplementations的库进行初筛，针对使用了swizzling的库使用ar -x后，再针对.o文件进行二次筛选。最后通过反编译工具，反编译.o文件，就可以查到使用swizzling的具体方法。对于新增了swizzling使用的库需要严格审核，确保无问题后才合并到主工程。


#### 1.3 静态分析

静态分析可以有效地发现并预防一些问题，如使用 Xcode 自带的 Analyze 以及 Facebook 的 infer 工具。注册观察者未移除、delegate 没使用 weak 修饰、对象未使用以及未做类型或 null 的判断都可以通过静态分析及时发现。

这块要做的主要目的是跟踪分析结果，了解各版本以及各个库新增了多少，修复了多少，掌握其自动创建修复任务及分配情况，确保问题提早发现，数量能够及时收敛。