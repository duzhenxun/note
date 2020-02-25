这里我主要练习使用go调用etcd的put/get/delete/lease/watch方法
### 1、put,get使用
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
### 2、get 前缀获取
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

### 3、delete删除操作
```golang
package main

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/clientv3"
	"log"
	"time"
)

func main()  {
	var(
		config clientv3.Config
		client *clientv3.Client
		err error
		kv clientv3.KV
		delResp * clientv3.DeleteResponse
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
	if delResp,err = kv.Delete(context.TODO(),"/cron/job1",clientv3.WithPrevKV());err!=nil{
		fmt.Println(err)
		return
	}
	if len(delResp.PrevKvs)!=0{
		for k,v:=range delResp.PrevKvs{
			fmt.Println(k,string(v.Key),string(v.Value))
		}
	}

}

```

### 4、lease租约
```golang
package main

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/clientv3"
	"log"
	"time"
)

func main()  {
	var(
		client *clientv3.Client
		err error
	)

	if client,err = clientv3.New(clientv3.Config{
		Endpoints:            []string{"localhost:2379"},
		DialTimeout:          5*time.Second,
	});err!=nil{
		fmt.Println(err)
	}

	//申请一个lease租约
	lease := clientv3.NewLease(client)
	//申请一个10秒的租约
	leaseGrantResp,err := lease.Grant(context.TODO(),10)
	if err!=nil{
		fmt.Println(err)
		return
	}
	//put一个kv,让它与租约联系起来。实现10秒后自动过期
	leaseId:=leaseGrantResp.ID
	kv:=clientv3.NewKV(client)
	lockKey:="/cron/lock/job1"
	putResp,err := kv.Put(context.TODO(),lockKey,"",clientv3.WithLease(leaseId))
	if err!=nil{
		fmt.Println(err)
		return
	}
	log.Println(lockKey+" 写入成功",putResp.Header.Revision)

	//测试代码，定期查看一下key是否过期
	for{
		getResp,err:=kv.Get(context.TODO(),lockKey)
		if err!=nil{
			fmt.Println(err)
			return
		}
		if getResp.Count==0{
			log.Println(lockKey+" 过期了")
			break
		}
		log.Println(lockKey+" 还没有过期")
		time.Sleep(2*time.Second)
	}
}

```
运行结果：
```shell
2020/02/25 19:34:13 /cron/lock/job1 写入成功 24
2020/02/25 19:34:13 /cron/lock/job1 还没有过期
2020/02/25 19:34:15 /cron/lock/job1 还没有过期
2020/02/25 19:34:17 /cron/lock/job1 还没有过期
2020/02/25 19:34:19 /cron/lock/job1 还没有过期
2020/02/25 19:34:21 /cron/lock/job1 还没有过期
2020/02/25 19:34:23 /cron/lock/job1 还没有过期
2020/02/25 19:34:25 /cron/lock/job1 过期了
```
### lease keepAlive自动续约处理
```golang
package main

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/clientv3"
	"log"
	"time"
)

func main() {
	var (
		client *clientv3.Client
		err    error
	)

	if client, err = clientv3.New(clientv3.Config{
		Endpoints:   []string{"localhost:2379"},
		DialTimeout: 5 * time.Second,
	}); err != nil {
		fmt.Println(err)
	}

	//申请一个lease租约
	lease := clientv3.NewLease(client)
	//申请一个10秒的租约
	leaseGrantResp, err := lease.Grant(context.TODO(), 10)
	if err != nil {
		fmt.Println(err)
		return
	}

	leaseId := leaseGrantResp.ID
	keepRespChan, err := lease.KeepAlive(context.TODO(), leaseId)
	if err != nil {
		fmt.Println(err)
		return
	}
	go func() {
		for {
			select {
			case keepResp := <-keepRespChan:
				if keepResp == nil {
					log.Println("租约失效.服务器原因或其它的原因...")
					goto END
				} else {
					log.Println("收到续租应答", keepResp.ID)
				}
			}
		}
	END:
	}()

	//设置一个key,租约使用上面的id
	kv := clientv3.NewKV(client)
	lockKey := "/cron/lock/job1"
	putResp, err := kv.Put(context.TODO(), lockKey, "", clientv3.WithLease(leaseId))
	if err != nil {
		fmt.Println(err)
		return
	}
	log.Println(lockKey+" 写入成功", putResp.Header.Revision)

	//测试代码，定期查看一下key是否过期
	for {
		getResp, err := kv.Get(context.TODO(), lockKey)
		if err != nil {
			fmt.Println(err)
			return
		}
		if getResp.Count == 0 {
			log.Println(lockKey + " 过期了")
			break
		}
		log.Println(lockKey + " 还没有过期")
		time.Sleep(2 * time.Second)
	}
}

```
结果：
```shell
2020/02/25 20:50:04 收到续租应答 7587844626267763152
2020/02/25 20:50:04 /cron/lock/job1 写入成功 217
2020/02/25 20:50:04 /cron/lock/job1 还没有过期
2020/02/25 20:50:06 /cron/lock/job1 还没有过期
2020/02/25 20:50:07 收到续租应答 7587844626267763152
2020/02/25 20:50:08 /cron/lock/job1 还没有过期
2020/02/25 20:50:10 /cron/lock/job1 还没有过期
2020/02/25 20:50:11 收到续租应答 7587844626267763152

```


