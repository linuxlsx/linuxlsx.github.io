---
layout: post
title: "ARTS 第1周 关键词匹配算法调研"
date: 2019-03-21 23:09:03 +0800
comments: true
categories: ['ARTS']
---

## Algorithm

本周完成的算法题: [Find First and Last Position of Element in Sorted Array](https://leetcode.com/problems/find-first-and-last-position-of-element-in-sorted-array/)

题目要求:

```
Given an array of integers nums sorted in ascending order, find the starting and ending position of a given target value.

Your algorithm's runtime complexity must be in the order of O(log n).

If the target is not found in the array, return [-1, -1].
```

**分析:**

在`O(log n)` 时间复杂度的限制下，肯定是要用二分查找的。但是与普通的查找不同，这个题目实际的要求是查找目标的`左边界`和`右边界`。这样我们通过二分法查找到目标值以后，**不能立刻返回**，而是要继续向左或者向右查看是否有更左或者更右的目标。所以通过两次二分查找即可得到想要的结果。时间复杂度为 `2 * O(log n)`

代码实际比较简单，就不贴代码了。


## Review

本周Review [Why Your Brain Needs Idle Time](https://medium.com/s/the-nuance/why-your-brain-needs-idle-time-e5d90b0ef1df)

`适当的休息更有助于学习和消化知识`

每个人的精力是有限的，每天的工作和社交活动占用了很大的一部分，很多人只有在晚上淋浴和准备睡觉的时候才能让放松下来。

文中研究了`适当休息`是否会对老鼠过迷宫速度有影响。两只老鼠，一只通过迷宫后会休息下然后再重走一次迷宫，而另一只不休息直接重走。结果表明：休息过的老鼠第二次走迷宫的速度会比不休息的要**快**。

所以，不是总是让自己的大脑处于负荷运行的情况，适当的休息并不会降低效率，相反会提高效率。

## Tip

在写代码过程中经常会碰到用一个Map来保存(K,V)结构，如果一个K有多个V的时候，我们需要指定`List<V>`来保存。最近我碰到了一个新的用法，使用`apache-commons 4`中的 `MultiValuedMap`可以帮助我们简化代码。

MultiValuedMap 的类结构如下图: 

![类结构](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/multihashmap.jpg?x-oss-process=style/origin)

主要提供了两种实现：基于ArrayList的实现和基于HashSet的实现。主要区别就是在于对于`Key`是否允许重复的`V`存在。

## Share

[关键词匹配算法调研](http://linuxlsx.top/blog/2019/03/21/guan-jian-ci-pi-pei-suan-fa-diao-yan/)


