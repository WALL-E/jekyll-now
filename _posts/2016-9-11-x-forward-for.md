---
layout: post
title: X-Forwarded-For缺陷与陷阱
---

# 一、定义
X-Forwarded-For:简称XFF头，它代表客户端，也就是HTTP的请求端真实的IP，只有在通过了HTTP 代理或者负载均衡服务器时才会添加该项。它不是RFC中定义的标准请求头信息，在squid缓存代理服务器开发文档中可以找到该项的详细介绍。

# 二、起源
X-Forwarded-For(XFF)是用来识别通过HTTP代理或负载均衡方式连接到Web服务器的客户端最原始的IP地址的HTTP请求头字段。 Squid 缓存代理服务器的开发人员最早引入了这一HTTP头字段，并由IETF在Forwarded-For HTTP头字段标准化草案中正式提出。

# 三、格式
这一HTTP头一般格式如下:
X-Forwarded-For: client1, proxy1, proxy2, proxy3

其中的值通过一个 逗号+空格 把多个IP地址区分开, 最左边(client1)是最原始客户端的IP地址, 代理服务器每成功收到一个请求，就把请求来源IP地址添加到右边。 在上面这个例子中，这个请求成功通过了3台代理服务器：proxy1, proxy2 及 proxy3。请求由client1发出，到达了proxy3(proxy3可能是请求的终点)。请求刚从client1中发出时，XFF是空的，请求被发往proxy1；通过proxy1的时候，client1被添加到XFF中，之后请求被发往proxy2;通过proxy2的时候，proxy1被添加到XFF中，之后请求被发往proxy3；通过proxy3时，proxy2被添加到XFF中，之后请求的的去向不明，如果proxy3不是请求终点，请求会被继续转发。

# 四、来源IP可信吗？
因为伪造这一字段非常容易，所以应该谨慎使用X-Forwarded-For字段。正常情况下XFF中最后一个IP地址是最后一个代理服务器的IP地址, 这通常是一个比较可靠的信息来源。下面会详细说明，在什么情况下，这个信息并不准确。

## 1. 客户端设置代理
如果客户端使用了http代理(一般大公司内部上网都需要使用代理)，同时，这个代理并没有正确设置(也可能是故意不设置)XFF，那么，XFF中client1的位置就会被http代理的ip取代，那么我们就无法正确获取客户端的真实IP。

## 2. 网络NAT设备
为了减缓可用的IP地址空间的枯竭， 大多数ISP网络或者公司网络中都会使用NAT设备，这样，一个公司或者一个小区的用户就可以共享使用一个IP来实现上网需求。NAT设备工作在IP层和TCP层，所以XFF不管怎么设置也无法解决这个问题。
这种情况下，我们也无法拿到客户的真实IP。

### 查看真实IP
Windows：

```
无线局域网适配器 无线网络连接:

   连接特定的 DNS 后缀 . . . . . . . : lan
   描述. . . . . . . . . . . . . . . : Intel(R) Centrino(R) Wireless-N 1000
   物理地址. . . . . . . . . . . . . : 00-1E-64-91-DF-00
   DHCP 已启用 . . . . . . . . . . . : 是
   自动配置已启用. . . . . . . . . . : 是
   本地链接 IPv6 地址. . . . . . . . : fe80::7d79:4d1d:58e6:32cc%12(首选)
   IPv4 地址 . . . . . . . . . . . . : 192.168.1.186(首选)
   子网掩码  . . . . . . . . . . . . : 255.255.255.0
   获得租约的时间  . . . . . . . . . : 2016年9月9日 20:45:08
   租约过期的时间  . . . . . . . . . : 2016年9月12日 0:01:30
   默认网关. . . . . . . . . . . . . : 192.168.1.1
   DHCP 服务器 . . . . . . . . . . . : 192.168.1.1
   DHCPv6 IAID . . . . . . . . . . . : 201334372
   DHCPv6 客户端 DUID  . . . . . . . : 00-01-00-01-1D-69-C7-A1-00-27-13-65-48-AE

   DNS 服务器  . . . . . . . . . . . : 8.8.8.8
   TCPIP 上的 NetBIOS  . . . . . . . : 已启用

以太网适配器 本地连接:

   媒体状态  . . . . . . . . . . . . : 媒体已断开
   连接特定的 DNS 后缀 . . . . . . . :
   描述. . . . . . . . . . . . . . . : Intel(R) 82567LF Gigabit Network Connecti
on
   物理地址. . . . . . . . . . . . . : 00-27-13-65-48-AE
   DHCP 已启用 . . . . . . . . . . . : 是
   自动配置已启用. . . . . . . . . . : 是
```

Linux