### 6、Watch 监听

```golang
package main

import (
	"context"
	"fmt"
	"github.com/coreos/etcd/clientv3"
	"github.com/coreos/etcd/mvcc/mvccpb"
	"log"
	"time"
)

func main() {
	var (
		config  clientv3.Config
		client  *clientv3.Client
		err     error
		kv      clientv3.KV
		getResp *clientv3.GetResponse
		watchRespChan clientv3.WatchChan
	)
	config = clientv3.Config{
		Endpoints:   []string{"localhost:2379"},
		DialTimeout: 5 * time.Second,
	}

	if client, err = clientv3.New(config); err != nil {
		log.Println(err.Error())
		return
	}
	defer client.Close()
	kv = clientv3.NewKV(client)
	go func() {
		for{
			kv.Put(context.TODO(), "/cron/jobs/ado", "watch ado")
			kv.Delete(context.TODO(),"/cron/jobs/ado")
			time.Sleep(1*time.Second)
		}

	}()

	if getResp, err = kv.Get(context.TODO(), "/cron/jobs/ado"); err != nil {
		fmt.Println(err)
		return
	}

	if len(getResp.Kvs) != 0 {
		fmt.Println(string(getResp.Kvs[0].Value))
	}

	//当前etcd集群事务ID，单调递增的
	watchStartRevision := getResp.Header.Revision + 1

	//创建一个监听器
	watcher := clientv3.NewWatcher(client)
	//返回一个chan

	//watchRespChan = watcher.Watch(context.TODO(), "/cron/jobs/ado", clientv3.WithRev(watchStartRevision))

	//TODO 这里加一个测试代码，
	//===== 模拟5秒后关闭watch监听 START
	ctx,cancelFunc:=context.WithCancel(context.TODO())
	time.AfterFunc(5*time.Second, func() {
		cancelFunc()
	})
	watchRespChan = watcher.Watch(ctx,"/cron/jobs/ado", clientv3.WithRev(watchStartRevision))
	//===== 模拟5秒后关闭watch监听 END

	//循环chan中的数据
	for watchResp := range watchRespChan {
		for _, event := range watchResp.Events {
			switch event.Type {
			case mvccpb.PUT:
				fmt.Println("修改为：", string(event.Kv.Value), "revsion:", event.Kv.CreateRevision, event.Kv.ModRevision)
			case mvccpb.DELETE:
				fmt.Println("删除了", "Revision:", event.Kv.ModRevision)
			}
		}
	}
}

```
结果：
```shell
watch ado
删除了 Revision: 208
修改为： watch ado revsion: 209 209
删除了 Revision: 210
修改为： watch ado revsion: 211 211
删除了 Revision: 212
修改为： watch ado revsion: 213 213
删除了 Revision: 214
修改为： watch ado revsion: 215 215
删除了 Revision: 216

Process finished with exit code 0


```

