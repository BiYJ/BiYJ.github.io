---
title: 第 k 小/大元素
categories: algorithm
---

```
#include<bits/stdc++.h>
using namespace std;

void swap_t(int a[],int i,int j)
{
    int t=a[i];
    a[i]=a[j];
    a[j]=t;
}

int par(int a[],int p,int q)//p是轴,轴前面是比a[p]小的，后面是比a[p]大的
{
    int i=p,x=a[p];
    for(int j=p+1;j<=q;j++)
    {
        if(a[j]>=x)
        {
            i++;
            swap_t(a,i,j);
        }
    }
    swap_t(a,p,i);
    return i;//返回轴位置
}

int Random(int p,int q)//返回p，q之间的随机数
{
    return rand()%(q-p+1)+p;
}

int Randomizedpar(int a[],int p,int q)
{
    int i=Random(p,q);
    swap_t(a,p,i);//第一个和第i个交换，相当于有了一个随机基准元素
    return par(a,p,q);
}

int RandomizedSelect(int a[],int p,int r,int k)
{
    if(p==r)
        return a[p];
    int i=Randomizedpar(a,p,r);
    int j=i-p+1;
    printf("i=%d j=%d\n",i,j);
    if(k<=j)
        return RandomizedSelect(a,p,i,k);
    else
        return RandomizedSelect(a,i+1,r,k-j);
}

int main()
{
    int n;
    scanf("%d",&n);
    int a[n];
    for(int i=0;i<n;i++)
    {
        scanf("%d",&a[i]);
    }
    int x=RandomizedSelect(a,0,n-1,2);
    printf("%d\n",x);
}
```