---
title: vsomeip STR(Suspend To RAM)
date: 2026-07-09 10:00:00 +0800
categories: [SOME/IP, vsomeip]
tags: [cpp,linux,system]
---

## 1 前言

STR(Suspend To RAM) 是系统级的休眠挂起。在车载场景里, ECU 进入休眠前如果直接断电, 远端节点是感知不到本节点服务下线的——SOME/IP 的服务发现依赖 TTL, 对端要等到 TTL 超时才会把服务标记为不可用. 这段空窗期里, 对端仍会向一个已经"死了"的服务发请求、等响应, 最终触发超时和错误的 availability 状态.

![STR 直接断电的困境](/assets/images/2026/Jul/01-scene-power-cut-problem.png)

vsomeip 给了系统软件一个显式的接口 `set_routing_state()`, 让本节点在真正休眠前先把服务"优雅地"撤下来: 主动发 StopOffer / StopSubscribe, 清掉本地路由表和订阅记录. 唤醒后再把服务重新 offer 出去, 整个过程对应用层尽量透明.

本文记录 vsomeip 对 STR 的支持, 以及 `suspend_resume_tests` 这个测试 case 是怎么验证它的. 阅读前建议先了解 vsomeip 的 Routing Host / Routing Proxy 架构.读者也可以根据自己的技术经验和偏好，选择和阅读关注的话题章节

对于实际操作感兴趣的读者也可以参考"vsomeip test-运行vsomeip的官方测试"这篇文章搭建本地代码和测试

- 第2章到第5章介绍主要架构和代码的逻辑
- 第6章介绍官方仓库中的对应场景的测试: suspend_resume_tests
- 第7章则从这个测试过程中记录的网络报文情况来做进一步的分析
- 第8章再次介绍关键设计


## 2 接口

### 2.1 set_routing_state()

路由状态枚举定义在 `interface/vsomeip/enumeration_types.hpp`:

```cpp
enum class routing_state_e : uint8_t {
    RS_RUNNING   = 0x00,  // 正常运行
    RS_SUSPENDED = 0x01,  // 进入STR挂起
    RS_RESUMED   = 0x02,  // STR恢复
    RS_SHUTDOWN  = 0x03,  // 系统关闭
    RS_DIAGNOSIS = 0x04,  // 诊断模式
    RS_UNKNOWN   = 0xFF
};
```

接口声明在 `interface/vsomeip/application.hpp`:

```cpp
virtual void set_routing_state(routing_state_e _routing_state) = 0;
```

这个接口名义上是 `application` 的方法, 但注释明确写了: 只有路由管理进程(Routing Host, 也就是 routingmanagerd 或者充当 Host 的那个应用)才应该调用它.

原因在于它背后要做的事远不止改一个状态变量——要停掉 Service Discovery、通过 IPC 向所有 Proxy 广播 SUSPEND 命令、向网络发 StopOffer/StopSubscribe、清理路由表. 这些控制权只有 Routing Host 才有, 普通 Proxy 应用既没有 SD 的控制权, 也碰不到路由表, 调它也没意义.

### 2.2 register_routing_state_handler()

普通应用是被动的, 它们通过注册回调来接收状态变更通知:

```cpp
// interface/vsomeip/application.hpp
virtual void register_routing_state_handler(const routing_state_handler_t &_handler) = 0;

// interface/vsomeip/handler.hpp
typedef std::function<void(routing_state_e)> routing_state_handler_t;
```

Host 切状态时会调这个 handler 通知宿主应用, 应用据此做自己的休眠/恢复处理(比如停掉业务线程、保存上下文).


## 3 挂起流程(RS_SUSPENDED)

挂起的根本矛盾: 本节点要断电休眠, 但它在 SOME/IP 网络里还"挂着"——既作为 provider 向外 offer 服务, 也作为 subscriber 订阅着远端服务, 本地还缓存着路由表和事件 payload. 直接断电的话, 远端会继续往一个已死的节点发请求、推事件直到 TTL 超时; 本地缓存在唤醒后也全是陈旧的, 用了就出错. 所以挂起要做的就是把本节点从网络里干净地摘除, 同时清空本地陈旧状态, 为唤醒后的全新开始铺路.

