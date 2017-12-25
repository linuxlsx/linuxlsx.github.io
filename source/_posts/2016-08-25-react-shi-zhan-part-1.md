---
layout: post
title: "React 实战Part 1"
date: 2016-08-25 21:00:08 +0800
comments: true
categories: React
---


React 是由FB开发的一款高性能的前端渲染框架。我们在公司的内部项目中也开始采用React 来替换原有的开发框架。由于前端资源不足，所以自告奋勇提出由自己来完成一部分的前端工作，从此开始接触到React.

<!-- more -->

由于我也是刚接触到React, 所以这一篇不讲(其实是讲不出来)什么原理性的东西，只是把我这几天用React 碰到的一些问题给记录下。

首先是推荐一款优秀的前端组件库: `antd`。是由蚂蚁金服的同学发起并维护的。[官网地址](http://ant.design/)。整体设计非常简洁，模块化，用起来非常容易上手。

碰到的第一个问题其实也跟 `antd` 相关，在表单中使用了表单的 `getFieldProps` 属性。刚开始也是瞎蒙的，看着示例代码就开搞。在使用了这个属性的同时又设置了 onChange 属性。 如下:

    <Select {...getFieldProps('componentId', {
        initialValue: (PageData.create && PageData.create.id) ? PageData.create.componentId : '',
        rules: [
            {required: true, message: '请选择组件实例分类'},
            ],
              )} onChange={this.handleComponentChange}>
                {
                  this.props.componentDatas.map((o, n)=>{return (
                    <Option value={'' + o.id} key={n}>{o.name}</Option>
                  )})
                }
    </Select>

测试过程中碰到的问题就是死活无法选中下拉框中的选项，但是事件又触发了。去掉`onChange` 属性能够选中选项，但没法处理事件。检查文档发现 `getFieldProps` 已经包含了`id` `value` `ref` `onChange` 属性，不能重复设置属性。 所以正确的做法是在 `getFieldProps` 中指定时间处理函数。



第二个问题跟CodeMirror相关。项目中需要编辑代码，自然想到就是将CodeMirror引入到项目中来。搜索后发现[react-codemirror](https://github.com/JedWatson/react-codemirror) 还不错，就引入到项目中。第一次尝试很成功一次就过，但是当同一个页面中出现多个CodeMirror 的实例时会出现 **状态变化后, 只有最底下的一个实例能够响应状态的变更**。这个显然不是我想要的结果。GitHub 项目中的 issue 是个好东西，你碰到过的问题肯定有其他人也碰到过。该项目的
 [issue #46](https://github.com/JedWatson/react-codemirror/issues/46) [issue #47](https://github.com/JedWatson/react-codemirror/issues/47) [issue #49](https://github.com/JedWatson/react-codemirror/issues/49)都提到了这个问题。 这个问题的原因就是 `componentWillReceiveProps` 这个方法被 `debounce` 后变成异步执行的了，所有只会有最后一个实例才会收到状态变更的消息。

 解决办法就是将 `componentWillReceiveProps` 改为同步执行的。目前作者还没有修复这个问题，只能在代码中通过重写这个方法来完成。具体的实现参见 [解决方案](https://github.com/mozilla-services/react-jsonschema-form/pull/175)


两个问题都不是什么大问题，但是从发现问题到解决问题还是花费了我不少的时间。三个解决问题的经验:

* 查看文档。文档中通常都会详细的说明一些需要特殊注意的场景，而这些场景往往就是出问题的地方。
* 如果是开源项目，那就先检查 issues 中是否已经有过类似的问题了。看是不是可以通过升级版本解决问题。
* 再不行的话只能通过万能的 Google 了，前提是要能科学上网。    
