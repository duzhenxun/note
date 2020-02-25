### 一、传统crontab的缺点
- 机器故障，任务停止调度
- 任务数量多，单机硬件资源耗尽
- 需要人工去机器上配cron
- 任务状态不方便查看
### 二、分布式架构 核心要素
- 调度器：需要高可用，确保不会因单点故障停止调度
- 执行器：需要扩展性，提供大量任务的并行处理能力
### 三、常见开源调度架构
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582642571045-1582642571053.png)
### 四、伪分布式设计
- 分布式网格环境不可靠，RPC异常属于常态
- Master下发任务RPC异常，导致Master与Worer状态不一致
- Worker上报任务RPC异常，导致Master状态信息落后
- 状态不一致：Master下发任务给work1异常，实际work1收到并执行开始
- 并发执行：Master重试下发任务给w