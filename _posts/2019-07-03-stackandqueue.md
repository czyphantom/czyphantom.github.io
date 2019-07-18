---
layout:     post
title:      单调栈和单调队列
subtitle:   
date:       2019-07-03
author:     CZY
header-img: img/bg.png
catalog: true
tags:
    - 算法
---

# 单调栈

单调栈也就是满足单调性的栈结构，将一个元素插入单调栈时，需要保证该元素插入到栈顶后整个栈满足单调性的前提下弹出最少的元素。

看一道经典的例题：现在有n个宽度为1、高度分别为h1,h2，h3......hn的长方形从左到右排列成柱状图，求包含的长方形的最大面积是多少。

分析一下这道题，要使面积最大，那么需要柱状图中最矮的那个柱的高度要尽量高，可以使用单调栈保存柱的索引，代码如下：

```java
public int solve(int[] h) {
    int res = 0;
    Stack<Integer> stack = new Stack<>();
    for (int i = 0;i < h.length;i++) {
        while (!stack.isEmpty() && h[stack.peek()] >= h[i]) {
            int cur = stack.pop();
            res = Math.max(res, h[i] * (stack.isEmpty() ? i : (i - stack.peek() - 1)));
        }
        stack.push(i);
    }
    return res;
}
```

# 单调队列

单调队列的典型使用就是滑动窗口，看一道经典的题目：给定一个长度为n的数列和一个整数k，求长度为k的子数组的最小值的集合。

暴力解法就直接O(nk)，但是可以发现其实很多比较都是不必要的，我们使用双端队列保存数字下标，在加入一个数a[i]时，如果双端队列的末尾a[j]>=a[i]时，此时不断从末尾移除数字，这些数字都不可能是最小值，直到a[j] < a[i]或者队列为空，最后待k个数都加入后，队头元素即为最小值。代码如下：

```java
public int[] solve(int[] nums, int k) {
    int[] res = new int[nums.length - k];
    Dequeue deq = new LinkedList<>();
    for (int i = 0;i < nums.length;i++) {
        while(!deq.isEmpty() && deq.peekLast() >= nums[i]) {
            deq.pollLast();
        }
        deq.offer(i);
        if (i - k + 1 >= 0) {
            res[i-k+1] = nums[deq.peekFirst()];
            if (deq.peekFirst() == i-k+1) {
                deq.pollFirst();
            }
        } 
    }
    return res;
}
```