实现集中在 `routing_manager_impl.cpp` 的 `set_routing_state(RS_SUSPENDED)` 分支. 本节点同时是 provider 和 subscriber, 对外两个方向的订阅关系都要撤, 再加上对内清空, 一共四块:

#### a. 前提: 关闸门 + 通知本地

```cpp
discovery_->stop();   // SD 停收停发, 不再处理进来的 OfferService/FindService
send_suspend();       // 通过 IPC 向所有连接的 Proxy 广播 SUSPEND_ID(0x30)
```

先把 SD 闸门关上, 清理过程中不再有新的订阅/offer 插进来造成状态混乱. 同时广播 SUSPEND 命令, 让本地所有 proxy app 一起做各自的清理:

```cpp
// routing_manager_stub.cpp
void routing_manager_stub::send_suspend() const {
    protocol::suspend_command its_command;
    std::vector<byte_t> its_buffer;
    protocol::error_e its_error;
    its_command.serialize(its_buffer, its_error);
    if (its_error == protocol::error_e::ERROR_OK)
        broadcast(its_buffer);  // 广播给所有已连接的 proxy
}
```

Proxy 收到 `SUSPEND_ID` 后走 `on_suspend()`, 清掉自己作为 provider 时记录的远端订阅者计数:

```cpp
// routing_manager_client.cpp
void routing_manager_client::on_suspend() {
    std::lock_guard<std::mutex> its_lock(remote_subscriber_count_mutex_);
    // Unsubscribe everything that is left over.
    for (const auto &s : remote_subscriber_count_) {
        for (const auto &i : s.second) {
            for (const auto &e : i.second)
                routing_manager_base::unsubscribe(
                    VSOMEIP_ROUTING_CLIENT, nullptr,
                    s.first, i.first, e.first, ANY_EVENT);
        }
    }
    remote_subscriber_count_.clear();
}
```

#### b. 对外·provider 方向: 撤掉我 offer 的服务

本节点作为服务提供方, 要把别人订阅我服务的记录清掉, 并告诉远端我的服务下线:

```cpp
expire_subscriptions(true);   // 强制过期所有远程订阅(别人订阅我服务的记录)
// 遍历 offered services, 调 stop_offer_service() 收集后统一发出
discovery_->send_collected_stop_offers(_service_infos);
```

`expire_subscriptions(true)` 的 `_force=true` 是关键: 正常过期要等超时, 但挂起是立刻要走, 没时间等, 所以直接强过. 紧接着发 StopOffer, 让远端马上知道服务下线, 而不是干等到 TTL 超时.

#### c. 对外·subscriber 方向: 撤掉我订阅的服务

本节点作为订阅方, 订阅的远端服务也要撤. 不主动发 StopSubscribe 的话, 远端会一直给一个休眠节点推事件:

```cpp
discovery_->unsubscribe_all_on_suspend();  // 给所有已订阅的 eventgroup 发 TTL=0 的 StopSubscribe
```

`unsubscribe_all_on_suspend()`(`service_discovery_impl.cpp`) 把所有已订阅的 eventgroup 按 remote 地址分组, 组织成 TTL=0 的 StopSubscribe 发出去, 然后清空 `subscribed_` map. TTL=0 是 SOME/IP SD 里"立即撤销"的语义, 远端收到后马上停止推送.

#### d. 对内清空: 路由表 + 缓存

```cpp
remote_subscription_state_.clear();      // 清空远程订阅状态机
// 遍历 services_remote_, 删路由信息 + 清事件 payload 缓存
del_routing_info(s.first, i.first, has_reliable, has_unreliable);
unset_all_eventpayloads(s.first, i.first);
```

路由表和事件 payload 缓存一并清掉, 保证 resume 后是干净的重新开始. 唤醒后网络拓扑可能已经变了, 这些缓存留着只会出错.

![挂起四步流程](/assets/images/2026/Jul/02-flowchart-suspend-four-steps.png)


## 4 恢复流程(RS_RESUMED)

