---
title: 给Hexo + Github pages搭建的个人博客绑定域名
tags: [ web ]
keywords: [ web ]
date: 2019-12-21 12:56:15
categories:
- 软件技术
index_img: /img/blog/github-pages-bind-domain-name-solutions/banner.jpeg
banner_img: /img/blog/github-pages-bind-domain-name-solutions/banner.jpeg
---

最近给博客绑定了域名，趁着还记得一些细节的时候把关键点都写下来 : )
<!-- more -->
# 给Hexo + Github pages搭建的个人博客绑定域名

前段时间使用[Hexo](https://github.com/hexojs/hexo)给自己搭了一个博客，托管在Github上面。不过托管在Github上的博客对外暴露的是`github.io`的子域名，最近我又买了一个独立的域名绑定在这个博客上，看上去更顺眼了一些 ; )

给自己的博客绑定域名有什么好处呢？个人认为最大的好处应该是让我们的博客在网上有一个独立的入口了，这个入口不依赖于其他的实现。这个类似于面向对象编程里面接口和实现的关系。域名就是接口，用户通过域名访问我们的博客时可以不用在意我们把博客托管在哪个服务器上，而我们在Github pages上托管博客的域名，如`yhs0092.github.io`，就是具体的实现。  
假设我们觉得从国内访问Github pages太慢了，还可以在国内另外找个服务器托管一份博客，可以把国内访问者的流量导流到国内的这个服务器上，加速访问过程。如果我们想把自己的博客迁移到另外一个地方了，或者某个服务器坏掉了，我们也可以很方便地做服务器地址切换。这些操作对于访问者来说都是无感知的。  
除此之外，听说有个独立的域名对于搜索引擎爬取我们的博客内容也有好处，不过这个也取决于你买到的域名好不好。

## 购买域名

选择域名也是一个技术活，建议大家在下单购买自己中意的域名之前，先去检查一下这个域名之前有没有被人使用过，以及这个域名是否被各个搜索引擎ban掉了，这里的门道比较多，我刚刚开始接触，也是在网上搜资料看的，这里就不多说了，建议大家直接搜资料看看 ; )

这里是在华为云上购买的域名，进入华为云的域名注册服务，可以选一个中意的域名，先搜一下看看哪些后缀的域名还没有被占用。例如搜索一下“seatime”域名，可以看到部分顶级域名下的“seatime”二级域名已经被注册了。
![](/img/blog/github-pages-bind-domain-name-solutions/search_for_available_domainnames.jpg)
于是只能在还没有人使用的域名里选购一个了（先检查一下这个域名有没有被ban过！）。假设我们购买的是`seatime.online`，将其加入清单，然后购买，这样域名就到手了。

**注意**： 在国内购买域名需要实名认证才能使用

## 配置域名解析记录

买到的域名还需要配置一下才能指向我们的Github pages地址。从华为云DNS服务的公网域名列表中找到刚刚购买的域名，点击右边的“解析”，进入配置页。
![](/img/blog/github-pages-bind-domain-name-solutions/domain_name_config_entry.jpg)
点击右上角的“添加记录集”，将之前搭建的Github pages地址填进去，如下图所示。注意这里选择的类型是`CNAME`，不同类型的记录功能各不相同，有兴趣的同学可以去搜一些资料了解一下。
![](/img/blog/github-pages-bind-domain-name-solutions/add_dns_record.jpg)
然后等待一段时间，你就能通过`seatime.online`解析到`yhs0092.github.io`的地址了。

## 配置CNAME文件

DNS配置生效后，新购买的域名就已经指向Github pages页面地址了。但此时如果去访问这个域名的话，得到的是Github的404返回页，这是因为我们还没有在Github pages中配置域名。我们还需要在工程的根目录下放置一份CNAME文件指向我们的域名。
这里说的“根目录”是指的托管在Github pages上的静态页面工程的根目录，而对于hexo源工程而言，CNAME文件应该放在`source/`目录下，文件内容就是我们的域名，如下：
```
$ cat source/CNAME
seatime.online
```
然后执行 `hexo clean && hexo g && hexo d` 完成构建和部署就行了。
