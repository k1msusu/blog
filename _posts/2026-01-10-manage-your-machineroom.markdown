---
layout : post
title:  "Mangage your machineroom"
date:   2026-01-10 14:10:10 +0800
categories: jekyll update
---
## 交换机日常维护

### 一、交换机环境要求

```
推荐温度 20 - 25
推荐湿度 40% - 70%
太热 交换机核心芯片过热，烧坏板子
太干 静电上身，轻则烧掉端口，重则整板失效】

```

```
温度计挂机柜，定期记录，空调避免冷热交替，使用加湿/除湿
```

### 二、电源/线缆的整理

```
使用稳压电源或者 UPS  
电源线路做好标记，整齐放置
定期检查是否存在热插拔风险，接触不良、电源老化
```

```
线缆不能死弯，影响传输
光纤要插紧，尽量统一线缆管理，不同业务使用不同颜色
```

### 三、软件、硬件的维护

```
硬件：定期清灰，检查风扇、指示灯、端口灯是否有异常，设备鼓包等
软件：保存配置文件，定期备份，查看资源使用率，实时查看日志监控
```

### 四、Console登陆

```
# 设备挂了，远程连接不上
1. 用 Console 连接交换机和电脑 （USB 或者 RS232 接口）
2. 打开终端工具（SecureCRT、Putty），设置波特率 9600，数据位 8 ，无校验位，停止位 1 ，无流控
3. 进入命令行界面
```

### 五、Telnet登录

```
# 缺点：明文传输
sys
user-interface vty 0 4
authentication-mode password
set password cipher YourPassword
user privilege level 15
protocol inbound telnet
# 华为，建议小型环境使用
```

### 六、SSH登录

```
# 加密版 Telnet 安全性能高
sys
stelnet server enable
local-user admin password irreversible-cipher YourPassword
local-user admin privilege level 15
local-user admin service-type ssh
aaa
authentication-scheme default
authorization-scheme default
domain default enable default-domain
user-interface vty 0 4
authentication-mode aaa
# 华为，使用 XSHELL 登录
```

### 七、常见巡检命令和异常情况介绍

```
display interface brief  # 查看端口状态
display interface GigabitEthernet 0/0/1  # 查看端口详细信息
display mac-address  # 查看 MAC 地址表
display vlan  # 查看 vlan
display version
display cpu-usage
display memory-usage
display logbuffer  # 查看日志
# 端口关闭，限速配置
# 日志级别太低，看不到关键报警
# 撕拉网线，交换机级联太深
```

### 八、交换机日常维护清单

有流程、有章法、有隐患洞察意识，建立维护体系

```
Everyday
巡检核心交换机的 CPU 、内存、端口状态
查看日志缓冲区是否有新报错
检查设备是否有异常重启、端口是否 flap
```

```
Everyweek
核查 VLAN 、接口限速、防环配置
对比备份配置、发现配置漂移
网管平台生成健康报告
```

```
Everymonth
固件设备、补丁检查是否为最新稳定版本
验证 SNMP 、SSH 、Telnet 访问策略和密码策略
确认机房环境温度、湿度达标
```

```
Everyquarter
全面配置备份、做好异地灾备同步
检查设备标签、端口标号、机柜配线整理
跑一次广播风暴模拟测试，验证保护机制
```
