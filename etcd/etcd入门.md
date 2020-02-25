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
### 三、实践任务
- [ ] 搭建etcd,熟悉命令行操作
- [ ] 使用go调用etcd的put/get/delete/lease/watch方法
- [ ] 使用txn事务功能，实现分布式乐观锁

#### 1、命令行使用etcd
```shell
etcdctl put /crontab/jobs/job1 job1
etcdctl put /crontab/jobs/job2 job2
etcdctl get /crontab/jobs/job1
etcdctl get /crontab/jobs/ --prefix
etcdctl delete /crontab/jobs/job1
etcdctl del /crontab/jobs/job1
etcdctl get /crontab/jobs/job1
//再开一个终端进行监听
etcdctl watch "/crontab/jobs/" --prefix

etcdctl put /crontab/jobs/job1 job11
etcdctl put /crontab/jobs/job1 job11111
//对key修改后，另一个终端会监听到变化
```
#### 2、使用go调用etcd

##### 简单添加与获取代码
```golang
package main

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/clientv3"
	"log"
	"time"
)
//连接etcd,设置key,获取key
func main(){
	var(
		config clientv3.Config
		client *clientv3.Client
		err error
		kv clientv3.KV
		putResp * clientv3.PutResponse
		getResp * clientv3.GetResponse
	)
	config = clientv3.Config{
		Endpoints:            []string{"localhost:2379"},
		DialTimeout:          5*time.Second,
	}

	if client,err=clientv3.New(config);err!=nil{
		log.Println(err.Error())
		return
	}
	defer client.Close()

	kv = clientv3.NewKV(client)
	//创建一个key
	job1:="/cron/job1"
	if putResp,err = kv.Put(context.TODO(),job1,"job1,ado",clientv3.WithPrevKV());err!=nil{
		fmt.Println(err)
	}else{
		fmt.Println(putResp.Header.Revision)
		//获取上次一的值
		//fmt.Println(string(putResp.PrevKv.Value))
	}

	//获取key 值
	if getResp,err=kv.Get(context.TODO(),job1);err!=nil{
		fmt.Println(err)
	}else{
		fmt.Println(job1+" 的值是:",getResp.Kvs)
	}


}

```
运行结果
```shell
21
/cron/job1 的值是: [key:"/cron/job1" create_revision:20 mod_revision:21 version:2 value:"job1,ado" ]

```
##### 接前缀获取
```golang
package main

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/clientv3"
	"time"
)

func main()  {
	var(
		client *clientv3.Client
		err error
		kv clientv3.KV
		getResp *clientv3.GetResponse
	)

	if client,err = clientv3.New(clientv3.Config{
		Endpoints:            []string{"localhost:2379"},
		DialTimeout:          5*time.Second,
	});err!=nil{
		fmt.Println(err)
	}

	kv = clientv3.NewKV(client)

	if getResp,err = kv.Get(context.TODO(),"/cron/jobs/",clientv3.WithPrefix());err!=nil{
		fmt.Println(err)
	}else{
		//总个数
		fmt.Println(getResp.Count)
		//分别打出所有的key,value
		for k,v:=range getResp.Kvs{
			fmt.Println(k,string(v.Key),string(v.Value))
		}
	}
}

```
运行结果
```shell
2
0 /cron/jobs/job1 job1,ado
1 /cron/jobs/job2 job2,zhangsa
```

##### 

