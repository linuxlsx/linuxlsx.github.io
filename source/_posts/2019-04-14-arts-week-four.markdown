---
layout: post
title: "ARTS 第4周 负载均衡算法"
date: 2019-04-14 00:06:02 +0800
comments: true
categories: ["ARTS", "负载均衡算法", "一致性Hash"]
---

## Algorithm

本周完成的算法题: [First Missing Positive](https://leetcode.com/problems/first-missing-positive/)

本题非独立完成，根据[Find the Smallest Integer Not in a List](https://stackoverflow.com/questions/1586858/find-the-smallest-integer-not-in-a-list) 参考实现

题目要求:

```Text
给定一个没有排序的Int数组，做到数组中不存在的最小正整数。

Note:
1. 算法复杂度必须是O(n)
2. 使用常数个额外的空间
```

<!-- more -->

**分析:**

这个题的解法很巧妙，通过利用数组的下标和其对应的值做比较就可以确定确实的最小正整数。算法运行时间至多为`O(3N)`，而且只需要一个额外的Int变量。

```Java
    public int firstMissingPositive(int[] nums) {

        int p;
        //第一个O(N)
        for (int i = 0; i < nums.length; i++) {
            p = nums[i];
            
            //为什么这里也只会最多执行O(N)次
            //因为在条件中会判断 nums[p - 1] != p，然后在循环体里面会设置 nums[p - 1] = p
            //那么每执行一次赋值，就会减少一个可赋值的位置。
            //就可以保证至多执行N次
            while (p > 0 && p < nums.length && nums[p - 1] != p) {
                nums[i] = nums[p - 1];
                nums[p - 1] = p;
                p = nums[i];
            }
        }

        //第三个O(N)
        for (int i = 0; i < nums.length; i++) {
            if (i + 1 != nums[i]) {
                return i + 1;
            }
        }

        ////只有数组是 [1,2,3,4] 这种情况才会走到这里
        return nums.length + 1;
    }
```

## Review

本周Review [What is Clean Code?](https://medium.com/s/story/reflections-on-clean-code-8c9b683277ca)

本文是作者对于《Clean Code》一书的总结，从他的视角来看，可以归纳总结为三点: 

* **工匠精神(Craftsmanship Matters)** 可以`Works`的代码并不能说已经`done`，粗制滥造的代码会比你想象的快的出现问题。(Poorly crafted code frays at the edge much faster than you might expect.)
* **今天的额外努力会减少明天的痛苦(Extra Effort Today Saves Pain Tomorrow)** 就跟买衣服，鞋子一样，好的鞋子可能在价格上会高些，但是它可以穿两年、三年甚至十年，那么长远来看，这个价格就不高了。高质量的代码也是一样的，可以在开始时会困难些，但是会降低未来的技术债和维护成本。
* **你写的代码不仅仅是你的(Your Code Is Not Your Own)** 过于聪明的把戏，黑科技和编程技巧只是当时作者的乐趣(Overly clever tricks, hacks, and sleights of programmatic hand are only fun for the author.)，但是会给后续的维护者(也可能是你自己)带来无数的麻烦。

对于什么是`Clean Code`，也总结了一下几点:

* Clean code is simple。虽然可能在算法或者系统层级上复杂，但是实现上要简单。少用比较冷门的技巧。
* Clean code is readable。在命名规范、缩进、结构和流程要有良好的设计，虽然显得有些古板但是会减少后来着的理解难度。
* Clean code is considerate。Clean code 要为后续的读者考虑，可以假设他们也是拥有同样背景和经验的人。
* Clean code is tested。Clean code 要有充分的测试
* Clean code is practiced。Clean code 需要多练习
* Clean code is relentlessly refactored。Clean code 需要经常重构
* Clean code is SOLID。遵循[Solid 原则](https://medium.com/@severinperez/writing-flexible-code-with-the-single-responsibility-principle-b71c4f3f883f)


结合自身的实际来说，我觉得最重要的是 `tested` 和 `readable`。只有测试过的代码才能保证联调、测试和线上运行的时候不出现问题。另外很少有情况是你一直维护一套代码，所以让代码有良好的可读性会对后来者有很大帮助，同时自己也会有成就感。

## Tip

本周分享一个线上排查问题时需要将日志内容按照不同的要求写到不同的文件。

```Bash
awk -F',' '{if ($2 ==1) print $1 > "down.txt"; else if($2 == 6) print $1 > "delete.txt"; else print $1 > "other.txt"}' xxx.log
```

## Share

**主要负载均衡算法**

分布式场景下client 访问 server 的策略有很多，典型的有 轮询，随机，Hash，最少连接，响应速度，加权算法等。

**轮询算法**：将所有请求，依次分发到每台服务器上。缺点: 所有机器的请求一样， 处理能力慢的机器压力变大。
**随机算法**：请求随机分配到各个服务器。简单。缺点：理论上所有机器的请求一样，处理能力慢的机器压力变大。
**最少连接**：将请求分配到连接数最少的服务器（目前处理请求最少的服务器）。优点：根据服务器当前的请求处理情况，动态分配。缺点: 实现比较复杂，需要感知服务器的状态
**Hash** : 根据IP地址进行Hash计算，得到IP地址。优点： 将来自同一IP地址的请求，同一会话期内，转发到相同的服务器；实现会话粘滞。缺点：机器变化时会需要重新计算，最坏情况下所有请求都需要变化。
加权算法：在轮询，随机，最少链接，Hash’等算法的基础上，通过加权的方式，进行负载服务器分配。优点：根据权重，调节转发服务器的请求数目；缺点：使用相对复杂；
**一致性 Hash** ：一致性Hash 的实现是将 node 映射到一个 固定长度的圆环上，请求计算出Hash值按照预定的规则（一般是按照顺时钟方向）落到最近的服务器上。这样当出现机器的变化时，只会影响受影响机器与该机器反方向最近有效服务器之间的节点。其他的节点不会收到影响。当新增node时，只需要计算出该node的hash地址，然后将node 添加到 环的对应位置，只有新node 与后一个node之间的数据会受到影响。这种设计下的扩展性和容错性都比较好。比较不好的问题是在服务器比较少的情况下容易出现负责不均和的情况下。

一致性Hash 的实现是将 node 映射到一个 固定长度的圆环上，并在有效节点中引入多个虚节点（每个虚节点都是有效的的一个影子）。这个所有请求能够均匀的落到node上。

在现在的一致性hash实现上，Google 的研究人员发现在多次负载均衡过后不能够获得最优的负载均衡的效果，开发一个带负载因子(a)的一致性hash算法。

这个算法的能够保证一致性和负责的均衡性（所有的机器的load 都是平均load 的 （1+a）倍）。当出现机器的新增和删除后，通过节点移动的操作重新使得负载重新达到均衡，而这个移动的次数只跟负载因子(a)相关，而与实际节点数的多少无关。每个新增或者删除节点的动作只会导致  O(1/a2) 次其他节点的移动。 

更多一致性Hash参考 : [一致性Hash在负载均衡中的应用](https://juejin.im/post/5b8f93576fb9a05d11175b8d) 