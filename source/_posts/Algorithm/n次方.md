---
title: n 次方
categories: algorithm
---

## 1、问题描述

计算 a<sup>n</sup>

## 2、算法分析

先将 n 变一变，寻找新的计算路径。<font color=#cc0000>预处理就是变治法的根本</font>。

如果单纯循环执行 n 次相乘，那么时间复杂度为 O(n)。可以利用<font color=#cc0000>二进制幂</font>大大改进效率。

主要思路是：将十进制的 n 转换成二进制的数组序列 b\[\]。二进制幂求解有两种方法：从左至右二进制幂和从右至左二进制幂。

①、从左至右二进制幂

变换：a<sup>n</sup> = a<sup>(b\[n\]2<sup>m</sup> \+ ... \+ b\[0\]2<sup>0</sup>)</sup>

先求 n 的二进制串，如：n = 5 => 1 0 1，那么 b\[2\] = 1, b\[1\] = 0, b\[0\] = 1

二进制求 n 的伪代码：

Horner(b[0...n], x)  
k = b[n]  
for i = n-1 downto 0 do  
  p = x*k + b[i]  
return p

那么 n 用作 a 的指数时意义是什么样的呢：

a<sup>p</sup> = a<sup>1</sup>  
for i = n - 1 downto 0 do  
  a<sup>p</sup> = a<sup>(2p+b[i])</sup>

②、从右至左二进制幂

n 变换方法与上面相同，然后从 b\[0\] -> b\[n\] 方向逐步求解。

时间复杂度：O(logn)

## 3、代码实现

```c
#include <stdio.h>
#include <stdlib.h>

/**
  *  @brief  返回 x 的二进制串（数组）
  */
int GetBinArray(int x, int arr[], int length)
{
    int idx = 0;
    
    while(x > 0) {
        
        // 获取末位的二进制
        arr[idx++] = (x & 1) ? 1 : 0;
        if (idx == length)
            break;
        
        // 右移两位
        x = x >> 1;
    }
    
    return idx;
}

/**
  *  @brief   a^n = a^（b[n]2^n + ... + b[0]2^0）= a^（b[n]2^n）* ... * a^b[0]。 b 数组元素不是 1 就是 0
  */
int Pow_Bin_RightToLeft(int number, int power)
{
    if (power == 0)
        return 1;
    
    int length = sizeof(int) * 8; // 32
    int *pint = (int *)malloc(length);
    
    // 获取幂的二进制数组
    length = GetBinArray(power, pint, length);
    
    int item = number;
    int ret  = 1;
    
    for (int i = 0; i < length; i++) {
        
        // 二进制值为 1，计入结果
        if (pint[i] == 1)
            ret *= item;
        item *= item;
    }
    free(pint);
    
    return ret;
}

/**
  *  @brief   a^n = a^（b[n]2^n + ... + b[0]2^0）=（（b[n]*2 + b[n-1]）*X +  ....）2 + b[0]。 b 数组元素不是 1 就是 0
  */
int Pow_Bin_LeftToRight(int number, int power)
{
    if (power == 0)
        return 1;
    
    int length = sizeof(int)*8;
    int *pint = (int *)malloc(length);
    
    length = GetBinArray(power, pint, length);
    
    int ret = number;
    
    for (int i = length - 1 - 1; i >= 0; i--) {
        
        ret *= ret;
        
        if(pint[i] == 1)
            ret *= number;
    }
    
    free(pint);
    
    return ret;
}

int main()
{
    int num = 8, power = 6;
    int ret1 = Pow_Bin_RightToLeft(num, power);
    int ret2 = Pow_Bin_LeftToRight(num, power);
    
    printf("Pow_Bin_RightToLeft: %d^%d == %d\n", num, power, ret1);
    printf("Pow_Bin_LeftToRight: %d^%d == %d\n", num, power, ret2);
    
    return 0;
}

Pow_Bin_RightToLeft: 8^6 == 262144
Pow_Bin_LeftToRight: 8^6 == 262144
```

文章：[计算 n 次方--变治法](https://blog.csdn.net/Jammg/article/details/51694140)