恢复是挂起的镜像: 唤醒后本节点状态是"空的"(挂起时把对外订阅和本地缓存都清光了), 要把自己重新接回网络. 核心矛盾是怎么让远端快速重新发现我, 又不引入陈旧状态. 这决定了恢复的顺序——先复位服务元数据, 通知应用做好恢复准备, 然后才启动 SD 把服务推出去.

实现集中在 `set_routing_state(RS_RESUMED)` 分支(`routing_manager_impl.cpp:3860-3895`), 完整 6 步:

```cpp
last_resume_ = std::chrono::steady_clock::now();   // 1. 记录恢复时间戳(诊断用)

// 2. 复位服务元数据: TTL + mainphase
for (const auto &its_service : get_offered_services())
    for (const auto &its_instance : its_service.second) {
        its_instance.second->set_ttl(DEFAULT_TTL);
        its_instance.second->set_is_in_mainphase(false);
    }

discovery_->set_diagnosis_mode(false);             // 3. 退出诊断模式
routing_state_handler_(_routing_state);            // 4. 通知应用层恢复完成
discovery_->start();                               // 5. 启动 SD

// 6. 重新 offer 所有服务, 触发 initial offer 阶段
for (const auto &its_service : get_offered_services())
    for (const auto &its_instance : its_service.second)
        discovery_->offer_service(its_instance.second);
```

几个顺序和参数的选择都有讲究:

#### a. TTL 必须在 offer 之前复位

挂起时发 StopOffer, 服务的 TTL 被设成 0. 如果带着 TTL=0 重新 offer, 远端收到 TTL=0 的 OfferService 会直接当作"服务下线", 等于刚恢复就被判死. 所以第 2 步先把 TTL 改回 DEFAULT_TTL, 第 6 步才真正发 offer.

#### b. set_is_in_mainphase(false) 推回 initial offer 阶段

SD 有两个阶段: initial offer 阶段(带 repetition, 短间隔重复发几次) 和 main phase(低频 cyclic offer). `set_is_in_mainphase(false)` 把服务推回 initial offer 阶段.

唤醒是个"重新上线"事件, 远端可能已经等了很久, 需要快速重新发现. 直接进 main phase 的低频周期, 远端要等好几个周期才捞到; 走 initial offer 的 repetition 能在最短时间内把服务推到对端.

#### c. 退出诊断模式 + 通知应用, 都在启动 SD 之前

`set_diagnosis_mode(false)` 退出诊断模式(对应枚举里的 RS_DIAGNOSIS, 诊断模式下 SD 行为不一样), 确保恢复走的是正常 SD 路径.

`routing_state_handler_(_routing_state)` 调用第 2.2 节注册的 handler, 通知应用层"恢复完成", 应用据此恢复自己的业务线程、上下文. 它刻意放在 `discovery_->start()` 之前: 让应用先做好恢复准备, 再开始对外发 SD, 避免远端已经开始重订阅、推事件了, 应用层还没准备好接.

#### d. Proxy 不收显式的 RESUME 命令

挂起时有 `SUSPEND_ID` 广播给 Proxy, 但恢复时**没有**对应的 `RESUME_ID` 命令发给 Proxy. Proxy 的恢复完全靠 SD 间接完成:

```
Host: 重启 SD, 重新 offer 服务
  → Host 通过 routing_info 通知 Proxy (RIE_ADD_SERVICE_INSTANCE)
    → Proxy: send_pending_subscriptions()            // 先重建订阅
    → Proxy: host_->on_availability(AS_AVAILABLE)    // 再通知应用服务可用
```

这是有意为之. Proxy 不掌握 SD, 没有能力主动发起服务发现; 它能做的只是响应 Host 的通知. Host 重新 offer 后, 通过已有的 routing_info 通道告诉 Proxy"服务又可用了", Proxy 先把挂起前 pending 的订阅重新发出去, 再回调应用层 availability. 注意顺序是先重订阅再通知可用——这样应用收到 available 回调时订阅已经就位, 不会出现"知道服务可用却还没订阅"的空窗.

Proxy 侧的恢复逻辑和正常的服务可用通知复用同一条路径, 不用单独搞一套 resume 逻辑, 职责单一.

