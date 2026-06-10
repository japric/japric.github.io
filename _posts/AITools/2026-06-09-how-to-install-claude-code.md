---
title: 如何安装Claude Code和配置
date: 2026-06-09 10:00:00 +0800
categories: [AITools]
tags: [claude, ai, tools]
---

# 1 前言

`工欲善其事，必先利其器`，想要使用好Claude Code，第一步就是安装好它。本文详细介绍如何安装Claude Code和如何配置第三方api模型

文章主要是针对笔者的ubuntu 24.04系统, 仅供参考

claude官方提供了几种安装方式，不过针对于国内网络环境，并非所有方式都适用。截至到文章撰写的26年06 推荐可以**直接参考2.1章节**，配置可以**直接参考3.1章节**

官方文档参考 https://code.claude.com/docs/zh-CN/setup#native-install-recommended


# 2 安装步骤

选择其中一种安装方式进行安装，推荐直接使用2.1章节的方式

## 2.1 使用nodejs安装

### 2.1.1 安装nodejs

参考: https://www.runoob.com/nodejs/nodejs-install-setup.html


下载源码, 这里笔者下载的node v24

```bash
wget https://nodejs.org/dist/v24.16.0/node-v24.16.0-linux-x64.tar.xz
tar xf  node-v24.16.0-linux-x64.tar.xz       // 解压
cd node-v24.16.0-linux-x64                  // 进入解压目录
./bin/node -v                                  // 执行node命令 查看版本
```

软链接到系统路径, 注意sudo命令执行后输入密码

```bash
sudo ln -s ~/node-v24.16.0-linux-x64/bin/node /usr/local/bin/node
sudo ln -s ~/node-v24.16.0-linux-x64/bin/npm /usr/local/bin/npm
```

把路径添加到环境变量配置文件, 因为通过nodejs安装的程序会在nodejs本身的路径下，如果不做配置 本小节后文中安装完claude code后会通过命令行启动 会找不到

```bash
echo 'export PATH="~/node-v24.16.0-linux-x64/bin:$PATH"' >> ~/.bashrc
```

### 2.1.2 通过nodejs命令安装claude code


这里可以指定特定的版本 例如这里的`2.1.76`, 也可以用`npm view @anthropic-ai/claude-code versions`查看npm中可用的版本

执行安装命令

```bash
npm install -g @anthropic-ai/claude-code@2.1.76

added 2 packages, removed 1 package, and changed 1 package in 2s

2 packages are looking for funding
  run `npm fund` for details
```

安装完成之后可以打印出版本信息
```bash
claude -v
2.1.76 (Claude Code)
```


## 2.2 失败的安装方式1(native-install)

`native-install` 作为现在claude官方推荐的安装方式，通过执行一个在线的脚本来进行安装

```
curl -fsSL https://claude.ai/install.sh | bash
```

在国内网络，大概率会遇到`App unavailable in region`， claude官方做了地区限制，所以这种方式没有什么简单的好办法可以成功


## 2.3 失败的安装方式2(brew)

`homebrew` 同样是现在claude官方推荐的安装方式， ubuntu上有`linuxbrew`基本逻辑和macos上的`homebrew`类似, 这里我们以`brew`通称

### 2.3.1 安装brew

因为国内网络环境，这里推荐不使用官方的链接 而使用国内镜像的安装源

而对于国内的安装源，清华的安装源做了一个下载的排队限制，所以最终推荐的是**中科大的安装源**


配置一些环境变量并执行 brew的安装脚本命令
```
export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.ustc.edu.cn/brew.git"
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.ustc.edu.cn/homebrew-core.git"
export HOMEBREW_BOTTLE_DOMAIN="https://mirrors.ustc.edu.cn/homebrew-bottles"
export HOMEBREW_API_DOMAIN="https://mirrors.ustc.edu.cn/homebrew-bottles/api"

/bin/bash -c "$(curl -fsSL https://mirrors.ustc.edu.cn/misc/brew-install.sh)"
```

安装完成后, 再次配置环境变量 以便下次打开终端可以直接启动和使用`brew`
```
# Run these commands in your terminal to add Homebrew to your PATH:
    echo >> ~/.bashrc
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv bash)"' >> ~/.bashrc
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv bash)"
# Run these commands in your terminal to add the non-default Git remotes for Homebrew/brew and Homebrew/homebrew-core:
    echo '# Set non-default Git remotes for Homebrew/brew and Homebrew/homebrew-core.' >> ~/.bashrc
    echo 'export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.ustc.edu.cn/brew.git"' >> ~/.bashrc
    echo 'export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.ustc.edu.cn/homebrew-core.git"' >> ~/.bashrc
    export HOMEBREW_BREW_GIT_REMOTE="https://mirrors.ustc.edu.cn/brew.git"
    export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.ustc.edu.cn/homebrew-core.git"
```


### 2.3.2 通过brew安装claude code(超时)

笔者年初时候曾经使用brew成功在ubuntu系统上安装过claude code, 不过不清楚通过brew最终下载的软件地址是否做了限制，当前跑`brew install -vd --cask claude-code`最终指向超时失败


```bash
brew install -vd --cask claude-code
                                                                                               Downloaded   16.9MB/ 16.9MB
/home/linuxbrew/.linuxbrew/Homebrew/Library/Homebrew/brew.rb (Cask::CaskLoader::FromAPILoader): loading claude-code
==> Fetching downloads for: claude-code

/usr/bin/env /home/linuxbrew/.linuxbrew/Homebrew/Library/Homebrew/shims/shared/curl --disable --cookie /dev/null --globoff --show-error --user-agent Linuxbrew/5.1.14\ \(Linux\;\ x86_64\ Ubuntu\ 24.04.4\ LTS\)\ curl/8.5.0 --header Accept-Language:\ en --retry 3 --fail --location --silent --head --request GET https://downloads.claude.ai/claude-code-releases/2.1.150/linux-x64/claude
✘ Cask claude-code (2.1.150)
Error: Download failed on Cask 'claude-code' with message: Download failed: https://downloads.claude.ai/claude-code-releases/2.1.150/linux-x64/claude
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:02:30 --:--:--     0
curl: (22) The requested URL returned error: 504
Error: Kernel.exit
/home/linuxbrew/.linuxbrew/Homebrew/Library/Homebrew/cmd/install.rb:381:in 'Kernel#exit'
/home/linuxbrew/.linuxbrew/Homebrew/Library/Homebrew/cmd/install.rb:381:in 'Homebrew::Cmd::InstallCmd#run'
/home/linuxbrew/.linuxbrew/Homebrew/Library/Homebrew/brew.rb:114:in '<main>'
```



# 3 配置claude code

claude code默认启动后是需要登陆claude的注册账号来使用相应模型，不过我们可以通过添加`~/.claude/settings.json`配置来使用第三方api模型

`cc-switch`是一个便捷的图形化配置工具，可以方便的修改`~/.claude/settings.json`配置


## 3.1 安装cc-switch

`cc-switch`官方的链接可访问 [cc-switch/releases](https://github.com/farion1231/cc-switch/releases/tag/v3.16.1)

页面这里选择 https://github.com/farion1231/cc-switch/releases/download/v3.16.1/CC-Switch-v3.16.1-Linux-x86_64.deb 并下载

在ubuntu通过dpkg命令安装

```bash
sudo dpkg -i CC-Switch-v3.16.1-Linux-x86_64.deb
```

## 3.2 通过cc-switch配置claude code
详细的配置方式可以参考 [runoob -- cc-switch](https://www.runoob.com/ai-agent/cc-switch.html)
