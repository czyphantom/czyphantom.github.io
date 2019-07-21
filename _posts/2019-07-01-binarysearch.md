---
layout:     post
title:      二分答案
subtitle:   
date:       2019-07-01
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - 算法
---

二分搜索我们都知道，但是除了在有序数组中查找某个数之外，二分的思想还可以在其他问题上得到应用，例如在求解最优解的时候。

我们考虑一下“求满足某个条件C(x)的最小的x这个问题，如果对于任意满足C(x)的x，如果所有的x'>=x也满足C(x')，那么我们就可以用二分的方式去找到答案，这就是二分答案。模板如下：

```java
int max = 10000;
int min = 0;

//求某个最大值
public int find(int x) {
    int low = min, high = max;
    int ans = Integer.MIN_VALUE;
    while (low <= high) {
        int mid = (low + high)/2;
        if (check(mid)) {
            low = mid+1;
            ans = mid;
        }
        else {
            high = mid - 1;
        }
    }
    return ans;
}

private boolean check(int x) {
    //check it
}
```

现在来看一道例题：有N条绳子，它们的长度分别为Li。如果从它们中切割出K条长度相同的绳子，这K条绳子能有多长，答案保留道小数点后两位。

套用模板，令C(x)=可以得到K条长度的绳子的长度x。问题变为求解满足C(x)的x的最大值。将C(x)转化一下，改为：C(x)=floor(Li/x)的总和大于等于K。于是可以得到代码如下：

```java
double MAX = 100000.0;

public double getMaxX(double[] l, int k) {
    double low = 0.0, high = MAX+1.0;
    //循环100次直到解的范围足够小
    for (int i = 0;i < 100;i++) {
        double mid = (low+high)/2;
        if (check(mid, l, k)) {
            low = mid;
        } else {
            high = mid;
        }
    }
    return low;
}

private boolean check(double x, double[] l, int k) {
    int count = 0;
    for (int i = 0;i < l.length;i++) {
        count += Math.floor(l[i]/x);
    }
    return count >= k;
}
```

二分答案在极大化最小值、极小化最大值以及极大化平均值中都有相当多的应用。

# 参考资料

[挑战程序设计竞赛](https://book.douban.com/subject/24749842/)
[二分](https://oi-wiki.org/basic/binary/)