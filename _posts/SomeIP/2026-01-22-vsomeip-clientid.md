---
title: vsomeip clientid分配逻辑
date: 2026-01-22 19:20:00 +0800
categories: [SOME/IP, vsomeip]
tags: [cpp,linux,system]
---

## 1 前言

vsomeip通信的一个简化逻辑是 vsomeip app1(client) 通过 vsomeip routingmanager host 找到vsomeip app2(server) 最终完成通信

三者之间相互通过clientid这个关键信息联系对方

## 2 vsomeip2上的实例分析


### 2.0 实例基本准备

实例使用vsomeip2(2.14.16)的代码
`https://github.com/japric/vsomeip/tree/vsomeip2`


> note: vsomeip2 最高支持boost1.6.6
> 可以在 https://archives.boost.io/release/1.65.1/source/boost_1_65_1.tar.gz 下载到1.6.5版本


目录结构

```
├── boost_1_65_1/                    # Boost 1.65.1 源码
├── boost_1_65_1_install/            # Boost 1.65.1 编译安装目录
└── vsomeip2/
    ├── config/
    │   └── vsomeip-local.json       #  vsomeip本地配置文件
    ├── examples/
    │   ├── request-sample.cpp       #  request_sample 源码
    │   └── response-sample.cpp      #  response_sample 源码  
    ├── daemon/
    │   └── vsomeipd.cpp             #  vsomeipd 源码
    └── build/
        ├── examples/
        │   ├── request-sample       #  request_sample 可执行文件
        │   └── response-sample      #  response_sample 可执行文件
        └── daemon/
            └── vsomeipd             #  vsomeipd 可执行文件
```

实例中我们需要三个程序
- vsomeipd (vsomeip routing manager host)
- response_sample (vsomeip app2: vsomeip server)
- request_sample (vsomeip app1: vsomeip client)


基本的一些设置和命令
```
export LD_LIBRARY_PATH=/yourpath/boost_1_65_1_install/lib:$LD_LIBRARY_PATH

export VSOMEIP_CONFIGURATION=/yourpath/vsomeip2/config/vsomeip-local.json

VSOMEIP_APPLICATION_NAME=vsomeipd ./yourpath/vsomeip2/build/daemon/vsomeipd &

VSOMEIP_APPLICATION_NAME=service-sample ./yourpath/vsomeip2/build/examples/response_sample &

VSOMEIP_APPLICATION_NAME=client-sample ./yourpath/vsomeip2/build/examples/request_sample &

```


### 2.1 从代码看clientid的分配逻辑


`vsomeip/implementation/utility/src/utility.cpp`
```
开始
  ↓
检查共享内存存在？
  ├─ No → 返回 ILLEGAL_CLIENT
  └─ Yes
        ↓
    加锁保护
        ↓
    [Linux/Unix] 检查 routing manager 进程存活？
        ↓
    路由管理器选举
        ├─ 配置指定且匹配(本进程被指定为routing_manager_host) → 检查 routing_manager_host_ == 0？
        │     ├─ FALSE → 返回 ILLEGAL_CLIENT
        │     └─ TRUE → 还没有进程已经成为routing_manager_host → set_client_as_manager_host = true
        ├─ 未配置且无管理器 → set_client_as_manager_host = true
        └─ 其他情况 → （继续）
            ↓
    容量检查: max_used_client_ids_index_ < max_clients_？
        ├─ No → 返回 ILLEGAL_CLIENT
        └─ Yes
            ↓
    ID 分配策略选择
        ├─ 有预配置 ID 且可用 → 使用预配置 ID
        └─ 其他 → 自动配置算法
                ↓
            确定搜索起点: max_assigned_client_id_ 或 client_base_
                ↓
            循环搜索: while (已使用 || 已配置 || <= max_assigned)
                ├─ 超出范围 → 回绕到 client_base_
                ├─ 搜索次数 > max_clients_ → 返回 ILLEGAL_CLIENT
                └─ 找到可用 ID → 更新 max_assigned_client_id_
            ↓
    最终设置
        ├─ set_client_as_manager_host? → 设置 routing_manager_host_ 和 pid_
        └─ 记录到 used_client_ids_[]，更新 max_used_client_ids_index_
            ↓
    解锁并返回客户端 ID
```

这里系统里不同vsomeip app(包括vsomeip routing manager host)分享和查询`max_used_client_ids_index_`和`used_client_ids_[]`这种公共需要的信息，事实上利用了**共享内存**`/dev/shm/vsomeip`


