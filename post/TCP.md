---
title: "TCP三次握手四次挥手"
date: 2021-05-11T15:39:00+08:00
draft: true
toc: true
---
TCP是一个巨复杂的传输协议，想要把每个环节聊清楚的话，需要花费很大的精力，这篇文章主要是想从宏观上聊一聊TCP这件事情，里面每个环节的具体细节希望在以后的文章中来“详谈”。
## TCP的定义
基于不可靠的IP，面向字节流的可靠传输协议（UDP是面向数据报的不可靠传输协议）
## 基于定义TCP要解决哪些问题
由定义可以看到，TCP协议是基于IP协议的，而IP协议本身不可靠，所以TCP要解决ip协议的这些缺陷。
1. 乱序问题
2. 丢包问题
3. 流量控制
## TCP建立链接前需要做哪些事情
1. 应用程序调用dns解析，把域名转成ip
2. 然后要求tcp建立链接
3. tcp封包到ip（后面有机会我们详细聊一聊封包到分用的这个过程）
4. ip查路由表，判断直连非直连，判断到底应该解析哪个ip地址(非直连解析网关)
5. 发送ARP二层广播包，所有网络的主机都会收到，只有一个主机发现我的ip地址正是你要的
6. 它就会回一个ARP的二层单播包，回给发送主机
7. 发送主机有了mac地址和映射就可以发包了，这就是在这两个主机之间进行通讯

## TCP 首部
接下来我们看一下TCP头部包含哪些信息
![TCP-Header-01](../images/TCP-Header-01.jpeg)
总结一下，一个完整的tcp头部，包含如下信息： 
- 源端口，目的端口
- 32bit序号
- 32bit确认序号
- 4bit首部长度+(保留6位：SYN,RST)+16bit窗口大小
- 校验和+16位紧急指针
- 选项
- 数据
解释一下上图中几个非常重要的东西：
- **Sequence Number**是包的序号，用来解决网络包乱序（reordering）问题。
- **Acknowledgement Number**就是ACK——用于确认收到，用来解决不丢包的问题。
- Window又叫Advertised-Window，也就是著名的**滑动窗口**（Sliding Window），用于解决流控。
- **TCP Flag** ，也就是包的类型，主要是用于操控TCP的状态机。

## tcp首部中6个标志，他们中的多个可同时被设置为1
- URG:紧急指针
- ACK:确认序号有效
- PSH:接收方应尽快将这个报文交个应用层
- RST:重间链接
- SYN:同步序号用来发起一个连接
- FIN:发端完成发送任务

## TCP三次握手时序图
{{< mermaid >}}
sequenceDiagram
client ->>server: SYN_SENT:1
Note right of server: LISTEN(listen())、SYN_RCVD
server -->>client: SYN:1,ACK+1
Note left of client: establshed状态
client -->> server: ACK+1 
Note right of server: establshed状态
{{< /mermaid >}}


## TCP四次挥手时序图
{{< mermaid >}}
sequenceDiagram
opt 客户端断开阶段(客户端发起)
client ->> server(listen): FIN M
server(listen) ->> server(listen): CLOSE_WAIT_1状态
server(listen) -->> client: ACK M+1
client ->> client: CLOSE_WAIT_2状态
end

opt 服务端断开阶段(服务端发起)
server(listen) ->> server(listen): LAST_ACK
server(listen) -->> client: FIN N
client ->> client: TIME_WAIT
client ->> server(listen): ACK N+1
server(listen) ->> server(listen): CLOSED
end
{{< /mermaid >}}

##  为什么要有TIME_WAIT状态
通过上图，我们注意到，从TIME_WAIT 到CLOSE状态，又一个超时设置，这个超时时间是2MSL，为什么要有TIME_WAIT，而不是直接CLOSE掉呢？
- 防止延迟的数据段被其他使用相同源地址、源端口、目的地址以及目的端口的 TCP 连接收到；
- 保证 TCP 连接的远程被正确关闭，即等待被动关闭连接的一方收到 FIN 对应的 ACK 消息；









