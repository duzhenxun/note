### 对比
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582610005666-1582610005708.png)

### 一、etcd 支撑
- 服务发现
- 集群状态存储
- 配置同步
- 集群状态存储
- 配置同步
- 分布式锁

### 二、etcd原理
#### 1、抽屉理论 大多数
#### 2、etcd与Raft的关系
- Raft是强一致的集群日志同步算法
- etcd是一个分布式KV存储
- etcd利用raft算法在集群中同步key-value
#### 3、quorum模型
集群需要2N+1个节点
当leader复制给2N+1个节点后本地提交，返回客户端
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/02/25/1582610351731-1582610351736.png)


#### 4、重要特性
- 底层存储按key有序排列的，可以顺序遍历
- 因为key有序，天然支技按目录结构高效遍历
   - xxx/xxx/xx 
- 支持复杂事务，提供类似if...then ...else...的事务能力
- 基于租约机制实现key的TTL过期
- mvcc多版本控制
- watch机制 监听kv变化
  - sdk监听某个key,从n版本监听。
  - watcher 推送给sdk版本后的变化
### 实践任务
- 搭建etcd