### 2.2 routing manager host决策


#### 2.2.1 routing manager的基本逻辑

Routing Manager Host 是整个节点的路由控制中心，负责服务发现、消息路由和系统管理；Routing Manager Proxy 是应用程序的路由接口，负责本地服务管理和消息转发。它们形成了一个中心化但分布式的架构，既保证了路由的一致性和效率，又提供了良好的扩展性和故障隔离能力。

```
节点A:
├── Routing Manager Host (routing_manager_impl)
│   ├── 服务注册表
│   ├── SD协议处理
│   └── 远程通信管理
├── Application 1 + Routing Manager Proxy
├── Application 2 + Routing Manager Proxy
└── Application N + Routing Manager Proxy

```

通信关系:
- 所有Proxy通过Unix套接字连接到Host
- Host负责跨节点通信
- 应用间消息通过Host路由

#### 2.2.2 host和proxy的数据字段使用


| 功能           | Host（路由管理器主机）            | Proxy（客户端）           |
| -------------- | --------------------------------- | ------------------------- |
| 互斥锁管理     | √ 初始化并管理锁状态              | ! 使用锁但处理 EOWNERDEAD |
| 路由管理器设置 | √ 设置 routing_manager_host_=自己 | x 只读查询                |
| 进程ID管理     | √ 设置 pid_=getpid()              | x 只读检查和监控          |
| 客户端数量限制 | √ 强制执行 max_clients_ 限制      | ! 检查但不强制执行        |
| 故障恢复       | √ 清理僵尸客户端和重置状态        | ! 检测故障并可能接管      |
| 初始化状态     | √ 设置 initialized_=1             | x 等待并检查状态          |


#### 2.2.3 host决策

- 配置优先: 显式配置永远优先于自动配置
- 先到先得: 自动配置模式下，第一个应用成为host
- 单host原则: 每个节点只能有一个routing manager host
- 进程监控: Linux平台支持host进程的健康检查和自动接管
- 无热切换: 不支持运行时的host角色转移

```
应用启动 → 读取配置 → 检查routing字段
    ↓
配置为空？
├─ 是 → 自动配置模式 → 竞争host角色
└─ 否 → 显式配置模式 → 检查应用名匹配
    ↓
创建routing_manager_impl (host) 或 routing_manager_proxy (proxy)
```


#### 2.2.4 host接管机制

针对于host接管的机制则体现在

```c++
// vsomeip/implementation/utility/src/utility.cpp
        const std::string its_name = _config->get_routing_host();
        ...
        pid_t pid = getpid();
        if (its_name == "" || _name == its_name) {
            if (the_configuration_data__->pid_ != 0) {
                if (pid != the_configuration_data__->pid_) {
                    if (kill(the_configuration_data__->pid_, 0) == -1) {
                        VSOMEIP_WARNING << "Routing Manager seems to be inactive. Taking over...";
                        the_configuration_data__->routing_manager_host_ = 0x0000;
                    }
                }
            }
        }
```

自动配置模式 (its_name == ""):
配置文件中未指定routing host或指定为空
任何应用都有权检查和接管

匹配指定模式 (_name == its_name):
当前应用名与配置指定的routing host匹配
只有被指定的应用有权检查和接管



### 2.3 共享内存的使用


#### 2.3.1 共享内存和数据字段


涉及到具体的共享内存的数据结构

```c++
// vsomeip/implementation/configuration/include/internal.hpp
struct configuration_data_t {
    volatile char initialized_;     // 偏移 0x00, 1字节
    // 填充3字节到8字节边界 (pthread_mutex_t需要8字节对齐)
    pthread_mutex_t mutex_;         // 偏移 0x08, 40字节 
    pid_t pid_;                    // 偏移 0x30, 4字节
    // 填充4字节对齐
    unsigned short client_base_;    // 偏移 0x34, 2字节
    unsigned short max_clients_;    // 偏移 0x36, 2字节  
    int max_used_client_ids_index_; // 偏移 0x38, 4字节
    unsigned short max_assigned_client_id_; // 偏移 0x3C, 2字节
    unsigned short routing_manager_host_;   // 偏移 0x3E, 2字节
    // used_client_ids__ 数组从偏移 0x40 开始
};
```


在一个实例里我们可以把`/dev/shm/vsomeip`中的内容打印出来

