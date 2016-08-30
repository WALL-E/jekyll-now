---
layout: post
title: 关于HTTP Restful的几个真相
---

```
一切阅读都是误读。
```
推荐阅读《Architectural Styles and
the Design of Network-based Software Architectures》，如果你已经读过这篇文章，下面的内容就不用看啦。

## Http的缩写
我们先看看维基百科上的解释

* 英文
    
    [Hypertext Transfer Protocol](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) 
    
* 中文

    [超文本传输协议](https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)
    

我们再看一下单词**Transfer**的中文翻译

```
transfer
英 [trænsˈfɜ:(r)]  美 [trænsˈfɚ] 
vt.
使转移;使调动;转让（权利等）;让与
vi.
转让;转学;转乘;转会（尤指职业足球队）
n.
转移;调动;换乘;（运动员）转会
```

很容易发现，单词**Transfer**根本没有**传输**的含义，维基百科中文版的翻译完全是错误的。如果这个还没有足够的说服力，那么，我们看看Fielding博士的论文《Architectural Styles and
the Design of Network-based Software Architectures》，文章中专门提到，“HTTP 不是一种传输协议”。[点击这里查看原文](http://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm), 下面是章节节选

```
6.5.3 HTTP is not a Transport Protocol

HTTP is not designed to be a transport protocol. It is a transfer protocol in which the messages reflect the semantics of the Web architecture by performing actions on resources through the transfer and manipulation of representations of those resources. It is possible to achieve a wide range of functionality using this very simple interface, but following the interface is required in order for HTTP semantics to remain visible to intermediaries.

```

**注**：*真是不幸, HTTP协议刚刚传入我国时, 即被翻译为“超文本传输协议”, 显然是错误的，之后以讹传讹就更是贻害无穷。*


## 先有Http，后有Restful？
想要知道Restful是什么东东，最好的办法是了解一下它的发展历史。从Ruby on Rails的发展历程可以发现一些踪迹。

Ruby on Rails 1.1: ActionWebService

```
在笔者看来，Fielding这篇博士论文在Web发展史上的价值，不亚于Web之父Tim Berners-Lee关于超文本的那篇经典论文。然而遗憾的是，这篇博士论文在诞生之后的将近5年时间里，一直没有得到足够的重视。例如Web Service相关规范SOAP/WSDL的设计者们，显然不大理解REST是什么，HTTP/1.1究竟是一个什么样的协议、为何要设计成这个样子。

摘自 http://www.infoq.com/cn/articles/understanding-restful-style/
```


Ruby on Rails 1.2: ActionResource

```
直到2005年，随着Ajax、Rails等Web开发技术的兴起，在Web开发技术社区掀起了一场重归Web架构设计本源的运动，REST架构风格得到了越来越多的关注。在2007年1月，支持REST开发的Ruby on Rails 1.2版正式发布，并且将支持REST开发作为Rails未来发展中的优先内容。Ruby on Rails的创始人DHH做了一个名为“World of Resources”的精彩演讲，DHH在Web开发技术社区中的强大影响力，使得REST一下子处在Web开发技术舞台的聚光灯之下。

摘自 https://blackanger.gitbooks.io/tao-of-chef/content/chapter_5_rails/restful.html
```

Ruby on Rails 现状

```
稍早的版本的Rails中提供了ActionWebService作为开发XML-RPC和SOAP的web服务的基础。但是最近的Rails 1.2更加倾向于是用REST方式的web服务，而ActionWebService在Rails 2.0中作为plugin而不再是rails核心的一部分。
```


**注**：*大部分人印象中，都觉着restful值是http协议的一种应用风格，原因可能是大家先接触到的http，而后才了解到restful。很明显，这是错误的概念，实际上，正式由于有rest架构的方法论指导着http协议规范的制定和发展，Web才有了今天巨大的成就*

## Restful不是全部
与REST架构风格并行的还有几种架构风格

* 分布式对象（Distributed Objects，简称DO） 架构实例有CORBA/RMI/EJB/DCOM/.NET Remoting等等
* 远程过程调用（Remote Procedure Call，简称RPC） 架构实例有SOAP/XML-RPC/Hessian/Flash AMF/DWR等等

## 时间线
1. 1991年，发布http0.9
2. REST的第一版开发于 1994 年 10 月和 1995 年 8 月之间
3. 1996年，发布http1.0
4. 2000年，发布http1.1，发表论文《Architectural Styles and
the Design of Network-based Software Architectures》
5. 2014年，发布http2.0
