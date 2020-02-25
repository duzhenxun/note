## 一、相关知识
### 1、传统crontab的缺点
- 机器故障，任务停止调度
- 任务数量多，单机硬件资源耗尽
- 需要人工去机器上配cron
- 任务状态不方便查看
### 2、分布式架构 核心要素
- 调度器：需要高可用，确保不会因单点故障停止调度
- 执行器：需要扩展性，提供大量任务的并行处理能力
### 3、常见开源调度架构
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582642571045-1582642571053.png)

### 4、伪分布式设计
- 分布式网格环境不可靠，RPC异常属于常态
- Master下发任务RPC异常，导致Master与Worer状态不一致
- Worker上报任务RPC异常，导致Master状态信息落后
- 状态不一致：Master下发任务给work1异常，实际work1收到并执行开始
- 并发执行：Master重试下发任务给work2,结果work2与work1同时执行一个任务
- 状态丢失：Master更新zookeeper中任务状态异常，此时Master宕机切换Standby,任务仍旧处理旧状态
- 分布系统中，异常是无处不在的，凡需要经过网络的操作，都可能出现异常
- 将应用状态放在存储中，必然会出现内存与DB状态不一致
- 应用直接利用raft管理状态，可以确保最终一致，但成本太高

### 5、CAP理论（常用于分布式存储）
- C 一致性，定入后立即读到新值
- A 可用性，通常保障最终一致
- P 分区容错性，分布式必须面对网络分区

### 6、BASE理论（常用于应用架构）
- 基本可用：损失部份可用性，保证整体可用性
- 软状态：允许状态同步延迟，不会影响系统即可
- 最终一致：经过一时间后，系统能够达到一致性

### 7、如何架构
- 架构简化，减少有状态服务，秉承最终一致
- 架构折中，确保异常可以被程序自我修复

### 8、整体架构
- 利用etcd同步全量任务列表到所有worker节点
- 每个worker独立调度全量任务，无需与master产生直接RPC
- 各个Worker利用分布式锁抢占，解决并发调度相同任务的问题
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582643440107-1582643440113.png)

## 二、功能设计
### 1、Master功能
- 任务管理HTTP接口：任务的增删改查
  - 写入到etcd中 
  -  /cron/jobs/任务名 -> {name:任务名，command:shell命令,cronExpr:cron表达式}
- 任务日志HTTP接口：查看任务执行日志
  - 写入到db中
  - {jobName:任务名，command:shell命令，err:执行报错,output:执行输出,startTime:开始时间，endTime:结束时间}
- 任务控制HTTP接口：强制结束任务接口
  - etcd /cron/killer/任务名
  - master 将要结束任务写到 /cron/killer/下面，worker端监听到进行杀死操作
- 实现web管理页面，前后端分离

### 2、Worker功能
- 任务同步：监听etcd中 /cron/jobs/目录变化
- 任务调度：基于cron表式达计算，触发过期任务
- 任务执行：协程池并发执行多任务，基于etcd分布式锁抢占
- 日志保存：捕获任务执行输出，保存到DB中
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582644546178-1582644546191.png)

#### 2.1 监听协程
- 利用watch API,监听/cron/jobs 和/cron/killer/ 目标的变化
- 将变化事件通过channel推送给调度协程，更新内存中的任务信息
#### 2.2 调度协程
- 监听任务变更event,更新内存维护的任务列表
- 检查任务cron表达式，扫描到期任务，交给执行协程运行
- 监听任务控制event,强制中断正在执行中的子进程
- 监听任务执行result,更新内存中任务状态，投递执行日志
#### 2.3 执行协程
- 在ETCD中抢占分布式乐观锁： /cron/lock/任务名
- 抢占成功则通过Command类执行shell任务
- 捕获Command输出并且等待子进程结束，执行结果投递给调度协程
#### 2.4日志协程
- 监听调度发来的执行日志，放入一个batch中
- 对新的batch启动定时器，超时未满自动提交
- 若batch被放满，那么立即提交，并取消自动提交定时器