```bash
hexdump -x /dev/shm/vsomeip | head -20
0000000    0001    0000    0000    0000    0000    0000    0001    0000
0000010    0000    0000    0000    0000    0090    0000    0000    0000
0000020    0000    0000    0000    0000    0000    0000    0000    0000
0000030    15da    002b    0001    00ff    0004    0000    0001    0001
0000040    0000    0001    1277    1344    0000    0000    0000    0000
0000050    0000    0000    0000    0000    0000    0000    0000    0000
```

我们分开来看这些共享内存中的, 可以归类到这个表格中

| 偏移      | 内存数据(2字节)   | 字段名                     | 解析值         | 说明                   |
| --------- | ----------------- | -------------------------- | -------------- | ---------------------- |
| 0x00-0x01 | 0001              | initialized_ + padding     | 0x01           | 已初始化               |
| 0x02-0x07 | 0000 0000 0000    | padding                    | padding        | 对齐到8字节边界        |
| 0x08-0x2F | 0001 0000 0000... | pthread_mutex_t            | 40字节互斥锁   | 进程间互斥锁           |
| 0x30-0x33 | 15da 002b         | pid_                       | 0x002b15da     | 进程ID = 2,823,642     |
| 0x34-0x35 | 0001              | client_base_               | 0x0001 = 1     | 客户端ID基础值         |
| 0x36-0x37 | 00ff              | max_clients_               | 0x00ff = 255   | 最大客户端数           |
| 0x38-0x3B | 0004 0000         | max_used_client_ids_index_ | 0x00000004 = 4 | 已使用客户端ID数组索引 |
| 0x3C-0x3D | 0001              | max_assigned_client_id_    | 0x0001 = 1     | 最大已分配客户端ID     |
| 0x3E-0x3F | 0001              | routing_manager_host_      | 0x0001 = 1     | 路由管理器主机ID       |


客户端ID数组 (used_client_ids__) 从0x40开始：

| 偏移      | 内存数据 | 数组索引 | 客户端ID | 十进制 | 应用程序       |
| --------- | -------- | -------- | -------- | ------ | -------------- |
| 0x40-0x41 | 0000     | [0]      | 0x0000   | 0      | (空槽位)       |
| 0x42-0x43 | 0001     | [1]      | 0x0001   | 1      | vsomeipd       |
| 0x44-0x45 | 1277     | [2]      | 0x1277   | 4727   | service-sample |
| 0x46-0x47 | 1344     | [3]      | 0x1344   | 4932   | client-sample  |




回到具体代码中，有以下的对应关系

```c++
// 实际结构体字段值：
the_configuration_data__->initialized_ = 0x01;                    //已初始化
the_configuration_data__->pid_ = 0x002b15da;                      //实际进程ID
the_configuration_data__->client_base_ = 0x0001;                  //基础ID=1  
the_configuration_data__->max_clients_ = 0x00ff;                  //最大255个
the_configuration_data__->max_used_client_ids_index_ = 4;         //使用了4个索引
the_configuration_data__->max_assigned_client_id_ = 0x0001;       //最大分配ID=1
the_configuration_data__->routing_manager_host_ = 0x0001;         //路由管理器=1
```


#### 2.3.2 routing manager 和共享内存的初始化

  场景1：Host先启动

  Host启动 → 创建共享内存 → 初始化所有字段 → 设置自己为routing_manager_host
  Proxy启动 → 连接现有内存 → 请求client_id → 正常工作

  场景2：Proxy先启动

  Proxy启动 → 创建共享内存 → 初始化所有字段 → routing_manager_host_=0x0000
  Host启动 → 连接现有内存 → 检查routing_manager_host_==0 → 设置自己为routing_manager_host

  场景3：多个Proxy先启动

  Proxy1启动 → 创建共享内存 → 初始化 → routing_manager_host_=0x0000
  Proxy2启动 → 连接现有内存 → 请求client_id
  Host启动 → 连接现有内存 → 设置自己为routing_manager_host


| 启动顺序      | 初始化者    | routing_manager_host设置 | 共享内存状态            |
| ------------- | ----------- | ------------------------ | ----------------------- |
| Host先启动    | Host        | Host设置自己             | Host创建和初始化        |
| Proxy先启动   | Proxy       | 保持0x0000，等待Host     | Proxy创建和初始化       |
| 多Proxy先启动 | 第一个Proxy | 保持0x0000，等待Host     | 第一个Proxy创建和初始化 |


