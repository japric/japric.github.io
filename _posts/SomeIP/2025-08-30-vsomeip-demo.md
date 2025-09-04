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


