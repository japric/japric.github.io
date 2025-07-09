---
title: linuxptp部署
date: 2025-07-08 17:20:00 +0800
categories: [PTP]
tags: [cpp,linux,system,time]
mermaid: true
---

## 1 准备

下载代码

```
git clone git@github.com:richardcochran/linuxptp.git
```

编译程序

```
make -j8
```

检查网卡是否支持硬件时间戳

>  假设通过ifconfig命令可以查询到本机的网卡是eth1

```
ethtool -T eth1

```

如果支持硬件时钟 则应当可以看到这样的输出信息

`PTP Hardware Clock: 0`

否则会出现

`PTP Hardware Clock: none`

## 2 部署

### 2.0 命令参数


#### 全部参数

```
./ptp4l -h

usage: ptp4l [options]

 Delay Mechanism

 -A        Auto, starting with E2E
 -E        E2E, delay request-response (default)
 -P        P2P, peer delay mechanism

 Network Transport

 -2        IEEE 802.3
 -4        UDP IPV4 (default)
 -6        UDP IPV6

 Time Stamping

 -H        HARDWARE (default)
 -S        SOFTWARE
 -L        LEGACY HW

 Other Options

 -f [file] read configuration from 'file'
 -i [dev]  interface device to use, for example 'eth0'
           (may be specified multiple times)
 -p [dev]  Clock device to use, default auto
           (ignored for SOFTWARE/LEGACY HW time stamping)
 -s        client only synchronization mode (overrides configuration file)
 -l [num]  set the logging level to 'num'
 -m        print messages to stdout
 -q        do not print messages to the syslog
 -v        prints the software version and exits
 -h        prints this message and exits
```


#### 重要参数


- `-m` 输出日志到命令行
- `-2` 传输协议使用2层的IEEE 802.3，`-4`(默认) 使用UDP
- `-H` (默认) 使用硬件时间戳(PHC网卡时钟)， `-S`使用软件时间戳(系统时间)
- `-s` 作为client/slave, (默认) 作为master
- `-f` 读取制定配置文件中的参数



### 2.1 同步软件时间戳

> 假定这里master使用网卡eno1, slave使用网卡enp0s31f6

master(主机)

```
sudo ./ptp4l -i eno1 -m -S -2
ptp4l[27368.164]: port 1 (eno1): INITIALIZING to LISTENING on INIT_COMPLETE

```


slave(从机)

```
sudo ./ptp4l -i enp0s31f6 -m -s -S -2
ptp4l[5965923.380]: port 1 (enp0s31f6): INITIALIZING to LISTENING on INIT_COMPLETE

ptp4l[5965927.579]: port 1 (enp0s31f6): LISTENING to UNCALIBRATED on RS_SLAVE
ptp4l[5965929.957]: master offset    -325832 s0 freq   -3679 path delay    146277
ptp4l[5965930.957]: master offset    -316292 s0 freq   -3679 path delay    142442
ptp4l[5965931.957]: master offset    -362534 s0 freq   -3679 path delay    142086
ptp4l[5965932.964]: master offset    -302878 s0 freq   -3679 path delay    142086
ptp4l[5965936.966]: master offset    -275640 s0 freq   -3679 path delay    140347
ptp4l[5965945.979]: master offset    -328474 s1 freq   -3844 path delay    124087
ptp4l[5965946.980]: master offset    -112075 s2 freq  -15163 path delay    124087
ptp4l[5965946.980]: port 1 (enp0s31f6): UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
ptp4l[5965947.980]: master offset     -27752 s2 freq   -6759 path delay    124087
ptp4l[5965948.981]: master offset      22251 s2 freq   -1736 path delay    12408
```


日志中的信息


- master offset : 即PTP协议中定义的主从端时间差，单位：ns
- s0，s1，s2 : 表示时钟伺服器的不同状态，s0表示未锁定，s1表示正在同步，s2表示锁定，锁定状态表示不会再发生阶跃行同步，只是缓慢调整


### 2.2 同步硬件时间戳

> 假定这里master使用网卡enp0s31f6, slave使用网卡eno1

