---
layout: post
title: "ARTS 第3周 GC复制算法"
date: 2019-04-08 00:00:13 +0800
comments: true
categories: ["ARTS"]
---

# Algorithm

本周完成的算法题: [Combination Sum](https://leetcode.com/problems/combination-sum/)

题目要求:

```
给定一个不重复的数组(candidates)，以及一个目标(target)，从candidates找到所有的不重复的组合使得他们的和等于target。

Note:
1. 所有的数字都是正整数
2. 同一个数字可以无限次的重复使用
```

<!--more-->

**分析:**

这题可以通过回溯法来解决，通过递归下降多次遍历数组来寻找满足条件的组合。

```Java
public class CombinationSum {

    public List<List<Integer>> combinationSum(int[] candidates, int target) {

        List<List<Integer>> lists = new ArrayList<>(8);

        Stack<Integer> stack = new Stack<>();
        stack.ensureCapacity(candidates.length);
        calc(candidates, target, 0,new Stack<>(), lists);

        return lists;
    }

    /**
     * 递归下降深度优先遍历。 算法复杂度 O((n^2 + n) / 2)
     * 最终结果是 6ms  beats 92%
     *
     * 分析了排在更前面的实现，发现主要的差异就是他们用List替代了Stack
     * 来保存中间状态，这样可以减少一个subList的操作。基本上可以到2ms
     *
     * @param candidates
     * @param target
     * @param start
     * @param stack         需要使用栈来保存中间状态。
     * @param lists
     */
    private void calc(int[] candidates, int target, int start, Stack<Integer> stack, List<List<Integer>> lists){

        for (int i = start; i < candidates.length; i++) {

            if(candidates[i] > target){
                continue;
            }

            stack.push(candidates[i]);
            if(candidates[i] == target){
                List<Integer> l = new ArrayList<>(stack.size());
                l.addAll(stack.subList(0, stack.size()));
                lists.add(l);
            }else if(candidates[i] < target) {
                calc(candidates, target - candidates[i], i ,stack , lists);
            }
            stack.pop();
        }
    }
}    
```

# Review

本周Review [Building and Scaling Data Lineage at Netflix to Improve Data Infrastructure Reliability, and Efficiency](https://medium.com/netflix-techblog/building-and-scaling-data-lineage-at-netflix-to-improve-data-infrastructure-reliability-and-1a52526a7977)

本文是Netflix的一篇技术文章。讲述了Netflix内部是如何构建一个数据链路关系系统的。

公司从小到大的过程中，一定会伴随着数据越来越多，数据链路越来越复杂。很快就没有人能够了解数据全貌了。文章开篇提了三个场景:

* 假设你是一个决策者，当你要根据数据看报做一个关键决策时，是否可以自己去验证下看报背后的数据到底是什么
* 假设你是一个开发者，当你决定要修改你提供的服务的数据结构时，是否可以知道哪些下游会受到影响
* 假设你现在负责平台的可靠性，你的任务是主动监测上游任务的问题，提前给数据作业owner报警。你要设计一个SLA预警系统，需要用到上下游数据依赖以及历史状态数据。`(Finally, imagine yourself in the role of a data platform reliability engineer tasked with providing advanced lead time to data pipeline (ETL) owners by proactively identifying issues upstream to their ETL jobs. You are designing a learning system to forecast Service Level Agreement (SLA) violations and would want to factor in all upstream dependencies and corresponding historical states)`

要满足以上的需求，就需要有一个 `complete and accurate data lineage system`。

自由和责任是Netflix内部文化的重要部分。核心思想是你可以自由的选择实现方式，但是要为其负责。同样这样的自由也会导致公司内部技术栈的多样性，同样也会带来更多的负责度。作者就面临这样的问题。以下是Netflix 的一个`Data Landscape`。

![Data Landscape](https://cdn-images-1.medium.com/max/1400/0*gYI3uCywVhSrcoRo) 

为了满足Netflix内部多样化的数据，`Data Lineage`有以下的几个设计原则：

* `Ensure data integrity` 数据完整性。 需要精确完整的保存数据关系来建立用户的信心。一个不完全可行系统带来伤害可能多过好处。
* `Enable seamless intergration` 无缝接入。需要能够满足新的数据工具快速接入。
* `Design a flexible data model` 灵活的数据结构。使用一个通用灵活的数据模型来表示不同的数据来源。

最终的系统实现图如下：

![系统实现图](https://cdn-images-1.medium.com/max/2600/0*Xp1KHPFm1R7GZGAI)

解释下这个图：

首先左侧是数据接入层，每个业务系统都有它自己独立数据处理逻辑，独立的数据模型。所以在使用之前需要统一进行转换。

中间一层就是数据转换层，转换层将所有收集上来的数据进行转换，统一用点和边的图模式表达。数据转换完毕后会将数据存储到图数据库中。

右侧就是数据存储和服务层。在数据之上建立了的REST服务层，对外提供数据服务。

# Tip

在Java中很多情况需要对一些特殊字符做一些转义。那么一般的写法就是为罗列出所有的需要进行替换的字符，然后不停的调用replace。比如像下面的代码：

```Java
keyword.replace("\\", "\\\\").replace("*", "\\*")
                .replace("+", "\\+").replace("|", "\\|")
                .replace("{", "\\{").replace("}", "\\}")
                .replace("(", "\\(").replace(")", "\\)")
                .replace("^", "\\^").replace("$", "\\$")
                .replace("[", "\\[").replace("]", "\\]")
                .replace("?", "\\?").replace(",", "\\,")
                .replace(".", "\\.").replace("&", "\\&")
                .replace("-", "\\-");
```

这样写虽然能够完成需求，但是不太优雅。在Java的正则表达式中实际是可以支持类似`$0 $1`这样的占位符的。那么像`.replace("[", "\\[")` 可以改写成 `.replace("[", "\\$0")`。 

在这个实现中，`$0` 代表的是整个正则的匹配结果，`$1 $2...$n` 代表的是从1-n的捕获组的内容。

具体的捕获组的介绍可以参考：[正则表达式的捕获组(capture group)在Java中的使用](https://blog.csdn.net/just4you/article/details/70767928)

上面的语句改成类似形式的案例：

```Java
String s = "\\*+|{}^$[]?,.&-";
System.out.println(s);
System.out.println(s.replaceAll("[\\\\\\*\\+\\|\\{\\}\\(\\)\\^\\$\\[\\]\\?\\,\\.\\&\\-]", "\\\\$0"));

//输出
\*+|{}^$[]?,.&-
\\\*\+\|\{\}\^\$\[\]\?\,\.\&\-
```

# Share

[GC算法-复制算法](http://linuxlsx.top/blog/2019/04/07/la-ji-hui-shou-de-suan-fa-yu-shi-xian-du-shu-bi-ji-fu-zhi-suan-fa/)