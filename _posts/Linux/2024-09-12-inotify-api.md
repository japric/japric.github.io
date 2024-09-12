---
title: 使用inotify来监控系统文件
date: 2024-09-12 17:20:00 +0800
categories: [Tools]
tags: [cpp,linux,system]
---

## 1 前言

对于linux系统的文件,如果需要监控其文件状态是否出现了增、删、改,在2.6.13或更新的linux kenerl版本上,我们可以用linux系统中提供的`inotify`api 来实现

详细的api说明可以参考: [man-pages](https://man7.org/linux/man-pages/man7/inotify.7.html)

## 2 应用实现


### 2.1 主要逻辑


关键的函数有三个

```
inotify_init()
inotify_add_watch()
inotify_rm_watch()
```


我们直接从代码实现来看api的使用

1.初始化inotify并得到文件描述符`fd`

```
    int fd = inotify_init();
    if (fd == -1) {
        std::cerr << "Failed to initialize inotify" << std::endl;
        return;
    }
```

2.添加需要监控的文件/文件夹路径 得到监控文件描述符`wd`

```
#define INOTIFY_FOLDER_MASK (IN_CREATE | IN_MODIFY | IN_DELETE)
    std::string path = "/home/xx/"; // 可以是一个目录
    // std::string path = "/home/xx/test.txt" // 也可以是一个文件
    int wd = inotify_add_watch(fd, path.c_str(), INOTIFY_FOLDER_MASK);
    if (wd == -1) {
        std::cerr << "Failed to add watch to [" << path << "]" << ", error:" + std::to_string(errno) << std::endl;
        return;
    }
```

3.通过`read`等待inotify事件唤醒, 并处理`inotify_event`

```
        char buf[1024];
        int len = read(fd, buf, sizeof(buf));
        if (len == -1) {
            if (errno == EINTR) {
                std::cerr << "Interrupted by signal" << std::endl;
            }
            break;
        }
        std::cout << "read ok, len:" << len << std::endl;
        for (int i = 0; i < len;) {
            struct inotify_event *event = (struct inotify_event *)&buf[i];
            i += sizeof(struct inotify_event) + event->len;
            if (event->mask & IN_CREATE) {
                std::cout << "File created: " << event->name << std::endl;
            } else if (event->mask & IN_MODIFY) {
                std::cout << "File modified: " << event->name << std::endl;
            } else if (event->mask & IN_DELETE) {
                std::cout << "File deleted: " << event->name << std::endl;
            }
        }
```

4.`inotify_rm_watch` 移除监控
```
    if (fd == -1) {
        return;
    }
    if (wd != -1) {
        inotify_rm_watch(fd, wd);
    }
    close(fd);
```

### 2.2 注意事项

在实际场景中,我们通过step3拿到了`inotify_event`，如何判断这是否是我们监控的目标文件，这需要根据step2中监控的path本身是文件还是目录来针对性处理

#### 2.2.1 inotify path为文件的处理

检查event中的wd是否和step2中拿到的一致

```
event->wd == wd
```
注意这种情况下拿到的`event->name`实际上是空的


#### 2.2.2 inotify path为目录的处理

检查`event->name`
```
event->name == targetName // targetName即为你这个目录中你要看的文件名 例如test.txt
```

### 2.3 逻辑优化

实际业务中可以把这些处理封装到一个`FileMonitor`类当中，对于变动文件的后处理也可以作为`std::functional`的`Callback`灵活添加实现

```
enum MONI_TYPE {
    MONI_FILE,
    MONI_FOLDER,
    MONI_END,
};
class FileMonitor {
public:
    FileMonitor();
    FileMonitor(std::string str);
    ~FileMonitor();
    void monitor();
    void registerHandle(std::function<void(std::string)>);
    void setTargetFileName(std::string name);
    static void stopRun();
private:
    int fd;
    int wd;
    std::string path;
    std::string targetfileName;
    MONI_TYPE moniType;
    static bool run;
    void init();
    void exit();
    void checkType();
    void addDeviceInotify();
    bool match(std::string, int);
    std::string getRealPath();
    std::function<void(std::string)> callBack;
};
```

自定义Callback, 例如发现目标文件变动后我需要读取它
```
void CallBack(std::string str) 
{
    std::string line;
    std::ifstream ifs(str);
    if (!ifs.good()) {
        std::cerr << "Failed to open file: " << str << std::endl;
        return;
    }
    // just read first for test
    if (std::getline(ifs, line)) {
        std::cout << "Read line: " << line << std::endl;
    }
}

int main()
{
    std::string pathStr = "/home/xx/";
    FileMonitor monitor(pathStr);
    monitor.registerHandle(CallBack);
}
```

文件变动后调用Callback
```
if (event->mask & IN_MODIFY) {
    std::cout << "File modified: " << event->name << std::endl;
    if (event->wd == wd) {
        if (callBack) {
            callBack(path);
        } else {
            std::cerr << "No handle function registered" << std::endl;
        }
    }
} 
```


一份笔者的完整的实现可以参考：
[FileMonitor](https://github.com/japric/CppTools/tree/main/Cases/FileMonitor)



## 3 拓展

对于仅监控单个文件变动的场景,上述代码即可满足需求.
当然对于更复杂的场景我们可以结合另外的系统api和库来实现

### 3.1 监控多个文件变动并处理

[linux 系统调用 inotify & epoll](https://blog.csdn.net/u012719256/article/details/52944001)


### 3.2 监控文件变动并异步处理

[利用 Boost.Asio 实现异步 inotify 事件监听](https://blog.xzr.moe/archives/473/)