![恢复六步流程](/assets/images/2026/Jul/03-flowchart-resume-six-steps.png)


## 5 Host 与 Proxy 职责对比

| 维度       | Routing Host                           | Routing Proxy                              |
| ---------- | -------------------------------------- | ------------------------------------------ |
| 角色       | 主导者、协调者                         | 被动执行者                                 |
| 挂起触发   | 收到 `set_routing_state(RS_SUSPENDED)` | 收到 Host 广播的 SUSPEND_ID                |
| 网络操作   | 发 StopOffer、StopSubscribe SD 消息    | 无                                         |
| SD 控制    | 停止/启动 Service Discovery            | 无 SD 控制权                               |
| 路由表操作 | 删远程服务、强过期订阅                 | 仅清本地 subscriber count                  |
| 恢复触发   | 收到 `set_routing_state(RS_RESUMED)`   | 通过 routing_info 间接感知(无 RESUME 命令) |
| 恢复动作   | 重新 offer 所有服务、重启 SD           | 收到 availability 通知后自动重订阅         |

整体交互时序:

```
+-----------------------------------------------------------------+
|                SUSPEND phase                                    |
+----------------------------------+------------------------------+
| Routing Host (impl)              | Routing Proxy (client)       |
+----------------------------------+------------------------------+
| 1. Stop SD                       |                              |
| 2. Broadcast SUSPEND_ID ---------> Recv SUSPEND_ID              |
| 3. Expire remote subscr.         |    cancel subscriptions      |
| 4. Send StopOffer (net)          |    clear subscriber cnt      |
| 5. Send StopSubscribe (net)      |                              |
| 6. Del remote svc table          |                              |
+-----------------------------------------------------------------+
|                RESUME phase                                     |
+----------------------------------+------------------------------+
| Routing Host (impl)              | Routing Proxy (client)       |
+----------------------------------+------------------------------+
| 1. Reset TTL                     |                              |
| 2. Restart SD                    |                              |
| 3. Re-offer all services         |                              |
| 4. SD discovers remotes          |                              |
| 5. Update routing_info  ---------> Recv availability cb         |
|                                  |    auto re-subscribe         |
+-----------------------------------------------------------------+
```

![Host 与 Proxy 职责对比](/assets/images/2026/Jul/04-comparison-host-vs-proxy.png)


## 6 测试: suspend_resume_tests

测试目录 `test/network_tests/suspend_resume_tests/`. 这个 case 用双机拓扑把整个 STR 流程跑一遍.

### 6.1 拓扑

```
+---- Master Node (service side)---------------+   +---- Slave Node (client side)--+
|                                              |   |                               |
|  routingmanagerd                             |   |  suspend_resume_test_client   |
|    routing="routingmanagerd"(Routing Host)   |   |    routing="suspend_resume..  |
|    listen SIGUSR1 -> RS_SUSPENDED            |   |    _test_client"(self=Host)   |
|    listen SIGUSR2 -> RS_RESUMED              |   |    client_id=0x4567           |
|    PID passed to service via argv            |   |                               |
|                                              |   +-------------------------------+
|  suspend_resume_test_service                 |
|    routing="routingmanagerd"(Routing Proxy)  |
|    client_id=0x7654                          |
|    holds daemon PID, signals daemon          |
+----------------------------------------------+
```

关键点: 测试里 STR 不是由系统电源管理触发的(测试环境没有那个), 而是用信号模拟. service 进程启动时通过 `argv[1]` 拿到 daemon 的 PID, 收到 client 发来的控制消息后, 向 daemon 发 SIGUSR1(挂起)/SIGUSR2(恢复).

信号处理放在独立的 `sighandler_thread` 里, 而不是直接在 signal handler 函数体里干活. 原因是 signal handler 里不能安全调用 boost logger 这类不可重入函数——handler 只置一个状态变量、`notify()` 唤醒专门的处理线程, 真正的 `set_routing_state()` 调用在线程里做.

```
SIGUSR1 → routing_state = RS_SUSPENDED → notify → sighandler_thread → set_routing_state(RS_SUSPENDED)
SIGUSR2 → routing_state = RS_RESUMED   → notify → sighandler_thread → set_routing_state(RS_RESUMED)
```

