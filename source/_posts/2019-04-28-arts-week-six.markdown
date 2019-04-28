---
layout: post
title: "arts-week-six"
date: 2019-04-28 21:50:10 +0800
comments: true
categories: ['ARTS', 'Parallel Stream', 'ForkJoinPool', 'Rete']
---

## Algorithm

本周完成的算法题: [Permutations](https://leetcode.com/problems/permutations/)

题目要求:

```
给定一个不重复的整数数组，求所有可能的排列
```

<!-- more -->

本题可以通过回溯法来解决，需要注意排除重复添加元素的情况

```Java
    public List<List<Integer>> permute(int[] nums) {
        List<List<Integer>> lists = new ArrayList<>();
        backtrack(nums, lists, new ArrayList<>());

        return lists;
    }

    private void backtrack(int[] nums, List<List<Integer>> lists, List<Integer> stack) {
        if (stack.size() == nums.length) {
            lists.add(new ArrayList<>(stack));
        } else {
            for (int i = 0; i < nums.length; i++) {
                if (stack.contains(nums[i])){
                    continue;
                }
                stack.add(nums[i]);
                backtrack(nums, lists, stack);
                stack.remove(stack.size() - 1);
            }
        }
    }
```


## Review

本周Review [Introduction to Java 8 Parallel Stream](https://medium.com/javarevisited/java-8-parallel-stream-java2blog-e1254e593763)

本文主要讲述了什么时候应该使用`Parallel Stream`。文中提到`Parallel Stream`要比`Sequential Stream` 花费更多的时间在线程协调上，所以在使用`Parallel Stream` 之前你需要考虑一下的几个因素：

* 要处理的数据集要大
* Java使用`ForkJoinPool`来达到并行的目的。所以源头的集合要可切分(Splitable)，ArrayList 要比 LinkedList 要好
* 现在确实有性能上的问题
* 要处理好数据共享的问题

通过`NQ`模型(来自 Brian Goetz)可以衡量是否要做并行化：

```
N x Q > 10000
```

其中：

N = 数据集中的条目数量（number of items in the dataset）
Q = 每个条目需要的工作（amount of work per item）

解释起来就是：如果数据集多但是每个条目需要工作少，或者数据集不多但是每个条目需要的工作多，你就可以通过并行化来获的性能上的优化。

## Tip

上面Review提到`Parallel Stream`使用到了`ForkJoinPool`，这中间一个小细节需要注意：

默认情况下所有的`Parallel Stream`都是共用`ForkJoinPool.common`这个池子，所以如果Stream后的任务比较耗时(比如有网络请求等)，会堵塞住整个池子，导致其他的Stream也会被影响到。

如果不想被其他的影响也不想影响其他，可以通过自定有的池子来执行。以下是一个示例:

```Java
    long firstNum = 1;
    long lastNum = 1000000;
 
    List<Long> aList = LongStream.rangeClosed(firstNum, lastNum).boxed()
      .collect(Collectors.toList());
 
    ForkJoinPool customThreadPool = new ForkJoinPool(4);
    long actualTotal = customThreadPool.submit(
      () -> aList.parallelStream().reduce(0L, Long::sum)).get();
```

参考:

[Think Twice Before Using Java 8 Parallel Streams](https://dzone.com/articles/think-twice-using-java-8)

[Custom thread pool in Java 8 parallel stream](https://stackoverflow.com/questions/21163108/custom-thread-pool-in-java-8-parallel-stream)

## Share

在项目中碰到了一个因为超大规模的数据(30亿)导致原本执行挺快的模块连续执行超过1天也没有执行完，而且也看不到执行完的希望。于是开始排查是哪里出现了问题。

### 背景描述

业务方基于业务需要配置了约11000条规则，每个规则都是对某个实体的属性判断，最终保存到系统是类似如下的表达式: 

```
"1":"((product_word = '育儿书') OR (product_word = '故事书' AND press = '故事会') OR (product_word = '绘画' AND author = '绘画大师') OR (product_word = '益智游戏类图书') OR (product_word = '儿童读物') OR (product_word = '启蒙认知类图书'))
```

我们需要给出每个规则会命中哪些数据，或者说每条数据会命中哪些规则。

### 原始方案

在最开始的实现中，我们是把规则当做一个求值表达式，通过内部的一个DSL引擎对每个规则来做判断，如果返回`true`就说明命中规则，如果是`false`说明没有命中规则。

单条规则每次求值时间在**0.04ms**，11000条规则每次执行需要**44ms**左右，在个人机器上测试10w条数据需要大概**4000s**，这个时间完全不可接受的。

### 分析问题

经过数据分析发现就是无效的规则执行太多了，在数据确定的情况下，很多规则是肯定不会命中的。比如有一条数据是这样的: 

```
product_word = 育儿书
press = 中信出版社
author = 小蘑菇
```

那它实际需要执行的规则是那些规则条件中包含了 育儿书、中信出版社、小蘑菇的规则，至于其他不包含这些的规则肯定是不会命中的，也就无需执行了。

所以解决方案是：预先为属性、属性值、规则之间建立类似倒排索引的结构，通过属性值来查找可能要执行的规则，然后再执行。

### 解决方案

在解决方案细节中我们需要额外考虑的一点是操作符的问题。对于不同的操作符需要做不同的设计: 

* 对于 **Equal(=)** 是可以直接通过值来获取到所有有关系的规则。
* `>` `>=` `<` `<=` `in` `not in` 等操作符需要做进步一求值才行，但是考虑这些操作符总体使用数量很少，那就直接把这些规则都加到候选规则集中，由求值引擎统一执行即可。


假设我们现在有三个属性 product_word，press，author 以及两个操作符 `=` `in`，那么最终构成的索引结构如下图：

![倒排索引](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E8%A7%84%E5%88%99%E5%BC%95%E6%93%8E%E4%B8%8E%E6%8E%A8%E7%90%86%E5%BC%95%E6%93%8E%E8%9E%8D%E5%90%88-%E6%B1%82%E5%80%BC%E4%BC%98%E5%8C%96.png?x-oss-process=style/black)

上下层级之间实现都是一个HashMap结构，可以快速的定位到目标数据。有了这个结构后再来看看当输入以下这条数据的时候，需要执行哪些规则：

```
product_word = 育儿书
press = 中信出版社
author = 小蘑菇
```

* 通过 `product_word = 育儿书` 可以找到规则 `R:1`
* `product_word` 还有一个 `in` 操作符，根据前面提到的原则把这个操作符关联的规则全部加入候选集。现在的规则包含  `R:1` `R:2`
* 通过 `press = 中信出版社` 可以找到规则 `R:1`
* 通过 `author = 小蘑菇` 可以找到规则 `R:1`

最后要执行的规则集为 `[R:1, R:2]`，比原来的方案要少执行了规则`R:3`。

使用这个方案后实际上一条数据最多可能需要执行的规则为`100条`，更多的少于`10条`。在本地测试同样执行10w条数据，只需要**27s**，性能提升将近**150倍**。

### 总结

在大规模数据处理中，尽可能的**减少输入数据量**，**减少关键步骤执行次数和时间**是优化性能的两大利器。