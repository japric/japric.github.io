---
title: vsomeip的Endpoint
date: 2024-11-04 17:20:00 +0800
categories: [SOME/IP, vsomeip]
tags: [cpp,linux,system]
---

## 1 前言

SOME/IP是由AUTOSAR标准化的一种新兴通信中间件标准。它旨在包含汽车、以太网导向的使用案例所需的所有特性，同时满足车辆资源消耗方面的严格要求。该中间件设计为在一种或多种传输协议（主要是 UDP和TCP）之上提供面向服务的抽象，并提供两种通信模式：请求/响应和发布/订阅。此外，SOME/IP 提供服务发现特性，它可以动态广播不同服务的可用性以及管理对选定事件的订阅。

**vsomeip**是SOME/IP规范的开源实现，作为COVESA项目的一部分设计。

## 2 架构逻辑

一个基本的架构可以表示为如图

![图片](/assets/images/2024/Nov/vsomeip_endpoint_arch.png){:.image--xl}

两个通过以太网链接相互连接的ECU（电子控制单元）。在 Linux 内核之上同时执行不同的应用程序。每个应用程序都有自己的vsomeip库实例。

应用程序在ECU系统内部通过使用local endpoint基于IPC方式进行通信。应用程序在ECU系统之间使用tcp/udp endpoint通过RPC方式进行通信。

## 3 软件逻辑

以`3.4.10`版本的vsomeip代码为例, 这里展开分析`implementation\endpoints`部分的代码实现。

### 3.1 endpoints

![图片](/assets/images/2024/Nov/vsomeip_endpoints.svg){:.image--xl}

endpoint相关类包括了一组接口和实现，用于给应用程序创建本地endpoint和远程endpoint，以**进行IPC和RPC通信**

`endpoint`定义原始的接口，`client_endpoint_impl`和`server_endpoint_impl`是两个基于模板的实现类，并派生出
- 4种本地endpoint: *boost::asio::ip::tcp*类型的`local_tcp_server_endpoint_impl`，`local_tcp_client_endpoint_impl`；*boost::asio::local::stream_protocol*类型的`local_uds_server_endpoint_impl`，`local_uds_client_endpoint_impl`
- 4种远程endpoint: *boost::asio::ip::tcp*类型的`tcp_server_endpoint_impl`，`tcp_client_endpoint_impl`；*boost::asio::ip::udp*类型的`udp_server_endpoint_impl`，`udp_client_endpoint_impl`

`endpoint_host`定义原始的接口
- `endpoint_manager_base`实现`endpoint_host`，通过`create_local()`来创建本地endpoint
- `endpoint_manager_impl`继承`endpoint_manager_base`，通过`create_client_endpoint()`和`create_server_endpoint`来创建远程endpoint


### 3.2 netlink_connector

netlink_connector类依赖netlink通信机制和内核进行通信，用于**监控网卡的状态**，根据传入的单播地址和组播地址监控。

netlink是linux平台上一种IPC机制，主要用于用户态进程与内核进程通信。此外还可以用于用户态进程之间通信，这个使用UDS（unix domain socket）也可以做到，上一节中的应用程序就是基于本地endpoint使用UDS进行通信。


#### 3.2.1 初始化和回调处理

初始化

```c++
void netlink_connector::start() {
	...
	socket_.open(nl_protocol(NETLINK_ROUTE), ec);                   // 协议类型为NETLINK_ROUTE，用于设置和查询路由表等网络核心模块
	...
	socket_.bind(nl_endpoint<nl_protocol>(
                RTMGRP_LINK |                                       // 网卡变动时会触发这个多播组
                RTMGRP_IPV4_IFADDR | RTMGRP_IPV6_IFADDR |           // ipv4/ipv6地址变动时会触发这个多播组
                RTMGRP_IPV4_ROUTE | RTMGRP_IPV6_ROUTE |             // ipv4/ipv6路由变动时会触发这个多播组
                RTMGRP_IPV4_MROUTE | RTMGRP_IPV6_MROUTE), ec);      // 多播路由发生更新时会触发这个多播组
    
    socket_.async_receive( // recv_buffer_ & netlink_connector::receive_cbk
            boost::asio::buffer(&recv_buffer_[0], recv_buffer_size),
            std::bind(
                &netlink_connector::receive_cbk,
                shared_from_this(),
                std::placeholders::_1,
                std::placeholders::_2
            )
        );
```

回调

```c++
void netlink_connector::receive_cbk(boost::system::error_code const &_error,
                 std::size_t _bytes) {
    struct nlmsghdr *nlh = (struct nlmsghdr *)&recv_buffer_[0];
    while ((NLMSG_OK(nlh, len)) && (nlh->nlmsg_type != NLMSG_DONE)) {
            char ifname[IF_NAMESIZE];
            switch (nlh->nlmsg_type) { // 根据多播组内的消息类型分别处理
                case RTM_NEWADDR: {    // IP地址变化
                }
                break;
                case RTM_NEWLINK: {    // 网卡变化
                	// 通知上层handler处理
                    handler_(true, ifname, true);
                    /** VSIP: Network interface "eth001" state changed: up **/
                }
                break;
                case RTM_NEWROUTE: {   // 路由添加
                	check_sd_multicast_route_match(...) {
                        // 通知上层handler处理
                        handler_(false, its_route_name, true);
                        /** VSIP: Route "239.001.0.0/16 if: eth001 gw: n/a" state changed: up **/
                	}
                    
                	// check_sd_multicast_route_match返回true，则通知上层组播准备好了
               	}
               	break;
                case RTM_DELROUTE: {   // 路由删除
                	check_sd_multicast_route_match(...) {
                		...
                	}
                	// check_sd_multicast_route_match返回true，则通知上层组播未准备好
                }
                break;
                ...
            }
}
```

自定义上层handle
```c++
void routing_manager_impl::start() {
	...
	netlink_connector_->register_net_if_changes_handler(
            std::bind(&routing_manager_impl::on_net_interface_or_route_state_changed,
            this, std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));
    ...
}
```


## 参考

1. [Scalable service-Oriented MiddlewarE over IP (SOME/IP)](https://some-ip.com/)
2. [SecureSOMEIP-VTM](https://giorio94.github.io/papers/01-SecureSOMEIP-VTM.pdf)
3. [vsomeip用到的socket](https://blog.csdn.net/CHALLENG_EVERYTHING/article/details/142690435)
3. [VSOMEIP代码阅读整理(1) - 网卡状态监听](https://blog.csdn.net/CHALLENG_EVERYTHING/article/details/142680124)

