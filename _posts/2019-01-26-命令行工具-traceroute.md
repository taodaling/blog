---
categories: tool
layout: post
---

traceroute是一个查看网络报文发送历程的工具。

```sh
$ traceroute google.com

traceroute to google.com (216.58.200.46), 30 hops max, 60 byte packets
 1  172.17.0.1 (172.17.0.1)  0.047 ms  0.014 ms  0.012 ms
 2  192.168.1.1 (192.168.1.1)  1.114 ms  1.036 ms  0.918 ms
 3  122.235.136.1 (122.235.136.1)  36.845 ms  37.767 ms  37.711 ms
 4  61.164.0.205 (61.164.0.205)  6.480 ms  7.152 ms  7.087 ms
 5  61.164.22.105 (61.164.22.105)  7.023 ms  6.967 ms 220.191.157.25 (220.191.157.25)  7.651 ms
 6  202.97.26.9 (202.97.26.9)  8.326 ms 202.97.55.17 (202.97.55.17)  6.657 ms 202.97.26.1 (202.97.26.1)  4.958 ms
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * *
13  * * *
14  * * *
15  * * *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  * * *
28  * * *
29  * * *
30  * * *
```

这样你就可以查看到你的报文经过了哪些路由器的转发。

traceroute的原理是利用ICMP和IP header中的TTL字段。它实际上利用了一个广度优先搜索的方式，先查找经过一次转发抵达的路由，再查找经过两次转发才能抵达的路由，依次查询下去。首先，traceroute送出一个TTL是1的IP数据报文到目的地，在遇到第一个路由器时，路由器会将报文头的TTL字段减少1，此时TTL成了0，该路由器就不再转发报文，但是会回送一个ICMP time exceeded的消息，消息中会包含路由器的IP地址。

至于当数据报文被成功发送到了目的地，由于数据报文中指定的是一个不可用的端口，因此目的地主机会回送一个ICMP port unreachable消息，这样traceroute就可以判别是否抵达目的地了。

traceroute限制了最大的转发数，默认是30，你可以通过-m选项修改。

```sh
$ traceroute -m 10 google.com

traceroute to google.com (216.58.200.46), 30 hops max, 60 byte packets
 1  172.17.0.1 (172.17.0.1)  0.047 ms  0.014 ms  0.012 ms
 2  192.168.1.1 (192.168.1.1)  1.114 ms  1.036 ms  0.918 ms
 3  122.235.136.1 (122.235.136.1)  36.845 ms  37.767 ms  37.711 ms
 4  61.164.0.205 (61.164.0.205)  6.480 ms  7.152 ms  7.087 ms
 5  61.164.22.105 (61.164.22.105)  7.023 ms  6.967 ms 220.191.157.25 (220.191.157.25)  7.651 ms
 6  202.97.26.9 (202.97.26.9)  8.326 ms 202.97.55.17 (202.97.55.17)  6.657 ms 202.97.26.1 (202.97.26.1)  4.958 ms
 7  * * *
 8  * * *
 9  * * *
10  * * *
```

# 参考

- [http://www.cnblogs.com/peida/archive/2013/03/07/2947326.html](http://www.cnblogs.com/peida/archive/2013/03/07/2947326.html)