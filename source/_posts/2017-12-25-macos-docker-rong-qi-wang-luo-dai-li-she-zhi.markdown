---
layout: post
title: "macOS Docker 容器网络代理设置"
date: 2017-12-25 10:47:46 +0800
comments: true
categories: [docker]
---

最近公司中绝大部分的容器都已经切换到Docker中了，所以准备深入学习下Docker 相关的内容。

在[Docker官网](https://www.docker.com/community-edition)下载了最新的Mac版本，然后按照[第一本Docker书](https://www.amazon.cn/dp/B015QSNIY0/ref=sr_1_2?s=books&ie=UTF8&qid=1514114219&sr=1-2&keywords=%E7%AC%AC%E4%B8%80%E6%9C%ACdocker%E4%B9%A6) 中逐步学习。不过这本书是基于Docker 1.9 版本编写，最新的Docker Mac 版本已经有了不少的变化, 不过基本的命令还是没有变化的。

<!-- more -->

按照书本上的例子，我下载了 ubuntu 的 docker镜像。按照 `docker run -i -t ubuntu /bin/bash` 的命令进入到容器中。按照示例接下来需要执行`apt-get -yqq update` 来更新元数据。但是在默认的配置下出现了意外的网络问题。

```
root@3511c8b519ea:/# apt-get -yqq update
W: The repository 'http://security.ubuntu.com/ubuntu xenial-security Release' does not have a Release file.
W: The repository 'http://archive.ubuntu.com/ubuntu xenial Release' does not have a Release file.
W: The repository 'http://archive.ubuntu.com/ubuntu xenial-updates Release' does not have a Release file.
W: The repository 'http://archive.ubuntu.com/ubuntu xenial-backports Release' does not have a Release file.
E: Failed to fetch http://security.ubuntu.com/ubuntu/dists/xenial-security/universe/source/Sources  Cannot initiate the connection to 6152:80 (0.0.24.8). - connect (22: Invalid argument)
E: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/xenial/universe/source/Sources  Cannot initiate the connection to 6152:80 (0.0.24.8). - connect (22: Invalid argument)
E: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/xenial-updates/universe/source/Sources  Cannot initiate the connection to 6152:80 (0.0.24.8). - connect (22: Invalid argument)
E: Failed to fetch http://archive.ubuntu.com/ubuntu/dists/xenial-backports/main/binary-amd64/Packages  Cannot initiate the connection to 6152:80 (0.0.24.8). - connect (22: Invalid argument)
E: Some index files failed to download. They have been ignored, or old ones used instead.
```

在网上搜索了相关资料后，发现是网络代理设置的问题：

```
root@3511c8b519ea:/# set | grep proxy
http_proxy=docker.for.mac.localhost:6152
https_proxy=docker.for.mac.localhost:6152
```

`docker.for.mac.localhost` 是从 **17.06** 版本开始引入的一个Mac DNS 名称，会被解析成外部主机的IP。6152 是我设置的代理端口。

要想代理能够生效需要增加一个 **http** 的前缀：

```
root@1e9af09f86a4:/# set | grep proxy
http_proxy=http://docker.for.mac.localhost:6152
https_proxy=http://docker.for.mac.localhost:6152
```

或者直接设置不需要使用代理：

```
root@1e9af09f86a4:/# set | grep proxy
http_proxy=no_proxy
https_proxy=no_proxy
```

这样设置完了以后，就可以更新 **apt-get** 的元数据，然后就可以安装想要的工具了。


#### 参考资料

1. [docker apt-get got `Cannot inititate the connection to 8000:80 (…) - connect (22: Invalid Argument)`](https://stackoverflow.com/questions/46632967/docker-apt-get-got-cannot-inititate-the-connection-to-800080-connect)
2. [Networking features in Docker for Mac](https://docs.docker.com/docker-for-mac/networking/#i-cannot-ping-my-containers)