master(主机)
```
sudo ./ptp4l -i enp0s31f6 -m -H -2
ptp4l[5969853.086]: selected /dev/ptp0 as PTP clock
```


slave(从机)
```
sudo ./ptp4l -i eno1 -m -H -s -2
ptp4l[31718.066]: selected /dev/ptp1 as PTP clock

ptp4l[31722.600]: port 1 (eno1): LISTENING to UNCALIBRATED on RS_SLAVE
ptp4l[31724.596]: master offset -117638482769 s0 freq      -0 path delay  28064540
ptp4l[31725.596]: master offset -118387485873 s1 freq -748980915 path delay 215455496
ptp4l[31726.596]: master offset  187344962 s2 freq -561635953 path delay 215455496
ptp4l[31726.596]: port 1 (eno1): UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
ptp4l[31727.596]: master offset  374792821 s2 freq -317984606 path delay  28064540
ptp4l[31728.597]: master offset  131179966 s2 freq -449159615 path delay  28064540
ptp4l[31729.596]: master offset   -4178511 s2 freq -545164102 path delay  50911932
ptp4l[31742.597]: master offset    -555856 s2 freq -561959682 path delay    -37373
ptp4l[31745.598]: master offset      22582 s2 freq -561623862 path delay    -17033
ptp4l[31746.598]: master offset      74400 s2 freq -561565270 path delay    -17033
ptp4l[31747.598]: master offset      39354 s2 freq -561577996 path delay    -17033
ptp4l[31748.598]: master offset      21601 s2 freq -561583943 path delay    -17033
ptp4l[31749.598]: master offset       9584 s2 freq -561589479 path delay    -17033
ptp4l[31750.598]: master offset     -12384 s2 freq -561608572 path delay     -1124
ptp4l[31751.598]: master offset     -20570 s2 freq -561620473 path delay     20118
ptp4l[31752.598]: master offset      -3403 s2 freq -561609477 path delay     27533
ptp4l[31753.598]: master offset       8624 s2 freq -561598471 path delay     29225
ptp4l[31754.598]: master offset      11602 s2 freq -561592906 path delay     29225
ptp4l[31755.598]: master offset       6699 s2 freq -561594328 path delay     31244
ptp4l[31756.600]: master offset       4517 s2 freq -561594501 path delay     32153
ptp4l[31757.598]: master offset       3244 s2 freq -561594419 path delay     32153
ptp4l[31758.598]: master offset       2035 s2 freq -561594654 path delay     32114
```

### 2.3 一些补充说明

#### 查询硬件时间

同步硬件时间戳的方式改变的是slave的网卡硬件时间，不会改变slave的系统时间

可以通过编译出来的`phc_ctl`来查询该硬件时间

```
sudo ./phc_ctl /dev/ptp1 get
phc_ctl[31856.285]: clock time is 1751998409.995321853 or Wed Jul  9 02:13:29 2025
```

#### 日志时间

类似日志`ptp4l[31752.598]`中的时间，实际上是单调系统时间`CLOCK_MONOTONIC`

这一点可以从源码中看到

```c++
// print.c
void print(int level, char const *format, ...)
{

	struct timespec ts;

    // ...

	clock_gettime(CLOCK_MONOTONIC, &ts);
    // ...
}
```

#### mac address

linuxptp构造消息的目标mac address为以下两个之一
`01:1B:19:00:00:00`
`01:80:C2:00:00:0E`

从[PTP multicast address](https://www.jianshu.com/p/5e1fa643586f) 的介绍来看

- `01:80:C2:00:00:0E` 用于P2P机制下的消息，不会被设备转发(forward)
- `01:1B:19:00:00:00` 用于其他情况的消息

对于gPTP(IEEE802.1AS),其规范中仅支持P2P模式，同时要求消息的mac address使用`01:80:C2:00:00:0E`

[8021AS-2011.pdf](https://www.ida.liu.se/~sohsa65/courses/tsn-course-2021/stds/8021AS-2011.pdf)

可参考 *7.5 Differences between gPTP and PTP*, *10.4.3 Addresses*


