### 对比
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582610005666-1582610005708.png)

### etcd 支撑
- 服务发现
- 集群状态存储
- 配置同步
- 集群状态存储
- 配置同步
- 分布式锁
### 传统存储模型

单点存储
调用者不可用时或能导致同步延迟
### etcd原理
#### 抽屉理论 大多数
#### etcd与Raft的关系
- Raft是强一致的集群日志同步算法
- etcd是一个分布式KV存储
- etcd利用raft算法在集群中同步key-value
### quorum模型
集群需要2N+1个节点
当leader复制给2N+1个节点后本地提交，返回客户端
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582610351731-1582610351736.png)



aaa