```
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eno16777728: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:42:15:de brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.248/24 brd 192.168.1.255 scope global dynamic eno16777728
       valid_lft 38029sec preferred_lft 38029sec
    inet6 fe80::20c:29ff:fe42:15de/64 scope link 
       valid_lft forever preferred_lft forever
```
### 查看自己的公网IP
访问以下两个网址，返回的网页会告诉你，你的公网IP是多少

* http://www.ip138.com
* http://www.ip.cn

可以用curl访问网址，举个例子

```
[root@localhost ~]# curl http://www.ip.cn
当前 IP：223.72.90.206 来自：北京市 移动
```

结果，我的真实ip是192.168.1.186(私有IP)，但是在XFF中会显示我的IP为223.72.90.206。

## 3. 后端服务购买安全服务
有的公司使用[加速乐](https://www.yunaq.com/), 我们可以认为使用这种服务后，相当于增加了一层http代理。

### DNS是否被“主动劫持”
我们可以查看一下域名是否使用了加速乐(查看其他的安全服务也是类似的方法)

```
[root@localhost ~]# dig www.XXXXXX.com

; <<>> DiG 9.9.4-RedHat-9.9.4-29.el7_2.3 <<>> www.51kahui.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31138
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.51kahui.com.		IN	A

;; ANSWER SECTION:
www.XXXXXX.com.	600	IN	CNAME	a29dXXXXXXa8f83b.cdn.jiashule.com.
a29dXXXXXXa8f83b.cdn.jiashule.com. 4 IN	A	111.13.147.215
a29dXXXXXXa8f83b.cdn.jiashule.com. 4 IN	A	111.13.147.195
```

结果，XFF中有一层proxy就是加速乐的IP，由于加速乐有很多节点，所以这个IP地址新并不是固定的。内部消息显示加速乐节点的IP要达到上万个，尽管我太想相信。但是这至少说明一点，就是XFF中proxy的IP地址是多变的。

### 是否被动注入安全服务
最近测试发现，使用电信4G网络的时候，XFF会莫名其妙的加入多个proxy代理，但是这个IP地址实在是无法查证，我们姑且认为他们也是一种安全服务吧

## 4. 攻击者
攻击者可以很容易伪造XFF，在发出请求时在XFF值中插入多个IP地址，以混淆服务端获取到客户端的真实IP。

**结论，从XFF中的获取到的客户端IP地址并不可信，在使用各种ACL策略的时候，需要读者权衡利弊来满足业务需求。**

# 五、Nginx中获取客户端IP的正确姿势
nginx目前在各个互联网公司中的标配, 其中ngx_http_realip_module模块就提供从XFF中实现获取用户的真实IP的功能。

## 配置方式
```
set_real_ip_from 10.0.0.0/8;
set_real_ip_from 192.168.1.0/24;
real_ip_header X-Forwarded-For;
real_ip_recursive on;

proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
```

但是，上面的配置有一些问题，你发现什么问题了吗？

## 问题所在
首先，我们解释一下上面最后一条配置语句的含义
```
proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
```

实际上的含义是

```
proxy_set_header  X-Forwarded-For '$X-Forwarded-For， $remote_addr';
```

我们可以看到，$proxy_add_x_forwarded_for等于$X-Forwarded-For加上$remote_addr。但是有一个问题，realip模块要比proxy模块运行的早，在realip模块中会把remote_addr重置为客户端IP，这样会导致 X-Forwarded-For设置错误。

我们拿proxy3为例，正常情况下，XFF传到proxy4服务时，XFF的值是
```
client1, proxy1,proxy2
```

如果按照上面的配置的话，XFF值会是
```
client1, proxy1,client1
```

如果内部有两级Nginx的话，XFF值会是
```
client1, proxy1,client1，cleint1
```

总之，只要是XFF出现的重复的IP地址，就有理由怀疑是Nginx配置引起的。

下面是才是realip的正确配置

```
set_real_ip_from 10.0.0.0/8;
set_real_ip_from 192.168.1.0/24;
real_ip_header X-Forwarded-For;
real_ip_recursive on;

proxy_set_header  X-Forwarded-For '$X-Forwarded-For, $origin_remote_addr';
```

$origin_remote_addr是$remote_addr的副本信息，当realip模块修改了$remote_addr的值时，$origin_remote_addr用已保留$remote_addr的原始值。

# 六、从XFF中可以得到哪些确定的信息
假设有这样一个XFF

```
X-Forwarded-For: client1, proxy1, proxy2, proxy3
```

其中，如果proxy2，proxy3是我们已知（可信）代理的话，我们可以确定proxy1的IP地址是可信的。


# 七、参考资料

* [维基百科](https://en.wikipedia.org/wiki/X-Forwarded-For)
* [奋力的蜗牛](http://tianxiamall.blog.163.com/blog/static/2084891122015144744102/)
* [Module ngx_http_realip_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html)
