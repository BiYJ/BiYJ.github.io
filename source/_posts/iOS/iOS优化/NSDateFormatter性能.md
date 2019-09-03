---
title: NSDateFormatter性能
categories: iOS优化
---

## 一、探究

```objc
NSDateFormatter * dateFormatter = [[NSDateFormatter alloc] init];
[dateFormatter setDateFormat:@"yyyy-MM-dd"];
NSString * current = [dateFormatter stringFromDate:[NSDate date]];
```

关于 NSDateFormatter 创建耗时的资料很多，下面开始测试一下，究竟有多耗时。

```objc
double begin = 0.0;
double end   = 0.0;
NSDateFormatter * formatter = nil;

{
    begin = CACurrentMediaTime();
    
    for (int i = 0; i < 1000; i++) {
        formatter  = [[NSDateFormatter alloc] init];
        [formatter setDateFormat:@"yyyy-MM-dd"];
        [formatter stringFromDate:[NSDate date]];
    }
    
    end = CACurrentMediaTime();
    NSLog(@"NSDateFormatter:       %8.2f ms", (end - begin) * 1000);
}

{
    begin = CACurrentMediaTime();
    
    formatter  = [[NSDateFormatter alloc] init];
    
    for (int i = 0; i < 1000; i++) {
        [formatter setDateFormat:@"yyyy-MM-dd"];
        [formatter stringFromDate:[NSDate date]];
    }
    
    end = CACurrentMediaTime();
    NSLog(@"NSDateFormatter once: %8.2f ms", (end - begin) * 1000);
}

-----------Xcode 10.1 iPhone 6s(10.0)----------

2019-03-01 10:08:42.184 Demo[95118:1359994] NSDateFormatter:          48.73 ms
2019-03-01 10:08:42.188 Demo[95118:1359994] NSDateFormatter once:     3.57 ms

2019-03-01 10:11:18.871 Demo[95164:1361958] NSDateFormatter:          61.18 ms
2019-03-01 10:11:18.875 Demo[95164:1361958] NSDateFormatter once:     3.85 ms

2019-03-01 10:12:03.123 Demo[95178:1362677] NSDateFormatter:          79.80 ms
2019-03-01 10:12:03.129 Demo[95178:1362677] NSDateFormatter once:     6.08 ms
```

上面可以看出两者之间消耗时间差距很大。<font color=#cc0000>创建单例很有必要</font>。

那是 [[NSDateFormatter alloc] init] 初始化消耗太高吗？

```objc
NSDateFormatter * formatter = nil;
double begin = 0.0;
double end   = 0.0;
double a = 0, b = 0, c = 0;

for (int i = 0; i < 1000; i++) {
    begin = CACurrentMediaTime();
    formatter  = [[NSDateFormatter alloc] init];
    end = CACurrentMediaTime();
    a += (end - begin);
    
    begin = CACurrentMediaTime();
    [formatter setDateFormat:@"yyyy-MM-dd"];
    end = CACurrentMediaTime();
    b += (end - begin);
    
    begin = CACurrentMediaTime();
    [formatter stringFromDate:[NSDate date]];
    end = CACurrentMediaTime();
    c += (end - begin);
}

NSLog(@"NSDateFormatter:alloc          %8.2f ms", a * 1000);
NSLog(@"NSDateFormatter:setFormat      %8.2f ms", b * 1000);
NSLog(@"NSDateFormatter:stringFromDate %8.2f ms", c * 1000);

-------------Xcode 10.1 iPhone 6s(10.0)-------------

2019-03-01 10:11:18.939 Demo[95164:1361958] NSDateFormatter:alloc              7.01 ms
2019-03-01 10:11:18.939 Demo[95164:1361958] NSDateFormatter:setFormat          0.28 ms
2019-03-01 10:11:18.939 Demo[95164:1361958] NSDateFormatter:stringFromDate    55.98 ms

2019-03-01 10:12:03.198 Demo[95178:1362677] NSDateFormatter:alloc              7.69 ms
2019-03-01 10:12:03.199 Demo[95178:1362677] NSDateFormatter:setFormat          0.25 ms
2019-03-01 10:12:03.199 Demo[95178:1362677] NSDateFormatter:stringFromDate    60.97 ms

2019-03-01 10:18:43.946 Demo[95261:1366071] NSDateFormatter:alloc              6.01 ms
2019-03-01 10:18:43.946 Demo[95261:1366071] NSDateFormatter:setFormat          0.20 ms
2019-03-01 10:18:43.946 Demo[95261:1366071] NSDateFormatter:stringFromDate    49.06 ms
```

