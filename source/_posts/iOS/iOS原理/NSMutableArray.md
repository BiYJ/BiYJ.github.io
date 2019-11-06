---
title : NSMutableArray
categories : iOS原理
---

## 一、线程安全

## 二、NSCoding

## 三、NSCoping


## 四、for-in 和 enumerate

```
    NSMutableArray * mArr = [NSMutableArray arrayWithCapacity:10000];
    for (int i = 0; i < 10000; i++) {
        [mArr addObject:@"1"];
    }
    
    CFTimeInterval begin1 = CACurrentMediaTime();
    for (NSString * s in mArr) {

    }
    CFTimeInterval end1 = CACurrentMediaTime();
    
    NSLog(@"1-----%f", end1 - begin1);

    CFTimeInterval begin2 = CACurrentMediaTime();
    [mArr enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {

    }];
    CFTimeInterval end2 = CACurrentMediaTime();
    
    NSLog(@"2-----%f", end2 - begin2);
	
2019-08-30 09:39:39.174152+0800 Demo[51351:647357] 1-----0.000040
2019-08-30 09:39:39.174740+0800 Demo[51351:647357] 2-----0.000488
```

当循环内不做任何操作时，两者相差一个指数级，但时间消耗对于 app 来说都可以忽略不计。

```
	for (NSString * s in mArr) {
        NSLog(@"1");
        NSLog(@"2");
    }

    [mArr enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        NSLog(@"1");
        NSLog(@"2");
    }];
    
2019-08-30 09:43:49.206158+0800 Demo[51419:650339] 1-----3.516211
2019-08-30 09:43:53.465185+0800 Demo[51419:650339] 2-----4.258798

2019-08-30 09:51:57.107924+0800 Demo[51596:657059] 1-----3.845085
2019-08-30 09:52:01.496830+0800 Demo[51596:657059] 2-----4.388771

2019-08-30 09:52:50.881513+0800 Demo[51611:657906] 1-----3.900525
2019-08-30 09:52:55.382088+0800 Demo[51611:657906] 2-----4.500293
```

如上增加相同的简单操作后，两者的是时间相差 0.5 - 0.7 秒，这样的时间差较为客观。但要注意：<font color=#cc0000>当修改循环内的操作时，两者的时间消耗并不确定谁更高效</font>。

```
	for (NSString * s in mArr) {
        NSLog(@"%@", s);
    }
    
    [mArr enumerateObjectsUsingBlock:^(NSString * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        NSLog(@"%@", obj);
    }];
    
2019-08-30 09:54:10.254080+0800 Demo[51636:659160] 1-----1.879232
2019-08-30 09:54:11.944881+0800 Demo[51636:659160] 2-----1.690610

2019-08-30 09:56:28.683932+0800 Demo[51670:660681] 1-----1.923775
2019-08-30 09:56:30.353526+0800 Demo[51670:660681] 2-----1.669475

2019-08-30 09:57:50.068303+0800 Demo[51697:661763] 1-----2.119558
2019-08-30 09:57:51.737062+0800 Demo[51697:661763] 2-----1.668615
```


## 五、Hash

```
{
    NSArray * arr = [NSArray new];
    NSArray * arr2 = @[ @"1", @"2" ];
    
    NSLog(@"%ld, %ld", arr.hash, arr2.hash);
}

0, 2
```

数组的 hash 函数返回的是元素数量。

## 六、componentsJoinedByString

```
{
    NSArray * arr2 = @[ @"1", @(1), [ViewController alloc] ];
    
    NSLog(@"%@", [arr2 componentsJoinedByString:@"|"]);
}
   
1|1|<ViewController: 0x7ff690c183d0>
```

不同类型的对象也可以拼接起来。