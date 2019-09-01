---
layout: post
title: "ARTS 第2周 GC引用计数算法"
date: 2019-03-29 21:25:12 +0800
comments: true
categories: ["ARTS", "Sudoku", "GC", "引用计数"]
---

## Algorithm

本周完成的算法题: [Valid Sudoku](https://leetcode.com/problems/valid-sudoku/)

题目要求:

```
通过二维数组定义了9x9的数独库，这个数独库的某些位置会被数字填充，其他地方填充'.'，现在需要你来验证这个数独库是否合法的。有效数独的规则如下:

1. 每一行都是1-9不重复的数字
2. 每一列都是1-9不重复的数字
3. 每一个3x3的小方格中也是1-9不重复的数字
```

<!-- more -->

**分析:**

要完成有效性的判定，双重循环遍历是肯定要的。我们需要考虑的是如何只使用双重循环就把所有事情搞定。通过对矩阵的观察，我们发现有以下的规律:

* 当我们访问`soduku[i][j]`，通过转换顺序`soduku[j][i]`，这样可以同时访问一行、一列。
* 可以通过 `(i / 3) 和 (j / 3)`得到一个`3x3`小方格，同时通过`(i % 3) * 3 + j % 3`计算出`sudoku[i][j]`在小方格中的位置。

通过以上两个规律，我们就可以在`n * n`次循环中得出是否合法的结果。

```Java
class Solution {
    public boolean isValidSudoku(char[][] board) {
    
        //实现的比较挫，这里定了9个数组用来代表9个 3x3的小方格
        char[] rect0 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect1 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect2 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect3 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect4 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect5 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect6 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect7 = {'.','.','.','.','.','.','.','.','.'};
        char[] rect8 = {'.','.','.','.','.','.','.','.','.'};

        Map<String, char[]> maps = new HashMap<>();
        maps.put("0_0", rect0);
        maps.put("0_1", rect1);
        maps.put("0_2", rect2);
        maps.put("1_0", rect3);
        maps.put("1_1", rect4);
        maps.put("1_2", rect5);
        maps.put("2_0", rect6);
        maps.put("2_1", rect7);
        maps.put("2_2", rect8);

        for (int i = 0; i < 9; i++) {
            char[] hor_array = {'.','.','.','.','.','.','.','.','.'};
            char[] ver_array = {'.','.','.','.','.','.','.','.','.'};
            for (int j = 0; j < 9; j++) {
                //行
                char hor_char = board[i][j];
                //列
                char ver_char = board[j][i];

                //如果第一次出现，则把对应的位置替换为对应的内容
                //如果第二次出现，那么内容就不是初始值了，可以认定是invalid
                if(hor_char != '.'){
                    if(hor_array[hor_char - '0'] == '.'){
                        hor_array[hor_char - '0'] = hor_char;
                    }else {
                        return false;
                    }

                    //计算当前位置所属的小方格
                    String key = (i / 3) + "_" + (j / 3);
                    //int index = (i % 3) * 3 + j % 3;
                    char[] rect = maps.get(key);
                    if (rect[hor_char - '0'] == '.') {
                        rect[hor_char - '0'] = hor_char;
                    }else {
                        return false;
                    }
                }

                if(ver_char != '.'){
                    if(ver_array[ver_char - 49] == '.'){
                        ver_array[ver_char - 49] = ver_char;
                    }else {
                        return false;
                    }
                }
            }
        }

        return true;
    }
}
```

实现思路上算是正确的，但不是最优的哈。最优代码参考:

```Java
    /**
     * 最快的版本。 10ms  beats 100%。
     * 对于每个横、竖和3x3 进行 `|` 操作，这样通过 `&`操作就能知道这个数字是否重复过。
     * @param board
     * @return
     */
    public boolean isValidSudokuFastest(char[][] board) {

        int [] vset = new int [9];
        int [] hset = new int [9];
        int [] bckt = new int [9];
        int idx = 0;
        for (int i = 0; i < 9; i++) {
            for (int j = 0; j < 9; j++) {
                if (board[i][j] != '.') {
                    idx = 1 << (board[i][j] - '0') ;
                    if ((hset[i] & idx) > 0 ||
                            (vset[j] & idx) > 0 ||
                            (bckt[(i / 3) * 3 + j / 3] & idx) > 0) return false;
                    hset[i] |= idx;
                    vset[j] |= idx;
                    bckt[(i / 3) * 3 + j / 3] |= idx;
                }
            }
        }
        return true;

    }
```

## Review

本周Review [A Technique for Deciding When to Say No](https://medium.com/s/story/the-law-of-two-thirds-cfad7c4d42eb)

`人不能完成所有事情，所以我们需要对要做的事情，想要的东西做取舍。当你选择某些项的时候，也就舍弃了其他的`

文章中提出了一个`2-3`法则，就是说在3个关键要素中，你只能同时选择两个。比如：质量、速度和价格，只能选择两个。如果你想要质量和速度，那么价格必然就贵；你想要低价同时也想保持质量，那么速度必然受到影响；你想要低价和速度，那么质量就无法保证。

这个观点和我们程序员熟悉的`CAP`理论有异曲同工之妙，同样的三个要素，不可兼得。

那么怎么样来进行选择呢？那么首先要弄明白的是你想要的到底是什么？这里要记住：`你是不可能做所有事情，得到所有东西的`。

对于生活，一定不要让自己时时处于忙碌的状态，忙碌意味着你不知道你想要的是什么，你不知道对什么说**不**，所以你的时间被无关的事情给占有了。

## Tip

**在数据量大的情况下，任何系统的性能隐患都被放大直到系统崩溃。**

在本周有一个算法同学写了一段类似如下的大数据处理SQL：

```SQL
select col1, col2, process(col4, map('{1,2,3,4,5}')) from table_a
```

SQL 很简单，`process` 和 `map`都是自定的函数。其中`map`函数会将参数转化成一个map结构。但是在实际运行的时候发现性能很差，100亿条数据执行了9个多小时。他找我帮他看看问题在哪里？

问题其实就是`map`函数导致的，这个sql每处理一条数据就会调用`map`函数，就会创建一个新对象。同时因为参数比较长，会多次触发map的扩容，导致整个SQL执行99%以上的时间其实在创建map和扩容中。

当修改实现改成使用缓存以后，性能溜得飞起，只要了**4分钟**就完成了。

在大数据环境下，对象的创建其实是很昂贵的操作。所以像`Hadoop`这样的大数据框架才会设计成对象复用。

## Share

[GC算法-引用计数法](http://linuxlsx.top/blog/2019/03/26/la-ji-hui-shou-de-suan-fa-yu-shi-xian-du-shu-bi-ji-yin-yong-ji-shu-fa/)