### 6.2 运行测试

和 debounce 测试一样, 两个终端分别跑 master / slave 的 starter 脚本. 网络隔离用前篇文章里配好的 `slave_ns` 命名空间.

```bash
# Master 侧, 在 vsomeip/build/test/network_tests/ 下
./suspend_resume_test_master_starter.sh

# Slave 侧, 另一个终端
sudo ip netns exec slave_ns bash -c \
  "export LD_LIBRARY_PATH=$(pwd)/../../:$LD_LIBRARY_PATH && ./suspend_resume_test_slave_starter.sh"
```

### 6.3 测试时序

client 的 `run_test()` 串行走 5 个阶段, 每个阶段都有 10s 限时:

```
时间  Master: routingmanagerd     Master: service          Slave: client
 │    (Routing Host)              (Routing Proxy)          (Routing Host)
 │
 ├──── 阶段0: 服务发现 ────────────────────────────────────────────────────
 │    SD: OfferService(0x6311) ──────────────────────────────────────────▶
 │                                                         on_availability(true)
 │
 ├──── 阶段1: toggle() 订阅/取消循环 ──────────────────────────────────────
 │                                                         subscribe → 收初始事件
 │                                                         sleep(3s) → unsubscribe
 │                               on_subscribe(false) sleep 2s  ← 慢速取消
 │                                                         subscribe → 收初始事件
 │                                                         sleep(2s) → unsubscribe
 │                               on_subscribe(false) sleep 2s
 │                                                         subscribe → 保持订阅
 │                                                         (if !has_received_)
 │                                                           cv_.wait(10s) 等收初始事件
 │
 ├──── 阶段2: 触发 STR ────────────────────────────────────────────────────
 │                                                         send_suspend() payload=0x00
 │                               on_message(TEST_SUSPEND)
 │                                 sr_run() 线程: kill(daemon, SIGUSR1)
 │    signal(SIGUSR1) → RS_SUSPENDED
 │      StopOffer / StopSubscribe 发出, SD 停止
 │                                      (等 TTL 超时, 最多 ~3s)
 │                                                         on_availability(false)
 │                                                           has_received_ = false
 │
 ├──── 阶段3: resume ──────────────────────────────────────────────────────
 │                               sleep(5s) ← 挂起模拟
 │                                 kill(daemon, SIGUSR2)
 │    signal(SIGUSR2) → RS_RESUMED
 │      TTL 重置, SD 重启, 重新 offer
 │    SD: OfferService ──────────────────────────────────────────────────▶
 │                                                         on_availability(true)
 │                                                         收到初始事件 (has_received_=true)
 │
 ├──── 阶段4: 收尾 ────────────────────────────────────────────────────────
 │                                                         send_stop() payload=0xFF
 │                               on_message(TEST_STOP) → 退出
```

![五阶段测试流程](/assets/images/2026/Jul/05-timeline-test-five-phases.png)

有两个设计点值得拎出来:

**toggle 的目的.** client 在触发 suspend 前故意做了 3 次 subscribe/unsubscribe 循环, 而且 service 侧 `on_subscribe(false)` 里 `sleep(2s)` 故意慢. 这是在制造竞态: suspend 触发时, 上一次 unsubscribe 的慢速处理可能还没完. 测试要验证在这种"订阅状态正在变动"的窗口里触发 suspend 不会死锁、不会留下半截状态.

**has_received_ 的重置时机.** client 在 `on_availability(false)`(服务变不可用)时把 `has_received_` 置 false. 这样 resume 后必须真正重新收到一次初始事件才算通过, 不能拿 suspend 前收到的旧事件顶数. 保证恢复后的事件投递是实打实的重新投递, 而不是误用了挂起前的状态.


## 7 报文级时序验证

`str_test.pcapng`是测试进行时通过wireshark抓取对应网卡并保存的网络报文. 
用 `str_test.pcapng` (60 帧, 20.9s) 逐帧验证上面的时序. `tshark -r str_test.pcapng` 原始输出如下 (字节数等非关键字段用 `**` 隐去):

