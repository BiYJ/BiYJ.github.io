---
title: if-else、switch、while、for
categories: C
---


文章主要会涉及如下几个问题：

1.  if-else 和 switch-case 两者相比谁的效率会高些？在日常开发中该如何抉择？
2.  如何基于赫夫曼树结构减少 if-else 分支判断次数？
3.  如何巧妙的应用 do...while(0) 改善代码结构？
4.  哨兵是什么东西？如何利用哨兵提高有序数组查找效率？
5.  如何降低 for 循环嵌套的时间复杂度？
6.  如何利用策略模式替换繁琐的 if-else 分支？

## 一、if-else 和 switch-case 效率问题

switch-case 与 if-else 的根本区别：

> switch 会生成一个<font color=#cc0000>跳转表</font>来指示实际的 case 分支的地址，而这个跳转表的索引号与 switch 变量的值是相等的。

所以 switch-case 不用像 if-else 那样遍历条件分支直到命中条件，只需访问对应索引号的表项从而到达定位分支。

具体地说，switch-case 会生成一份大小（表项数）为最大 case 常量 +1 的跳转表，程序首先判断 switch 变量是否大于最大 case 常量，若大于，则跳到 default 分支处理；否则取得索引号为 switch 变量大小的跳表项的地址（即跳表的起始地址+表项大小 * 索引号），程序接着跳到此地址执行，到此完成了分支的跳转。

```
int main() {
    unsigned int i, j;
    i = 3;

    switch (i) {
        case 0:
            j = 0;
            break;

        case 1:
            j = 1;
            break;
 
        case 2:
            j = 2;
            break;

        case 3:
            j = 3;
            break;

        case 4:
            j = 4;
            break;

        default:
            j = 10;
            break;
    }
}
```

用 gcc 编译器，生成汇编代码（不开编译器优化）

```
_main:                                  ## @main
Lfunc_begin0:
	.loc	1 12 0                  ## /Users/cykj/Desktop/Demo/Demo/MyC.c:12:0
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	movl	$0, -4(%rbp)
Ltmp0:
	.loc	1 14 7 prologue_end     ## /Users/cykj/Desktop/Demo/Demo/MyC.c:14:7
	movl	$3, -8(%rbp)
	.loc	1 16 13                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:16:13
	movl	-8(%rbp), %eax
	movl	%eax, %ecx
	movq	%rcx, %rdx
	subq	$4, %rdx
	.loc	1 16 5 is_stmt 0        ## /Users/cykj/Desktop/Demo/Demo/MyC.c:16:5
	movq	%rcx, -24(%rbp)         ## 8-byte Spill
	movq	%rdx, -32(%rbp)         ## 8-byte Spill
	ja	LBB0_6
## %bb.8:
	.loc	1 0 5                   ## /Users/cykj/Desktop/Demo/Demo/MyC.c:0:5
	leaq	LJTI0_0(%rip), %rax
	movq	-24(%rbp), %rcx         ## 8-byte Reload
	movslq	(%rax,%rcx,4), %rdx
	addq	%rax, %rdx
	jmpq	*%rdx
LBB0_1:
Ltmp1:
	.loc	1 18 15 is_stmt 1       ## /Users/cykj/Desktop/Demo/Demo/MyC.c:18:15
	movl	$0, -12(%rbp)
	.loc	1 19 13                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:19:13
	jmp	LBB0_7
LBB0_2:
	.loc	1 22 15                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:22:15
	movl	$1, -12(%rbp)
	.loc	1 23 13                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:23:13
	jmp	LBB0_7
LBB0_3:
	.loc	1 26 15                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:26:15
	movl	$2, -12(%rbp)
	.loc	1 27 13                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:27:13
	jmp	LBB0_7
LBB0_4:
	.loc	1 30 15                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:30:15
	movl	$3, -12(%rbp)
	.loc	1 31 13                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:31:13
	jmp	LBB0_7
LBB0_5:
	.loc	1 34 15                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:34:15
	movl	$4, -12(%rbp)
	.loc	1 35 13                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:35:13
	jmp	LBB0_7
LBB0_6:
	.loc	1 38 15                 ## /Users/cykj/Desktop/Demo/Demo/MyC.c:38:15
	movl	$10, -12(%rbp)
Ltmp2:
LBB0_7:
	.loc	1 42 1                  ## /Users/cykj/Desktop/Demo/Demo/MyC.c:42:1
	movl	-4(%rbp), %eax
	popq	%rbp
	retq
Ltmp3:
Lfunc_end0:
	.cfi_endproc
	.p2align	2, 0x90
	.data_region jt32
L0_0_set_1 = LBB0_1-LJTI0_0
L0_0_set_2 = LBB0_2-LJTI0_0
L0_0_set_3 = LBB0_3-LJTI0_0
L0_0_set_4 = LBB0_4-LJTI0_0
L0_0_set_5 = LBB0_5-LJTI0_0
LJTI0_0:
	.long	L0_0_set_1
	.long	L0_0_set_2
	.long	L0_0_set_3
	.long	L0_0_set_4
	.long	L0_0_set_5
	.end_data_region
                                        ## -- End function
```

由此看来，switch 有点以空间换时间的意思，而事实上也的确如此。

