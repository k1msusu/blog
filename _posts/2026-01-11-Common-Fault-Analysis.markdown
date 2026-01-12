---
layout: post
title:  "Common fault analysis"
date:   2026-01-12 13:13:22 +0800
categories: jekyll update
---
## 运维事故训练
### 一、网站很慢，系统没挂
```
# 事故类型：性能退化
# 人为制造事故：
后端接口增加响应时间 sleep(2)
并发请求(ab / hey / curl)
# 观测点
---nginx
request_time 上升
upstream_request_time 上升
二者接近
---prometheus
接口 P95 、 P99 延迟上升
QPS 稳定
---grafana
延迟曲线变厚
# 排查路径
看 grafana 延迟趋势
看 nginx 的日志字段
对比 request_time / upstream_request_time
# 关键判断
慢在后端，不在 nginx ，性能问题不等于故障
# 常见误判
重启 nginx 扩容 / 前端实例
```

### 二、redis 挂了，系统开始抽风
```
# 事故类型：依赖组件失效
# 人为制造事故：
停止 redis 服务，redis 服务设置极小
# 现象：
接口耗时明显增加
错误日志集中爆发
Mysql QPS 飙升
# 观测点
--- ELK
redis 连接异常
timeout / connection refused
--- Prometheus
MySQL 实例 Down
MySQL QPS 异常增长
# 判断关键
redis 是缓冲剂 不是数据库
# 处置措施
缓存降级策略
redis 提前告警触发
熔断 / fallback
```

### 三、MySQL 慢查询，系统没死但是特别卡
```
# 事故类型：数据库性能瓶颈
# 人为制造事故：
无索引 sql
并发访问
# 现象：
接口耗时上升
MySQL CPU 使用率升高
慢查询日志增长
# 排查路径
grafana 看 DB 延迟
kibana 查慢接口
slow query log 定位 sql
# 关键判断
nginx 没有问题
应用没有问题
sql 有问题
# 常见错误
盲目扩容
只看 QPS 不看延迟
```


### 四、Elasticsearch 把自己写死了
```
# 事故类型：监控系统自损
# 人为制造任务：
日志级别过高
不设置 ILM
长期高写入
# 现象：
ES 磁盘接近满载
kibana 查询变慢
写入延迟上升
# 观测点
---zabbix
磁盘写入告警，使用率
---ES
写入失败
shard 不可用
# 关键判断
监控系统不是无限的
# 处理措施
ILM
日志分级
定期清理
```
### 五、告警风暴，值班人员崩溃
```
# 事故类型：
告警失效
# 人为制造事故：
redis down
MySQL 慢
网络抖动
# 现象：
多系统同时告警
手机被轰炸
# 问题本质:
症状告警太多
根因告警太少
# 优化方向:
告警分级
告警抑制
根因优先
```
### 六、事故复盘
```
# 目标：防火 > 救火
# 复盘 SOP
事故现象：（用户感知、影响范围）
时间线：（指标、日志、告警）
根因分析：（技术、设计）
改进措施：（短期、长期）
# 价值
```