```
 1  0.000  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 2  0.030  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 3  0.090  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 4  0.210  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 5  0.450  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 6  1.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 7  2.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 8  3.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
 9  4.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
10  5.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
11  6.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
12  7.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
13  8.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
14  8.867  192.168.99.2 → 224.0.0.1    SOME/IP-SD ** [Find]
15  8.867  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [Offer]
16  8.870  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
17  8.870  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
18  8.876  192.168.99.1 → 192.168.99.2 SOME/IP    ** (0x6311, 0x8005)
19  9.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
20  9.490  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
21  9.490  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
22 10.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
23 10.490  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
24 10.491  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
25 11.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
26 11.491  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
27 11.491  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
28 11.867  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [StopSubscribe]
29 11.867  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
30 12.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
31 12.491  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [StopSubscribe][Subscribe]
32 13.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
33 13.491  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [StopSubscribe][Subscribe]
34 13.867  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [StopSubscribe]
35 13.868  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
36 13.868  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
37 13.873  192.168.99.2 → 192.168.99.1 SOME/IP    ** (0x6311, 0x0001)
38 13.873  192.168.99.1 → 192.168.99.2 SOME/IP    ** (0x6311, 0x8005)
39 13.873  192.168.99.1 → 192.168.99.2 SOME/IP    ** (0x6311, 0x8005)
40 14.170  ** → **                    ARP        ** Who has 192.168.99.2? Tell 192.168.99.1
41 14.170  ** → **                    ARP        ** Who has 192.168.99.1? Tell 192.168.99.2
42 14.170  ** → **                    ARP        ** 192.168.99.1 is at **
43 14.170  ** → **                    ARP        ** 192.168.99.2 is at **
44 14.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
45 14.490  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
46 15.490  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
47 15.490  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [StopSubscribe][Subscribe]
48 15.868  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
49 15.868  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
50 15.868  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
51 15.868  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [StopOffer]
52 15.873  192.168.99.1 → 192.168.99.2 SOME/IP    ** (0x6311, 0x8005)
53 15.873  192.168.99.2 → 192.168.99.1 ICMP       ** Destination unreachable (Port unreachable)
54 20.879  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [Offer]
55 20.882  192.168.99.2 → 192.168.99.1 SOME/IP-SD ** [Subscribe]
56 20.882  192.168.99.1 → 192.168.99.2 SOME/IP-SD ** [SubscribeAck]
57 20.887  192.168.99.1 → 192.168.99.2 SOME/IP    ** (0x6311, 0x8005)
58 20.887  192.168.99.1 → 192.168.99.2 SOME/IP    ** (0x6311, 0x8005)
59 20.893  192.168.99.2 → 192.168.99.1 SOME/IP    ** (0x6311, 0x0001)
60 20.893  192.168.99.1 → 224.0.0.1    SOME/IP-SD ** [StopOffer]
```

SD entry type 用 tshark 解码值.

### a. 初始 offer 阶段 — 指数退避可见

master 启动后的前 5 个 OfferService 间隔不是均匀的:

| 帧 | 时间 | 间隔 | 阶段 |
| --- | --- | --- | --- |
| 1 | 0.000 | — | initial offer |
| 2 | 0.030 | 30ms | repetition |
| 3 | 0.090 | 60ms | repetition |
| 4 | 0.210 | 120ms | repetition |
| 5 | 0.450 | 240ms | repetition |
| 6 | 1.490 | 1040ms | **main phase 开始** |
| 7-13 | 2.49-8.49 | ~1000ms | cyclic offer |

30→60→120→240ms 是典型的 SD initial offer 指数退避. 对应第 4.b 节 `set_is_in_mainphase(false)` 把服务推回 initial offer 阶段, 短间隔 repetition 让远端快速发现. resume 后也会走同样的流程 (本次 pcap 中 resume 后测试立即结束, 只抓到 1 帧 offer 就收到 TEST_STOP 退出了).

### b. toggle 阶段报文模式

client 的 toggle() 产生成对的 StopSubscribe (TTL=0) + Subscribe (TTL=3):

