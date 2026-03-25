---
title: vsomeip commonapi 官方示例01
date: 2026-03-18 19:20:00 +0800
categories: [SOME/IP, vsomeip, commonapi]
tags: [cpp,linux,system]
---

## 1 前言

**vsomeip**是SOME/IP规范的开源实现，作为COVESA项目的一部分设计。同时COVESA开发了**CommonAPI C++**的一套库和工具，方便开发者快速定义和实现IPC/RPC通信接口并基于vsomeip进行符合SOME/IP规范的通信.

本文主要会结合COVESA官方的capicxx-core-tools仓库中的示例01进行分析，介绍接口定义和接口代码实现逻辑


## 2 准备

参考前篇文章"基于vsomeip commonapi的demo" 在本地准备好代码，并至少完成到`./gen.sh`这一步

## 3 分析介绍

### 3.1 代码结构

笔者的代码也包括各个依赖例如`capicxx-core-tools` `capicxx-someip-runtime`, 呈现为如下结构

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


官方的示例01在 `capicxx-core-tools/CommonAPI-Examples/E01HelloWorld` 路径下，关键目录结构参考

```
capicxx-core-tools/CommonAPI-Examples/E01HelloWorld
├── ...
├── commonapi4someip.ini
├── fidl
│   ├── E01HelloWorld.fidl
│   └── E01HelloWorld-SomeIP.fdepl
├── src
│   ├── E01HelloWorldClient.cpp
│   ├── E01HelloWorldService.cpp
│   ├── E01HelloWorldStubImpl.cpp
│   └── E01HelloWorldStubImpl.hpp
├── src-gen
│   ├── core
│   │   └── v0
│   │       └── commonapi
│   │           └── examples
│   │               ├── E01HelloWorld.hpp
│   │               ├── E01HelloWorldProxyBase.hpp
│   │               ├── E01HelloWorldProxy.hpp
│   │               ├── E01HelloWorldStubDefault.hpp
│   │               └── E01HelloWorldStub.hpp
│   └── someip
│       └── v0
│           └── commonapi
│               └── examples
│                   ├── E01HelloWorldSomeIPCatalog.json
│                   ├── E01HelloWorldSomeIPDeployment.cpp
│                   ├── E01HelloWorldSomeIPDeployment.hpp
│                   ├── E01HelloWorldSomeIPProxy.cpp
│                   ├── E01HelloWorldSomeIPProxy.hpp
│                   ├── E01HelloWorldSomeIPStubAdapter.cpp
│                   └── E01HelloWorldSomeIPStubAdapter.hpp
├── vsomeip-client.json
├── vsomeip-local.json
└── vsomeip-service.json
```

其中大部分文件都是仓库的示例中原始带的，除了两处生成代码目录，通过`./gen.sh`调用`commonapi_core_generator`和`commonapi_someip_generator`生成
- `E01HelloWorld/src-gen/core` 中的代码是`E01HelloWorld.fidl`对应生成
- `E01HelloWorld/src-gen/someip`中的代码是由`E01HelloWorld-SomeIP.fdepl`对应生成

主要实现的业务逻辑则在`E01HelloWorld/src/`目录下


### 3.2 E01HelloWorld代码分析

```c++
    // E01HelloWorldService.cpp
    CommonAPI::Runtime::setProperty("LogContext", "E01S");			// for android log or dlt log
    CommonAPI::Runtime::setProperty("LogApplication", "E01S");		// for android log or dlt log
    CommonAPI::Runtime::setProperty("LibraryBase", "E01HelloWorld");

    std::shared_ptr<CommonAPI::Runtime> runtime = CommonAPI::Runtime::get(); // 

    std::string domain = "local"; 							// default value: "local"
    std::string instance = "commonapi.examples.HelloWorld";	// InstanceId in E01HelloWorld-SomeIP.fdepl
    std::string connection = "service-sample";

    std::shared_ptr<E01HelloWorldStubImpl> myService = std::make_shared<E01HelloWorldStubImpl>();
    bool successfullyRegistered = runtime->registerService(domain, instance, myService, connection); // 

    while (!successfullyRegistered) {
        std::cout << "Register Service failed, trying again in 100 milliseconds..." << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        successfullyRegistered = runtime->registerService(domain, instance, myService, connection);
    }

    std::cout << "Successfully Registered Service!" << std::endl;


```

#### 3.2.1 instance 字段 & fidl/fdepl配置


其中关键的字段`instance`需要和`E01HelloWorld-SomeIP.fdepl`中的`InstanceId`一致