1. 当分支较多时，当时用 switch 的效率是很高的。因为 switch 是随机访问的，就是确定了选择值之后直接跳转到那个特定的分支，但是 if-else 是遍历所有的可能值，直到找到符合条件的分支。

	但不总是那么好，因为每次计算会有一个二次查表过程。 具体需要看应用场景，举个例子：对于网络层的协议分析，99% 可能都是 IP 协议，因此基本上会在第一个 if 时就命中，只有一次计算。
	
	总结：对于分支较多或分布相对均匀的情况，使用 switch 可以提高效率；对于分支较少或分布不均匀的情况，使用 if-else 更好。

2. 由上面的汇编代码可知道，switch-case 占用较多的代码空间，因为它要生成跳转表，特别是当 case 常量分布范围很大但实际有效值又比较少的情况，switch-case 的空间利用率将变得很低。

3. switch-case 只能处理 case 为常量的情况，对非常量的情况是无能为力的。例如 if (a > 1 && a < 100)，是无法使用 switch-case 来处理的。所以 if-else 能应用于更多的场合，比较灵活。

文章：[switch 与 if-else 的效率问题](https://blog.csdn.net/kehui123/article/details/5298337)

## 二、用 do-while(0) 改善代码结构

先看一段代码，要重点注意代码中的注释。

```
- (NSString *)handleString:(NSString *)str
{
    if (![str isKindOfClass:[NSString class]]) {
        return nil;
    }
    if(str.length <= 0) {
        return nil;
    }
    // 第一部分逻辑依赖于前面的判断，只有判断通过的时候才执行
    code1...code1
     
    // 第二部分逻辑不依赖于前面的判断(第二部分中的逻辑可能会依赖第一部分逻辑处理结果)，无论判断是否通过都要执行
    code2...code2
}
```

试问，怎样做才能巧妙的满足上述注释代码的需求，因为上述代码中存在 return nil，一旦执行到此处，逻辑一和逻辑二处的伪代码都不会再执行。为了满足上述要求，我们可以巧妙的利用 break 退出临时构造的代码块，但不退出整个函数。

```
- (NSString *)handleString:(NSString *)str {
    do {
        if (![str isKindOfClass:[NSString class]]) {
            break;
        }
        if(str.length <= 0) {
            break;
        }
        // 第一部分逻辑依赖于前面的判断，只有判断通过的时候才执行
        code1...code2
    } while (0);    
    
    // 第二部分逻辑不依赖于前面的判断(第二部分中的逻辑可能会依赖第一部分逻辑处理结果),无论判断是否通过都要执行
    code2...code2
}
```

## 三、有序数组查找操作中的哨兵

正常的查找处理。

```
NSArray *arr = @[@1, @2, @3, @4, @5];
for (NSInteger i = 0; i < arr.count; i++) {
    if ([arr[i] integerValue] == 2) {
        NSLog(@"for 找到了");
    }
}
```

利用哨兵进行查找处理。

```
- (BOOL)search:(NSNumber *)key array:(NSMutableArray *)arr
{
    if (arr.count <= 0) {
        return NO;
    }

    NSNumber * firstObj = (NSNumber *)arr[0];
    if ([firstObj integerValue] == [key integerValue]) {
        return YES;
    }

    NSInteger i = arr.count - 1;
    NSLock * lock = [[NSLock alloc] init];
    [lock lock];
    arr[0] = key;
    // 同上面 for 循环相比，i < arr.count 的判断，在处理大批量数据时候，对性能提升比较大
    while ([arr[i] integerValue] != [key integerValue]) {
        i--;
    }
    arr[0] = firstObj;
    [lock unlock];

    return (i != 0);
}
```

仔细观察上述两段代码，同样是在有序数组中查找目标为 2 的元素，第一段代码是常规迭代处理，第二段代码是将要查找的元素设置为哨兵。同第一段代码相比第二种方式<font color=#cc0000>少了 i < arr.count 的判断</font>，在小批量有序数组查询中对效率的提升并无明显影响，但是在处理大批量数据时候，对性能提升还是比较明显的。

## 四、多层 for 嵌套处理

实际开发中应尽量避免使用双层 for 循环，客户端数据量比较小可能实际开发中并不是很注意这些。但是后端开发过程中，数据量比较大, 为了提升性能，有些公司后端开发中可能会直接规定避免使用多层 for 循环嵌套的形式。<font color=#cc0000>一般第二层或更深层的 for 循环可以使用字典替换</font>。双层 for 循环嵌套的时间复杂度是 n 的二次方。但如果内部 for 循环用字典代替时间复杂度为 O(2n)（实际是 O(n)）。如：两个数组中有且只有一个相同元素，寻找该元素。其中一个数组就可以先用字典做保存，遍历第一个数组的时候，同字典中的数据做比较即可。

```
NSArray *arr1 = @[@1, @2, @3, @4, @5];
NSArray *arr2 = @[@5, @6, @7, @8];
NSMutableDictionary * dict = [NSMutableDictionary dictionary];
for (NSInteger i = 0; i < arr2.count; i++) {
    [dict setObject:arr2[i] forKey:[NSString stringWithFormat:@"%ld", i]];
}

for (NSInteger i= 0 ; i < arr1.count; i++) {
    NSNumber * number = [dict objectForKey:[NSString stringWithFormat:@"%ld", i]];
    if ([arr1[i] integerValue] == [number integerValue]) {
        NSLog(@"相同的数据为:%@", number);
        break;
    }   
}
```

## 五、用策略模式替换 if-else

[https://www.jianshu.com/p/98fa80eebc52](https://www.jianshu.com/p/98fa80eebc52)


## 六、文章

[用if else,switch,while,for颠覆你的编程认知](https://www.jianshu.com/p/ceed2daebc47)