| 帧 | 时间 | 方向 | Entry Type | TTL | 说明 |
| --- | --- | --- | --- | --- | --- |
| 28 | 11.867 | slave→master | 0x06 StopSubscribe | 0 | toggle 第 1 次 unsubscribe |
| 29 | 11.867 | slave→master | 0x06 Subscribe | 3 | toggle 立即 resubscribe |
| 34 | 13.867 | slave→master | 0x06 StopSubscribe | 0 | toggle 第 2 次 unsubscribe |
| 35 | 13.868 | slave→master | 0x06 Subscribe | 3 | toggle 立即 resubscribe |

有个细节: toggle 的显式 unsubscribe/subscribe 拆成两条独立 SD 消息 (length=48 各一条), 但 cyclic 订阅刷新 (帧 31/33/47, t=12.491/13.491/15.491) 把 StopSubscribe+Subscribe 合到一条里 (length=64, 两个 entry). vsomeip SD 层刷新订阅时会把 stop 旧的 + subscribe 新的打包发, 省一半报文.

### c. suspend 过渡 — 完整报文序列

| 帧 | 时间 | 方向 | 类型 | 说明 |
| --- | --- | --- | --- | --- |
| 37 | 13.873 | slave→master:34504 | SOME/IP Request | **TEST_SUSPEND** payload=0x00 |
| 44 | 14.491 | master→224.0.0.1 | SD Offer (TTL=3) | cyclic offer, suspend 尚未生效 |
| 46 | 15.491 | master→224.0.0.1 | SD Offer (TTL=3) | cyclic offer, 仍未生效 |
| 48-50 | 15.868 | master→slave | SD SubscribeAck x3 | on_subscribe sleep 返回后积压的响应 |
| **51** | **15.868** | master→224.0.0.1 | **SD StopOffer (TTL=0)** | **suspend 生效** |
| 52 | 15.873 | master→slave:43126 | SOME/IP Notify | 最后的初始事件 (payload 010203) |
| 53 | 15.873 | slave→master | **ICMP port unreachable** | slave 端口 43126 已关闭 |

**StopOffer 的 SD 编码.** StopOffer 和正常 OfferService 的 SD entry type 都是 `0x01`, 区别只在 TTL:

```
帧 13 (Offer, TTL=3):    01 00 00 10  63 11 00 02  01 00 00 03  00 00 ...
帧 51 (StopOffer, TTL=0): 01 00 00 10  63 11 00 02  01 00 00 00  00 00 ...
                                                 TTL--^^^^^^^^^^
```

SOME/IP-SD 里 TTL=0 就是"立即撤销", 不另设 StopOffer type — `send_collected_stop_offers()` 发出来的就是这个.

**suspend 生效有 ~2s 延迟.** 从 TEST_SUSPEND (t=13.873) 到 StopOffer (t=15.868), 中间隔了约 2 秒. 这段时间 cyclic offer 仍在发 (帧 44/46), 说明 `discovery_->stop()` 还没执行到 — `set_routing_state(RS_SUSPENDED)` 还没开始. 原因是 service 的 `on_subscribe(false)` 里有 `sleep(2s)`: toggle 最后一次 unsubscribe 的回调还没返回, 事件处理线程被阻塞, TEST_SUSPEND 消息排在队列里等 on_subscribe 返回后才被 `on_message` 处理. 这正是 6.3 节 toggle 制造竞态想测的场景: 订阅状态正在变动时触发 suspend, 系统照样能处理. 帧 48-50 的 3 个 SubscribeAck 在 StopOffer 前一瞬间发出, 是阻塞期间积压的响应被集中处理.

帧 53 的 ICMP port unreachable 是 endpoint 清理的副产物: master 向 slave 旧端口 43126 发初始事件, 但该 UDP socket 已在 suspend 清理中关闭, slave kernel 回复 ICMP type 3 code 3.

### d. resume 过渡 — 完整报文序列

StopOffer (帧 51, t=15.868) 之后有 **5.006 秒完全沉默** (帧 53 t=15.873 → 帧 54 t=20.879), 对应 `sr_run()` 里的 `sleep(5s)` — 这段时间 SD 停止, 网络上没有任何 SOME/IP 报文. 5s 后:

