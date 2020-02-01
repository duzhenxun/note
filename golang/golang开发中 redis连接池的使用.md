作为一个phper,第一次听到连接池还有点蒙圈，转golang开发后连接池的概念会经常使用。
### 一、连接池是什么
连接池是什么？一个服务端资源的连接数量都是有限的，每次初始化时他建一定数量的连接，先把所有连接存起来，谁要用则从里面取，用完后放回去。如果超出连接池容量，要是排队等着或么直接丢弃。

比如我们做开发中常用的mysq,redis,php-fpm的配置
1，redis服务端设置
maxclients 最大连接数设置
2， mysql服务端设置 
max_connections 最大连接数
3，PHP-FPM 服务端设置
max_children 最大子进程数
start_servers 起始进程数

我们golang开发时连接redis用到自己设计的连接池概念，想要达到的效果是什么？
1，最大的连接数，最大是为了控制整个系统连接redis服务端的总数。超过个数量不再与reids进行连接。
2，最大的空闲连接数，最大空闲是想在没有redis操作时还可以保持的连接个数，这些连接会一直保持与redis服务端的连接（当然如果redis服务端设置了timeout非0，或是你在代码中手动设置了空闲连接超时间除外）。

### 二、如何使用Redis连接池
关于go的redis包github里常用的有两个：
github.com/gomodule/redigo/redis 
github.com/go-redis/redis 

“github.com/go-redis/redis” 这个包里的连接池使用有些模糊，作为一个golang新手不太建议使用这个包来做连接池的使用。
##### 例子1：go-redis/redis 这个包的简单使用
```golang
package main
import (
	"fmt"
	"github.com/go-redis/redis"
	"time"
)

func main() {
	var addr = "127.0.0.1:6379"
	var password = ""

	c := redis.NewClient(&redis.Options{
		Addr:     addr,
		Password: password,
	})
	p, err := c.Ping().Result()
	if err != nil {
		fmt.Println("redis kill")
	}
	fmt.Println(p)
	c.Do("SET", "key", "duzhenxun")
	rs := c.Do("GET", "key").Val()
	fmt.Println(rs)
	c.Close()
}

```

##### 例子2: “github.com/gomodule/redigo/redis” 这个包中有连接池的Api，非常好用
```golang
package main
import (
	"fmt"
	redigo "github.com/gomodule/redigo/redis"
	"time"
)

func main() {
	var addr = "127.0.0.1:6379"
	var password = ""

	pool := PoolInitRedis(addr, password)
	c1 := pool.Get()
	c2:=pool.Get()
	c3:=pool.Get()
	c4:=pool.Get()
	c5:=pool.Get()
	fmt.Println(c,c2,c3,c4,c5)
	time.Sleep(time.Second * 5)//redis一共有多少个连接？？
	c1.Close()
	c2.Close()
	c3.Close()
	c4.Close()
	c5.Close()
	time.Sleep(time.Second*5) //redis一共有多少个连接？？

	//下次是怎么取出来的？？
	b1:=pool.Get()
	b2:=pool.Get()
	b3:=pool.Get()
	fmt.Println(b1,b2,b3)
	time.Sleep(time.Second*5)
	b1.Close()
	b2.Close()
	b3.Close()

	//redis目前一共有多少个连接？？
	for{
		fmt.Println("主程序运行中....")
		time.Sleep(time.Second*1) 
	}
}

// redis pool
func PoolInitRedis(server string, password string) *redigo.Pool {
	return &redigo.Pool{
		MaxIdle:     2,//空闲数
		IdleTimeout: 240 * time.Second,
		MaxActive:   3,//最大数
		Dial: func() (redigo.Conn, error) {
			c, err := redigo.Dial("tcp", server)
			if err != nil {
				return nil, err
			}
			if password != "" {
				if _, err := c.Do("AUTH", password); err != nil {
					c.Close()
					return nil, err
				}
			}
			return c, err
		},
		TestOnBorrow: func(c redigo.Conn, t time.Time) error {
			_, err := c.Do("PING")
			return err
		},
	}
}

```

### 三、参数配置说明
MaxIdle:最大空闲连接数，没有redis操作进依然可以保持这个连接数量
MaxActive:最大连接数。同一时间最多有这么多的连接

一般go程序运行时选设置redis连接池的初始化。假如我们设置MaxIdle:2，MaxActive:3时。
连接时：调用pool.Get()时，先从MaxIdle中取出可用连接，如果失败，则看当前设置的MaxActive是否有超出最大数，没有超出则创建一个新的连接。
断开时：调用c.Close() 后，看当前连接数，如果比MaxIdle设置的数量大，则关闭redis连接。反之将连接放回到连接池中。大体如下流程图所示：
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579145998994-1579145998998.png)
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146025395-1579146025399.png)

### 四、分析与实验
例子2中，程序启动时休息的5秒里，我们先看一下redis中的当前的连接数，当前连接数只有1个，这个是当前cli的连接。说明go还没有和redis进行连接。我们可以通过info clients 或client list来查看当前redis连接情况。
```shell
redis-cli
//当前客户端连接数
127.0.0.1:6379> info clients
# Clients
connected_clients:1
```
我们get了5次，想是取出5个连接，但我们所设置的MaxActive是3，那redis最大连接数只有3个：
```golang
c1 :=pool.Get()
c2:=pool.Get()
c3:=pool.Get()
c4:=pool.Get()
c5:=pool.Get()
```
可以在redis中使用 info clients 命令查看
```sh
127.0.0.1:6379> info clients
# Clients
connected_clients:4 （这里显示4个是因为本身自己的cli连接redis时会有一个连接)
```

调用关闭接口，发现虽然是关闭了5次。但最终还有2个连接没有关闭。因为我们设置的MaxIdle是2
```language
c1.Close()
c2.Close()
c3.Close()
c4.Close()
c5.Close()
```
可以在redis中使用 info clients 命令查看
```sh
127.0.0.1:6379> info clients
# Clients
connected_clients:3 （这里显示3个是因为本身自己的cli连接redis时会有一个连接)
```


#### 以上就是我对redis连接池的使用笔记，由于初次尝试使用可能有的地方讲的不对，还请多多指教。微信号：5552123