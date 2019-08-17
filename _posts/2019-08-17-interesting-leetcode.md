---
title: leetcode Combinations 刷题总结
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

### 77. Combinations

> [leetcode链接](https://leetcode.com/problems/combinations/)

这题是要根据n，k输出1...n中的k个数的所有组合。
c++中两个很好用的函数prev_permutation和next_permutation, prev_permutation是按照字典顺序改变当前数组成小一点的数组，next_permutation是刚好大的数组。
prev_permutation用来解这题很方便。

``` c++
class Solution {
public:
    vector<vector<int>> combine(int n, int k) {
        vector<vector<int>> res;
        if (k <= 0) {
            return res;
        }

        vector<int> perm = vector<int>(n, 0);
        for (int i = 0; i < k; i++) {
            perm[i] = 1;
        }

        vector<int> temp = vector<int>(k, 0);
        do {
            int j = 0;
            for (int i = 0; i < n; i++) {
                if (perm[i] == 1) {
                    temp[j++] = i + 1;
                }
            }
            res.push_back(temp);
        } while (prev_permutation(perm.begin(), perm.end()));
        return res;
    }
};
```

解法是不断生成{1, 1, 1, 0, 0}的上一个排列，直到{0, 0, 1, 1, 1}，挑出排列中为1的下标对应的{1, 2, 3, 4, 5}对应的数，挑出来的一个数组就是一个组合。

next_permuation的可能实现：
```c++
template<class BidirIt>
bool next_permutation(BidirIt first, BidirIt last)
{
    if (first == last) return false;
    BidirIt i = last;
    if (first == --i) return false;
 
    while (true) {
        BidirIt i1, i2;
 
        i1 = i;
        if (*--i < *i1) {
            i2 = last;
            while (!(*i < *--i2))
                ;
            std::iter_swap(i, i2);
            std::reverse(i1, last);
            return true;
        }
        if (i == first) {
            std::reverse(first, last);
            return false;
        }
    }
}
```

题外话：
我在做这题的时候想知道怎么算 $ C_{(n, k)} $好，公式是：

$$ C_{(n, k)} = \frac{n!}{k!(n - k)!} = \frac{n * (n - 1) ... (n - k + 1)}{k!} $$

实现：用$ C_{(n, k)} = \frac{n * (n - 1) ... (n - k + 1)}{k!} $ 这个转换

```c++
int combination_number(int n, int k) {
    int res = 1;
    for (int i = 1; i <= k; i++) {
        // res *= (n - i + 1) / i;
        res *= (n - i + 1);
        res /= i;
    }
    return res;
}
```
