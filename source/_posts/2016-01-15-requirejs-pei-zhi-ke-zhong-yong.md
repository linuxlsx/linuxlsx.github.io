---
layout: post
title: "requireJs 配置可重用"
date: 2016-01-15 13:53:16 +0800
comments: true
categories: [requireJs, javascript, reuse]
---

最近做项目的时候需要配套一个后台界面供运营人员查看相关数据。因为后台应用没有专职的前端, 所以一般的做法是找一套 bootstrap 的模板, 然后再实现具体的逻辑。一般的 javascript 逻辑都是写个每个页面里面的，没什么重用性。想通过模块的方式来实现重用, 需要有个东西来管理模块之间的依赖, 正好有其他的后台应用使用了require.js, 就照样使用了起来。

<!-- more -->

具体关于 require.js 的使用介绍网上有很多, 我也看了好几篇, 推荐几篇比较靠谱的:

1. JavaScript模块化开发系列. [第一篇](http://www.feeldesignstudio.com/2013/09/javascript-module-pattern-basics/) [第二篇](http://www.feeldesignstudio.com/2013/09/javascript-module-pattern-commonjs/) [第三篇](http://www.feeldesignstudio.com/2013/09/javascript-module-pattern-amd/) [第4篇](http://www.feeldesignstudio.com/2013/09/javascript-module-pattern-requirejs/)
2. 阮一峰的`Javascript模块化编程`系列. [第一篇](http://www.ruanyifeng.com/blog/2012/10/javascript_module.html) [第二篇](http://www.ruanyifeng.com/blog/2012/10/asynchronous_module_definition.html) [第三篇](http://www.ruanyifeng.com/blog/2012/11/require_js.html)

学习完了以后对 require.js 以及 AMD 相关的概念有了一定的了解。也达到了模块化开发的目标。这里主要想说明的一个问题是如何使 require.js 的全局配置能达到重用的效果。

这个是我一个页面的JS模块:

    requirejs.config({
        baseUrl : "/assets",

        shim : {
            "bootstrap" : { "deps" :['jquery'] },
            "dist" :{"deps" :['bootstrap']},
            "paginator" : {"deps" : ['jquery']},
            "menu" :{"deps": ['jquery']}
        },

        paths: {
            jquery: 'plugins/jQuery/jQuery-2.1.4.min',
            bootstrap : 'bootstrap/js/bootstrap.min',
            dist: 'dist/js/app',
            paginator : 'plugins/bootstrap-paginator/bootstrap-paginator',
            menu : 'page/module/menu',
            pager : 'page/module/simplepager',
        }
    });
    require(['jquery', 'bootstrap', 'dist', 'paginator' , 'menu', 'pager'],function($, bootstrap, dist, paginator ,menu, pager){

        $(document).ready(function(){

            var init = function(){
                menu.initMenu();
                pager.initPager();
            }

            init();
        });
      });

`requirejs.config()` 其实是一个全局的配置, 但是按照标准的 requireJs 的引入方法是无法重用这个配置的。 在开始阶段我不得不在每个页面中重复的拷贝这个配置。对于有节操的码农来说这是不可接受的。所以我开始寻找解决重复配置的方法, 终于在`stackoverflow`上找到了答案。[原始的问题链接](http://stackoverflow.com/questions/16067042/requirejs-and-common-config)

解决的主要方法就是分开引入。先引入 require.js, 再引入你的全部配置 config.js, 然后在引入你要执行的模块。

    //1. 首先引入 require.js
    <script src="/assets/require.js"  type="text/javascript" ></script>
    //2. 引入全部的配置文件
    <script src="/assets/config.js" type="text/javascript" ></script>

    <script type="text/javascript">
        //3. 引入你要执行的模块
        require(['/assets/page/feedback.js'])
    </script>

通过以上的配置就能减少代码重复了。另外有文章提到可以通过 require.js 的插件机制来达到效果，但是没有找到可行的解决方案。


##### 参考资料
1. [require.js官网](http://requirejs.org/)
2. [setting-up-requirejs](https://github.com/togakangaroo/Blog/blob/master/setting-up-requirejs.md)