| 帧 | 时间 | 方向 | 类型 | 说明 |
| --- | --- | --- | --- | --- |
| **54** | **20.879** | master→224.0.0.1 | **SD Offer (TTL=3, session=0x0001)** | **resume re-Offer** |
| 55 | 20.882 | slave→master | SD Subscribe (TTL=3) | client 3ms 后即重订阅 |
| 56 | 20.882 | master→slave | SD SubscribeAck | |
| 57-58 | 20.887 | master→slave:50579 | SOME/IP Notify x2 | 初始事件重投递 (payload 010203) |
| 59 | 20.893 | slave→master:34504 | SOME/IP Request | **TEST_STOP** payload=0xFF |
| 60 | 20.893 | master→224.0.0.1 | SD StopOffer (TTL=0) | 服务退出 |

**re-Offer 的 session ID 重置.** 帧 54 的 session=0x0001, 从 StopOffer 的 0x0015 跳回 0x0001, 跟测试第一帧 offer (帧 1) 一样 — SD 模块完整走了一遍 stop → start, 计数器归零了. TTL 从 0 恢复到 3, 对应代码里第 4.a 节的 `set_ttl(DEFAULT_TTL)` — 如果不做这一步, 带 TTL=0 的 offer 会被远端直接当作服务下线.

**事件重投递.** 帧 57-58 (t=20.887) 是 resume 后的初始事件, method=0x8005, payload=`01 02 03`, 与 suspend 前的初始事件 (帧 18, t=8.876) payload 完全相同. 目标端口从 43126 变为 50579 — slave 在 resume 后重建了 UDP socket, 旧端口已在 suspend 时关闭 (ICMP 帧 53 已证).

re-Offer 后 3ms client 就发出 Subscribe, 8ms 后收到初始事件. 从 SD 重启到应用层拿到事件, 全程 8ms — initial offer 阶段短间隔的意义就在这.

### e. 端口变化

suspend 前后 client 的 UDP 端口发生了变化:

- suspend 前: `192.168.99.2:43126` (帧 18, 37-39, 52)
- resume 后: `192.168.99.2:50579` (帧 57-59)

suspend 清掉了旧 endpoint, resume 后 client 重建了新的 UDP socket. 旧端口 43126 的关闭有 ICMP 帧 53 为证.


## 8 关键设计细节

| 方面                          | 说明                                                                                                                                                                        |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TTL=3 的时序含义              | SD TTL=3s, cyclic_offer_delay=1000ms. suspend 后 service 停发 cyclic offer, client 最多等约 3s 才感知不可用. 这解释了阶段2的超时为什么设 10s(3s TTL + 余量)                 |
| send_suspend() 后 sleep(50ms) | 保证 suspend 控制消息在网络层真正发出去, 避免 client 已经进入等待而 service 还没收到                                                                                        |
| 订阅恢复不自动                | resume 不自动恢复订阅, 必须靠 SD 重新发现后由 client 自动重订阅. 这和 suspend 时 `unsubscribe_all_on_suspend()` 清掉远端订阅是对称的——挂起时撤干净, 恢复时重新建            |
| 事件恢复(field 重投递)        | service `offer_service()` 里用 `ET_FIELD` 注册事件并 notify 初始 payload {0x1,0x2,0x3}. resume 后重新 offer 触发 field 初始事件重新投递, 这是恢复后应用能重新拿到状态的关键 |
| send_stop 仅成功时发          | client `run_test()` 里 `was_successful` 为真才发 send_stop, 失败则不发, 让 service 侧 30s 超时, 测试框架正常记录失败                                                        |
| 线程安全                      | `routing_state_mutex_` 保护状态切换的原子性                                                                                                                                 |
| 时间记录                      | `last_resume_` 记录最近一次恢复时间, 诊断用                                                                                                                                 |
| 信号触发 vs 生产触发          | 测试用 SIGUSR1/SIGUSR2 触发 daemon 状态切换, 生产环境里由系统电源管理调用 `set_routing_state()`                                                                             |