```
define org.genivi.commonapi.someip.deployment for provider as Service {
    instance commonapi.examples.E01HelloWorld {
        InstanceId = "commonapi.examples.HelloWorld"
```


其实本质上是对应到生成代码中的匹配字符串

```c++
// E01HelloWorld/src-gen/someip/v0/commonapi/examples/E01HelloWorldSomeIPStubAdapter.cpp
void initializeE01HelloWorldSomeIPStubAdapter() {
    CommonAPI::SomeIP::AddressTranslator::get()->insert(
        "local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld", // 由fdepl文件定义生成的字段
         0x1234, 0x5678, 0, 1);
    CommonAPI::SomeIP::Factory::get()->registerStubAdapterCreateMethod(
        "commonapi.examples.E01HelloWorld:v0_1",
        &createE01HelloWorldSomeIPStubAdapter);
}
```


`E01HelloWorldSomeIPStubAdapter.cpp`中使用的地址字符串和`fidl``fdepl`中定义的完整关系可以参考如下

```
CommonAPI::SomeIP::AddressTranslator::get()->insert(
    "local:commonapi.examples.E01HelloWorld:v0_1:commonapi.examples.HelloWorld",
    // ──┬── ───────┬──────── ──────┬────────── ──┬─ ─────────────┬────────────
    //   │         │                │             │               │
    //   │         │                │             │               └─ fdepl InstanceId
    //   │         │                │             └─ fidl version (0.1)
    //   │         │                └─ fidl interface name
    //   │         └─ fidl package
    //   └─ 默认或代码指定
     0x1234, 0x5678, 0, 1);
    // ────┬─ ────┬─ ─┬ ─┬
    //     │      │   │  └─ minor version (1)
    //     │      │   └─ major version (0)
    //     │      └─ fdepl SomeIpInstanceID (22136)
    //     └─ fdepl SomeIpServiceID (4660)

```


#### 3.2.2 connection 字段 & vsomeip.json配置


```
C++ 用户代码 (E01HelloWorldService.cpp:26)
--------
runtime->registerService(domain, instance, myService, "service-sample")
      ↓
      
CommonAPI::Runtime (Runtime.hpp:116 - template方法)
--------
registerService() 
  → 调用 registerStub(_domain, Stub_::StubInterface::getInterface(), _instance, _service, _connectionId)
      ↓
      
CommonAPI::Runtime (Runtime.cpp)
--------
registerStub(_domain, _interface, _instance, _stub, "service-sample")
  → 调用 registerStubHelper(_domain, _interface, _instance, _stub, "service-sample", false)
      ↓
      
CommonAPI::Runtime (Runtime.cpp)
--------
registerStubHelper(..., "service-sample", _useDefault)
  → 遍历 factories_ map，找到 SomeIP Factory
  → 调用 factory.second->registerStub(_domain, _interface, _instance, _stub, "service-sample")
      ↓
      
CommonAPI::SomeIP::Factory (Factory.cpp)
--------
Factory::registerStub(_domain, _interface, _instance, _stub, "service-sample")
  → 调用 getConnection("service-sample")
      ↓
      
CommonAPI::SomeIP::Factory (Factory.cpp)
--------
Factory::getConnection("service-sample")
  → 查找 connections_ map: connections_.find("service-sample")
  → 如果找到：返回已有的 Connection 对象（复用）
  → 如果未找到：创建新的
      ↓
      
CommonAPI::SomeIP::Factory (Factory.cpp)
--------
new Connection("service-sample")
  → std::make_shared<Connection>("service-sample")
      ↓
      
CommonAPI::SomeIP::Connection (Connection.cpp)
--------
Connection::Connection("service-sample")
  → 初始化成员变量
  → application_ = vsomeip::runtime::get()->create_application("service-sample")
      ↓
      
vsomeip 库 (vsomeip内部实现)
--------
vsomeip::runtime::create_application("service-sample")
  → 读取环境变量 VSOMEIP_CONFIGURATION 指定的配置文件
  → 解析 JSON 配置文件
  → 在 applications[] 数组中查找 name == "service-sample"
      ↓
      
vsomeip-service.json
--------------------
{
    "applications": [
        {
            "name": "service-sample",  ← 必须匹配！
            "id": "0x1277"             ← 使用此 application ID
        }
    ],
    "routing": "service-sample"        ← 此应用作为 routing manager
}
      ↓
      
返回路径
--------
vsomeip::application 对象
  → 返回给 Connection::application_
  → Connection 对象创建完成
  → 存入 Factory::connections_["service-sample"]
  → 用于创建 StubAdapter
  → 注册服务成功
```




