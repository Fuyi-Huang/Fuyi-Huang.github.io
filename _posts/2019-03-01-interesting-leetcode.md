---
title: leetcode 刷题总结
tags: 
    - programming 
    - c++
    - leetcode
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
mathjax: true
mathjax_autoNumber: true
---

## 前言

> 这篇文章呢主要是记录我刷[leetcode](https://leetcode.com)时遇到的有趣的题。
这里说的有趣呢包含两方面，可能是题目很有趣，也有可能是它解法很有趣。

## leetcode题目

### sqrt(x)
> [leetcode链接](https://leetcode.com/problems/sqrtx/)

这题是牛顿法的应用，我一直很喜欢那些应用数学方法从而巧妙解决的编程题，所以这篇文章大概会收集很多类似的题目。

> 牛顿法([维基百科](https://zh.wikipedia.org/wiki/%E7%89%9B%E9%A1%BF%E6%B3%95))：求解f(x) = 0的一个解可以借由f(x)的导数（或者是其泰勒级数的前几项）求解
    首先，选择一个点x0，求解x0处切线与x轴的交点x1，也就是求解：
    
$$ 0 = (x - x0) * f'(x0) - f(x0) $$

> 通常求得的这个解x1会比x0更接近f(x)的零点，所以我们可以不断去迭代逼近零点。迭代方程如下：

$$ x_{n + 1} = x_{n} - \frac{f(x)}{f'(x)} $$

牛顿法一般可认为是求解平方根的最优解了，根据牛顿法我们可以写出求解平方根的迭代公式：$ x_{n + 1} = x_{n} - \frac{x^2}{2x_{n}} $

```c++
int mySqrt(int x)
{
    double res = x;
    while ( abs(res * res - x) < 1 )
    {
        res = res - (res * res - x) / (2 * res);
    }
    return (int)res;
}
```

但是呢，我一开始看题目只需要求解x的平方根向下取整的整数，所以我一开始是这样写的：

```c++
int mySqrt(int x)
{
    long res = x;
    while ( abs(res * res - x) < 1 )
    {
        res = res - (res * res - x) / (2 * res);
    }
    return res;
}
```

奇怪的是这样子写连第一个测试用例的通不过，第一个测试用例是4:

    4
    3
    3
    ...

原因呢也很简单，当不断迭代到 res * res 接近于x时，` (res * res - x) / (2 * res) `一直是0，因为x和res都是整数，所以迭代就变成了` res = res `

```c++
int mySqrt(int x)
{
    long res = x;
    while ( abs(res * res - x) < 1)
    {
        res = res - (res * res - x) / (2 * res) - 1;
    }
    return res;
}

// 或者更优雅的写法是
int mySqrt(int x)
{
    long res = x;
    while ( res * res > x)
    {
        res = (res + x / res) / 2;
    }
    return res;
}
```
