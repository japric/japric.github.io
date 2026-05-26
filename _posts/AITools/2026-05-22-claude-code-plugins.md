---
title: Claude Code的skills
date: 2026-05-21 10:00:00 +0800
categories: [AITools]
tags: [claude, ai, tools]
---

## 1 前言

Claude Code 支持通过skills扩展其能力, 通过插件可以让其更好用和强大. 本文旨在记录和推荐笔者使用的skills

其中涉及到的一些非官方skills，在文中同时提供安装命令


## 2 插件推荐

### 2.1 通用推荐

#### superpowers(offical) ★★★★★

> Core skills library for Claude Code: TDD, debugging, collaboration patterns, and proven techniques

如果必须装一个skills, superpowers

#### claude-hud ★★★★★

[jarrodwatts/claude-hud](https://github.com/jarrodwatts/claude-hud)


> /plugin marketplace add jarrodwatts/claude-hud
>
> /plugin install claude-hud
>
> /claude-hud:setup



给你的claude code装上"抬头显示", 方便查看`context用了多少`，`当前使用了什么模型`，`子任务在进行还是已经完成了`




#### planning-with-files ★★★★☆

[OthmanAdi/planning-with-files](https://github.com/OthmanAdi/planning-with-files)

> /plugin marketplace add OthmanAdi/planning-with-files
>
> /plugin install planning-with-files



llm模型的context有限制，遇到复杂问题避免不了的会`compact`和丢失信息，**planning-with-files**帮助claude code会话将复杂问题进行详细涉及和计划

在会话中通过提取关键信息、任务进度、计划 分别记录到三个文件中，将关键上下文落盘，结合任务计划完成后清除context的操作，是的llm模型的context持续保持在一个健康的低使用率状态. 三个文件的存在同时帮助用户在退出会话后可以随时找回上一次的关键信息和进度

```
findings.md
progress.md
task_plan.md
```

#### skill-creator ★★★☆

便捷的创建一个新的skills


### 2.2 coding

#### code-review(offical) ★★★★

#### clangd-lsp(offical) ★★★★

C/C++ language server (clangd) for code intelligence

语言服务器插件，对于c++代码能一定程度上从语法语句的角度 帮助aiagent在暴力grep之外能更高效的分析代码

类似的还有其他语言的插件

```
gopls-lsp
pyright-lsp
```

#### andrej-karpathy-skills ★★★★

[andrej-karpathy-skills](https://github.com/vtroisWhite/andrej-karpathy-skills/blob/main/README.zh.md)

> /plugin marketplace add forrestchang/andrej-karpathy-skills
>
> /plugin install andrej-karpathy-skills@karpathy-skills

把andrej karpathy在工程实践中的经验总结成的四条经典原则 用于指导aiagent规范计划和编码行为

| 原则 | 解决的问题 |
| --- | --- |
| 编码前先思考 | 错误假设、隐藏困惑、缺少权衡 |
| 简单优先 | 过度复杂、抽象臃肿 |
| 外科手术式修改 | 无关改动、修改了不该碰的代码 |
| 目标驱动执行 | 通过测试优先和可验证成功标准来提高执行杠杆 |


### 2.3 办公

#### document-skills ★★★★☆

帮助aiagent更高效的处理办公文档，可以生成、修改PPT/表格/DOC
