---
layout: post
title: Nginx(RestHub)故障分析
---

故障日期: 2016年08月15日

RestHub: 基于Nginx+lua实现的一个API网关产品


# 一. 故障发现
系统报警，支付服务某个接口有两次调用失败，错误码为502

```
502 BAD GATEWAY

The server, while acting as a gateway or proxy, received an invalid response from an inbound server it accessed while attempting to fulfill the request.

--摘自https://httpstatuses.com/502
```

# 二. 第一反应
根据以往的经验，502错误大多数时候是由于后端服务器拒绝连接引起的，常见的情况是服务没有启动(没有监听服务端口)。

于是，马上和开发、运维的同学询问支付系统是否做过升级，答案是没有，而且，运维同学同时也查看了支付的后端服务(Tomcat)并没有发现异常，也没有发现重启。

这个时候，基本可以断定是RestHub出了故障。

```
我们参加过一个项目的开发，有位高级工程师确信select系统调用在Solaris上有问题。再多的劝说或逻辑也无法改变他的想法（这台机器上的所有其他网络应用都工作良好这一事实也一样无济于事）。他花了数周时间编写绕开这一问题的代码，因为某种奇怪的原因，却好像并没有解决问题。当最后被迫坐下来、阅读关于select的文档时，他在几分钟之内就发现并纠正了问题。现在每当有人开始因为很可能是我们自己的故障而抱怨系统时，我们就会使用“select没有问题”作为温和的提醒。

--摘自《程序员修炼之道》
```

这本读过我读过很多遍，显然，我又忘了！！！



# 三. Resthub排障
RestHub是基于Nginx+lua实现的，所以错误日志就在Nginx自己的日志文件里，如果你熟悉Nginx的话，那么这里的日志格式应该很容易读懂。

## 日志内容
查看Resthub日志，发现确实支付业务的一个接口调用两次失败

```
2016/08/15 20:34:16 [error] 20486#0: *4076751 connect() failed (111: Connection refused) while connecting to upstream, client: 202.106.222.82, server: localhost, request: "POST /pay/v1/collect HTTP/1.0", subrequest: "http://Pay/api/v1/collect.htm", upstream: "http://127.0.53.53:80/api/v1/collect.htm?__v=v1&__trace_id=10.19.33.237-10982-20486-4076751-1-1471264456.065&__real_ip=202.106.222.82", host: "apis.qianbao.com"


2016/08/15 20:34:16 [error] 11174#0: *4076719 connect() failed (111: Connection refused) while connecting to upstream, client: 202.106.222.82, server: localhost, request: "POST /pay/v1/collect HTTP/1.0", subrequest: "http://Pay/api/v1/collect.htm", upstream: "http://127.0.53.53:80/api/v1/collect.htm?__v=v1&__trace_id=10.19.33.238-10982-11174-4076719-1-1471264456.799&__real_ip=202.106.222.82", host: "apis.qianbao.com"

```

## 最明显错误信息

下面的错误信息也验证了，我们对于502错误码的判断。

```
connect() failed (111: Connection refused) 
```

但是，根据运维同学的信息，后端服务并没有出现故障，所以Connection refused的动作应该不是后端服务的行为，再仔细查看日志，发现一点端倪

```
subrequest: "http://Pay/api/v1/collect.htm", upstream: "http://127.0.53.53:80/api/v1/collect.htm
```

Pay是支付服务的upstream的名字，为什么Nginx把他解析(resolv)到127.0.53.53, why?，这明显不科学嘛！
好吧，事情讲到现在，需要简单说一下RestHub关于Upstream的设计思路啦，要不然你就晕啦


## RestHub使用upstream的方式
这里涉及到数据库中的两个表

* `api_version` 保存api的接口信息，包括请求方法，路径，upstream名称
* `upstream` 保存upstream的服务器配置

那么，这时候带来3个问题

1. 当upstream表更新时，需要通知resthub重新生成upstream的配置文件
2. 当api_version表更新时，需要通知resthub重新刷新api_data的缓存
3. 当upstream更新时，需要通知resthub重新刷新api_data的缓存

但是，实际代码中，忽略了第三点。


## 故障是怎么被触发的

按照支付同学的要求，支付服务的upstream名称由Pay改为pay_biz，结果，resthub重新生成upstream的配置文件，但是没有更新api_data(api_data中还在引用称为Pay的upstream名字)，从日志中可以印证一下

```
subrequest: "http://Pay/api/v1/collect.htm", upstream: "http://127.0.53.53:80/api/v1/collect.htm
```

这样的话，日志中出现Pay就可以解释通啦。我们再回到前面一个问题来，为什么Pay会解析到127.0.53.53


## resthub的配置文件
这是resthub的转发逻辑的关键配置

```
location ~ ^http {
            resolver 192.168.1.231;
            internal;
            proxy_pass $uri$is_args$args;
}
```

从现象上看，Nginx找不到名称为Pay的upstream，就去DNS服务器去解析，下面是Nginx官方文档

```
proxy_pass http://$host$request_uri;

In this case, the server name is searched among the described server groups, and, if not found, is determined using a resolver.
```

[点击这里查看原文](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)


## 最后一个问题
为什么pay会解析到127.0.53.53呢？

我们先来做个实验

```
[root@DEV-161 ~] $ ping pay
PING pay (127.0.53.53) 56(84) bytes of data.
64 bytes from 127.0.53.53: icmp_seq=1 ttl=64 time=0.030 ms
64 bytes from 127.0.53.53: icmp_seq=2 ttl=64 time=0.038 ms
64 bytes from 127.0.53.53: icmp_seq=3 ttl=64 time=0.034 ms
^C
--- pay ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2027ms
rtt min/avg/max/mdev = 0.030/0.034/0.038/0.003 ms
```

确实，pay被解析到了。好吧，是时候求助google啦。下面是google告诉我的

```
ICANN Approves Name Collision Occurrence Management Framework.

Special IP Address (127.0.53.53) Alerts System Administrators of Potential Issue
```
[点击这里查看原文](https://www.icann.org/news/announcement-2-2014-08-01-en)

大致意思就是，pay之前是顶级域名，但是现在已经废弃啦，各位管理员该注意啦。除了pay之外，还有很多其他以常见单词命名的顶级域名。

[点击这里查看更多](https://www.icann.org/sites/default/files/ci-monitoring/citld-complete.csv)

# 启示
为了避免误用被废弃的顶级域名(以单词命名)，upstream的名称最好加上项目名称，或者简单一点就是把名字变的更长一些，这样就能避开一些不容易发现的巨坑。