从上面可以看出，实际<font color=#cc0000>最耗时的方法是 stringFromDate:/dateFromString:</font>。再往下细究。

```objc
double begin = 0.0;
double end   = 0.0;
NSDateFormatter * formatter = [[NSDateFormatter alloc] init];
    
for (int i = 0; i < 1000; i++) {
    
    [formatter setDateFormat:@"yyyy-MM-dd"];
    
    begin = CACurrentMediaTime();
    [formatter stringFromDate:[NSDate date]];
    end = CACurrentMediaTime();
    
    NSLog(@"%8.2f ms", (end - begin) * 1000);
}

-------------Xcode 10.0 iPhone 6s(10.0)-------------

2019-03-01 10:27:06.218 Demo[95456:1372764]     1.43 ms
2019-03-01 10:27:06.218 Demo[95456:1372764]     0.03 ms
2019-03-01 10:27:06.219 Demo[95456:1372764]     0.02 ms
2019-03-01 10:27:06.219 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.219 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.219 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.219 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.219 Demo[95456:1372764]     0.02 ms
2019-03-01 10:27:06.220 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.220 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.220 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.220 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.220 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.221 Demo[95456:1372764]     0.01 ms
2019-03-01 10:27:06.221 Demo[95456:1372764]     0.01 ms
```

从上面可以看出，只有<font =#cc0000>首次调用 stringFromDate:/dateFromString: 方法才会很耗时</font>。再往下细究。

还有人说应该针对 format 格式创建对应的单例对象。

```objc
double begin = 0.0;
double end   = 0.0;

// 不同的对象不同的 format 格式
{
    begin = CACurrentMediaTime();
    
    NSDateFormatter * formatter1  = [[NSDateFormatter alloc] init];
    NSDateFormatter * formatter2  = [[NSDateFormatter alloc] init];
    NSDateFormatter * formatter3  = [[NSDateFormatter alloc] init];
    NSDateFormatter * formatter4  = [[NSDateFormatter alloc] init];
    
    for (int i = 0; i < 1000; i++) {
        [formatter1 setDateFormat:@"yyyy-MM-dd"];
        [formatter1 stringFromDate:[NSDate date]];
        
        [formatter2 setDateFormat:@"MM-dd-yyyy"];
        [formatter2 stringFromDate:[NSDate date]];
        
        [formatter3 setDateFormat:@"MM-dd"];
        [formatter3 stringFromDate:[NSDate date]];
        
        [formatter4 setDateFormat:@"MM-yyyy"];
        [formatter4 stringFromDate:[NSDate date]];
    }
    end = CACurrentMediaTime();

    printf("NSDateFormatter: different format  %8.2f ms\n", (end - begin) * 1000);
}

// 同一个对象不同的 format 格式
{
    begin = CACurrentMediaTime();
    NSDateFormatter * formatter  = [[NSDateFormatter alloc] init];
    
    for (int i = 0; i < 1000; i++) {
        [formatter setDateFormat:@"yyyy-MM-dd"];
        [formatter stringFromDate:[NSDate date]];
        
        [formatter setDateFormat:@"MM-dd-yyyy"];
        [formatter stringFromDate:[NSDate date]];
        
        [formatter setDateFormat:@"MM-dd"];
        [formatter stringFromDate:[NSDate date]];
        [formatter setDateFormat:@"MM-yyyy"];
        [formatter stringFromDate:[NSDate date]];
    }
    end = CACurrentMediaTime();

    printf("NSDateFormatter:                   %8.2f ms\n", (end - begin) * 1000);
}

---------------Xcode 10.1 iPhone 6s(10.0)---------------


NSDateFormatter: different format     23.26 ms
NSDateFormatter:                      16.25 ms
```

如果不计 NSDateFormatter 对象的初始化时间，那么打印输出：

```objc
NSDateFormatter:different format     23.81 ms
NSDateFormatter:                     23.02 ms
```

两者相差不大，创建一个单例即可。dateFormatter 初次使用时消耗较大，设置 format 格式却并没有什么影响。

## 二、文章
[NSDateFormatter 性能测试](https://www.jianshu.com/p/b000518c3eb8)
