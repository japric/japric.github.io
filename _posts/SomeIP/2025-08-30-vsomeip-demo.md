---
title: 基于vsomeip commonapi的demo
date: 2025-08-30 17:20:00 +0800
categories: [SOME/IP, vsomeip]
tags: [cpp,linux,system]
---

## 1 前言

本文旨在提供一个集合vsomeip commonapi库以及示例的仓库, 用于了解源码、依赖关系和快速上手

**vsomeip**是SOME/IP规范的开源实现，作为COVESA项目的一部分设计。同时COVESA开发了**CommonAPI C++**的一套库和工具，方便开发者快速定义和实现IPC/RPC通信接口并基于vsomeip进行符合SOME/IP规范的通信

COVESA的设计理念可以参考官方的文档, 其中的IPC stack即为vsomeip

![The basic principle](/assets/images/2025/Aug/CommonAPIOverview01.png){:style="width: 75%; height: auto;"}

![how the elements of CommonAPI C++ fit together](/assets/images/2025/Aug/CommonAPIOverview02.png){:style="width: 75%; height: auto;"}

涉及到的库包括

运行时库
- [vsomeip](https://github.com/COVESA/vsomeip)
- [capicxx-core-runtime](https://github.com/COVESA/capicxx-core-runtime)
- [capicxx-someip-runtime](https://github.com/COVESA/capicxx-someip-runtime)

代码生成工具
- commonapi_core_generator
- commonapi_someip_generator

## 2 demo 仓库设计

考虑到涉及的仓库较多，以及需要预编译好的代码生成工具，这里选择使用git-repo工具以及manifests组织和管理仓库

同时集成了`capicxx-core-tools`仓库的几个官方example

```
.
├── build
├── capicxx-core-runtime
├── capicxx-core-tools
├── capicxx-someip-runtime
├── CMakeLists.txt -> vsomeip-demo/CMakeLists.txt
├── config
├── envsetup.sh -> vsomeip-demo/scripts/envsetup.sh
├── gen.sh -> vsomeip-demo/scripts/gen.sh
├── googletest
├── prebuilts
│   ├── commonapi_core_generator
│   └── commonapi_someip_generator
├── run.sh -> vsomeip-demo/scripts/run.sh
├── vsomeip
└── vsomeip-demo
```


## 3 demo 使用

使用环境 ubuntu20.04/ubuntu22.04

### preparation

#### tools

> install and configure these tools

- git-repo
- java 1.8
- libboost

libboost 可以直接所用apt来安装
```
sudo apt install libboost-dev libboost-system-dev libboost-thread-dev libboost-filesystem-dev libboost-log-dev
```

### download code

`repo init -u git@github.com:japric/vsomeip_repo.git -m default.xml --repo-url=https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/`

### build


```
chmod +x gen.sh
# generate interface code by commonapi tool in gen.sh
./gen.sh
mkdir build && cd build
cmake .. -DGTEST_ROOT=$(pwd)/../googletest
make -j8
cd ..
```

默认涉及的编译target参考如下, E0XXX(例如E01HelloWorldClient) 即为`capicxx-core-tools/CommonAPI-Examples/`中官方的例子

```
build$ make -j1
[ 37%] Built target vsomeip3
[ 40%] Built target vsomeip3-cfg
[ 50%] Built target vsomeip3-sd
[ 56%] Built target vsomeip3-e2e
[ 56%] Built target routingmanagerd
[ 59%] Built target CommonAPI
[ 68%] Built target CommonAPI-SomeIP
[ 68%] Built target E01HelloWorldClient
[ 71%] Built target E01HelloWorldService
[ 71%] Built target E01HelloWorld-someip
[ 75%] Built target E02AttributesClient
[ 75%] Built target E02AttributesService
[ 75%] Built target E02Attributes-someip
[ 75%] Built target E03MethodsClient
[ 75%] Built target E03MethodsService
[ 78%] Built target E03Methods-someip
[ 78%] Built target E04PhoneBookClient
[ 81%] Built target E04PhoneBookService
[ 84%] Built target E04PhoneBook-someip
[ 87%] Built target E05ManagerClient
[ 87%] Built target E05ManagerService
[ 90%] Built target E05Manager-someip
[ 90%] Built target E06UnionsClient
[ 93%] Built target E06UnionsService
[ 96%] Built target E06Unions-someip
[100%] Built target E08CrcProtectionClient
[100%] Built target E08CrcProtectionService
[100%] Built target E08CrcProtection-someip
```

### run

`source envsetup.sh`

> run service 

```
export VSOMEIP_CONFIGURATION=$(pwd)/config/vsomeip-local.json
VSOMEIP_APPLICATION_NAME=service-sample ./build/bin/E01HelloWorldService
```

> run client in another commandline window

```
export VSOMEIP_CONFIGURATION=$(pwd)/config/vsomeip-local.json
VSOMEIP_APPLICATION_NAME=client-sample ./build/bin/E01HelloWorldClient
```

服务端可以看到类似打印

```
[CAPI][INFO] Loading configuration file '/home/vsomeip_demo/capicxx-core-tools/CommonAPI-Examples/E01HelloWorld/commonapi4someip.ini'
[CAPI][INFO] Using default binding 'someip'
[CAPI][INFO] Using default shared library folder '/usr/local/lib/commonapi'
[CAPI][DEBUG] Loading library for local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld stub.
[CAPI][INFO] Loading configuration file /etc//commonapi-someip.ini
[CAPI][DEBUG] Added address mapping: local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld <--> [1234.5678(0.1)]
[CAPI][VERBOSE] Registering function for creating "commonapi.examples.E01HelloWorld:v0_1" proxy.
[CAPI][INFO] Registering function for creating "commonapi.examples.E01HelloWorld:v0_1" stub adapter.
[CAPI][DEBUG] Loading interface library "libE01HelloWorld-someip.so" succeeded.
[CAPI][INFO] Registering stub for "local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld"
2025-08-30 20:37:58.905028 [info] Using configuration file: "/home/vsomeip_demo/config/vsomeip-local.json".
2025-08-30 20:37:58.907083 [info] Parsed vsomeip configuration in 1ms
2025-08-30 20:37:58.907288 [info] Configuration module loaded.
2025-08-30 20:37:58.907383 [info] Security disabled!
2025-08-30 20:37:58.907462 [info] Initializing vsomeip (3.4.10) application "service-sample".
2025-08-30 20:37:58.908252 [info] Instantiating routing manager [Host].
2025-08-30 20:37:58.910002 [info] create_routing_root: Routing root @ /tmp/vsomeip-0
2025-08-30 20:37:58.910887 [info] Service Discovery enabled. Trying to load module.
2025-08-30 20:37:58.917587 [info] Service Discovery module loaded.
2025-08-30 20:37:58.918110 [info] Application(service-sample, 1277) is initialized (11, 100).
2025-08-30 20:37:58.919478 [info] Starting vsomeip application "service-sample" (1277) using 2 threads I/O nice 255
2025-08-30 20:37:58.920481 [info] main dispatch thread id from application: 1277 (service-sample) is: 7a37dfbfc640 TID: 29194
2025-08-30 20:37:58.920835 [info] shutdown thread id from application: 1277 (service-sample) is: 7a37df3fb640 TID: 29195
2025-08-30 20:37:58.920775 [info] Client [1277] routes unicast:127.0.0.1, netmask:255.255.255.0
2025-08-30 20:37:58.922517 [info] create_local_server: Listening @ /tmp/vsomeip-1277
2025-08-30 20:37:58.922856 [info] Watchdog is disabled!
2025-08-30 20:37:58.922852 [info] OFFER(1277): [1234.5678:0.1] (true)
Successfully Registered Service!
Waiting for calls... (Abort with CTRL+C)
2025-08-30 20:37:58.923588 [info] io thread id from application: 1277 (service-sample) is: 7a37e03fd640 TID: 29193
2025-08-30 20:37:58.923643 [info] io thread id from application: 1277 (service-sample) is: 7a37de3f9640 TID: 29197
2025-08-30 20:37:58.924244 [info] vSomeIP 3.4.10 | (default)
2025-08-30 20:37:58.924610 [info] Network interface "lo" state changed: up
2025-08-30 20:38:08.926091 [info] vSomeIP 3.4.10 | (default)
2025-08-30 20:38:11.336986 [info] Application/Client 1343 is registering.
2025-08-30 20:38:11.337652 [info] Client [1277] is connecting to [1343] at /tmp/vsomeip-1343
2025-08-30 20:38:11.339433 [info] REGISTERED_ACK(1343)
2025-08-30 20:38:11.433755 [info] REQUEST(1343): [1234.5678:0.4294967295]
sayHello('World'): 'Hello World!'
sayHello('World'): 'Hello World!'
sayHello('World'): 'Hello World!'
sayHello('World'): 'Hello World!'
sayHello('World'): 'Hello World!'
sayHello('World'): 'Hello World!'
sayHello('World'): 'Hello World!'
```

客户端可以看到类似打印
```
[CAPI][INFO] Loading configuration file '/home/vsomeip_demo/capicxx-core-tools/CommonAPI-Examples/E01HelloWorld/commonapi4someip.ini'
[CAPI][INFO] Using default binding 'someip'
[CAPI][INFO] Using default shared library folder '/usr/local/lib/commonapi'
[CAPI][DEBUG] Loading library for local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld proxy.
[CAPI][INFO] Loading configuration file /etc//commonapi-someip.ini
[CAPI][DEBUG] Added address mapping: local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld <--> [1234.5678(0.1)]
[CAPI][VERBOSE] Registering function for creating "commonapi.examples.E01HelloWorld:v0_1" proxy.
[CAPI][INFO] Registering function for creating "commonapi.examples.E01HelloWorld:v0_1" stub adapter.
[CAPI][DEBUG] Loading interface library "libE01HelloWorld-someip.so" succeeded.
[CAPI][VERBOSE] Creating proxy for "local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld"
2025-08-30 20:38:11.328411 [info] Using configuration file: "/home/vsomeip_demo/config/vsomeip-local.json".
2025-08-30 20:38:11.330330 [info] Parsed vsomeip configuration in 0ms
2025-08-30 20:38:11.330622 [info] Configuration module loaded.
2025-08-30 20:38:11.330733 [info] Security disabled!
2025-08-30 20:38:11.330816 [info] Initializing vsomeip (3.4.10) application "client-sample".
2025-08-30 20:38:11.330876 [info] Instantiating routing manager [Proxy].
2025-08-30 20:38:11.331232 [info] Client [1343] is connecting to [0] at /tmp/vsomeip-0
2025-08-30 20:38:11.331315 [info] Application(client-sample, 1343) is initialized (11, 100).
Checking availability!
2025-08-30 20:38:11.332833 [info] Starting vsomeip application "client-sample" (1343) using 2 threads I/O nice 255
2025-08-30 20:38:11.333728 [info] main dispatch thread id from application: 1343 (client-sample) is: 7740411fd640 TID: 29251
2025-08-30 20:38:11.333960 [info] shutdown thread id from application: 1343 (client-sample) is: 7740409fc640 TID: 29252
2025-08-30 20:38:11.334867 [info] io thread id from application: 1343 (client-sample) is: 7740419fe640 TID: 29250
2025-08-30 20:38:11.334950 [info] io thread id from application: 1343 (client-sample) is: 774033fff640 TID: 29253
2025-08-30 20:38:11.336373 [info] create_local_server: Listening @ /tmp/vsomeip-1343
2025-08-30 20:38:11.336631 [info] Client 1343 (client-sample) successfully connected to routing  ~> registering..
2025-08-30 20:38:11.336709 [info] Registering to routing manager @ vsomeip-0
2025-08-30 20:38:11.338978 [info] Application/Client 1343 (client-sample) is registered.
2025-08-30 20:38:11.434751 [info] ON_AVAILABLE(1343): [1234.5678:0.1]
Available...
2025-08-30 20:38:11.436637 [info] Client [1343] is connecting to [1277] at /tmp/vsomeip-1277
[CAPI][DEBUG] Message sent: SenderID: 1234 - ClientID: 4931, SessionID: 1
Got message: 'Hello World!'
[CAPI][DEBUG] Message sent: SenderID: 1234 - ClientID: 4931, SessionID: 2
Got message: 'Hello World!'
[CAPI][DEBUG] Message sent: SenderID: 1234 - ClientID: 4931, SessionID: 3
Got message: 'Hello World!'
[CAPI][DEBUG] Message sent: SenderID: 1234 - ClientID: 4931, SessionID: 4
Got message: 'Hello World!'
[CAPI][DEBUG] Message sent: SenderID: 1234 - ClientID: 4931, SessionID: 5
Got message: 'Hello World!'
[CAPI][DEBUG] Message sent: SenderID: 1234 - ClientID: 4931, SessionID: 6
Got message: 'Hello World!'
[CAPI][DEBUG] Message sent: SenderID: 1234 - ClientID: 4931, SessionID: 7
Got message: 'Hello World!'
```


