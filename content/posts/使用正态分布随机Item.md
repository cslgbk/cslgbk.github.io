---
title: "使用正态分布随机Item"
date: 2021-04-12T10:33:21+08:00
draft: false
---
this is a test for markdown
# 1.总体思路
<font size=3/>第一种是在利用SV中的权重分布模拟正态函数分布，第二种是利用DPI接口实现正态函数。考虑到实现难易程度，选择后者。总体框架如图1.1所示。</font>
<br/>

# 2.在C中实现正态分布
<font size=3/>实现正态分布函数有两种方法，一种是采用泰勒展开式，计算出最终值，一种是通过中心-极限定理得出正态分布。正态函数的泰勒展开式 -_-|| ... 一般采用中心-极限定理比较方便。
中心-极限定理的大致解释，每次从这些总体中随机抽取 n 个抽样，一共抽 m 次。 然后把这 m 组抽样分别求出平均值。 这些平均值的分布接近正态分布。</font>

```C代码
    #include<svdpi.h>           //
    #include<stdlib.h>
    unsigned int get_normal()
    {
        unsigned int dat=0;
        unsigned char i;
        unsigned int seed;
        seed = rand();
        srand(seed);
        for (i=0;i<10;i++)
        {
            dat += rand()%99;
        }
        return dat/i;
    }

```