Cron基本格式
|分|时|日|月|周|shell命令
|-|-|-|-|-|-|
|*/5|*|*|*|*|echo hello >/tmp/x.log|
|1-5|*|*|*|*|echo /usr/bin/python /data/x.py|
|0|10,22|*|*|*|echo hello|tail -1|

go开源Cronexpr库
Parse() 解析与校验Cron表达式
Next() 根据当前时间，计算下一次调度时间

### 例子1
模拟一下cron调用
```golang
package main

import (
	"fmt"
	"github.com/gorhill/cronexpr"
	"time"
)

func main()  {
	var(
		expr *cronexpr.Expression
		err error
		now time.Time
		nextTime time.Time
	)
	if expr,err = cronexpr.Parse("*/5 * * * * * *");err!=nil{
		fmt.Println(err)
		return
	}

	now = time.Now()
	//下次调度时间
	nextTime = expr.Next(now)
	//多长时间后运行
	time.AfterFunc(nextTime.Sub(now), func() {
		fmt.Println("被调度了:",nextTime)
	})

	time.Sleep(time.Second*5)

}

```

### 例子2
模拟多个cron调用

```golang
package main

import (
	"fmt"
	"github.com/gorhill/cronexpr"
	"time"
)

type CronJob struct {
	expr     *cronexpr.Expression
	nextTime time.Time
}

func main() {

	var (
		cronJob       *CronJob
		expr          *cronexpr.Expression //表达式
		now           time.Time
		scheduleTable map[string]*CronJob //任务表
	)
	scheduleTable = make(map[string]*CronJob)
	//当前时间
	now = time.Now()
	//Cron表达式
	expr = cronexpr.MustParse("*/5 * * * * * *")
	cronJob = &CronJob{
		expr:     expr,
		nextTime: expr.Next(now),
	}
	//任务注册到任务表中
	scheduleTable["job1"] = cronJob

	expr = cronexpr.MustParse("*/5 * * * * * *")
	cronJob = &CronJob{
		expr:     expr,
		nextTime: expr.Next(now),
	}
	//任务注册到任务表中
	scheduleTable["job2"] = cronJob

	// 需要有1个调度协程，它定时检查所有的Cron任务，谁过期就执行谁
	go func() {
		var (
			jobName string
			cronJob *CronJob
			now     time.Time
		)
		//定时看任务表
		for {
			now = time.Now()
			for jobName, cronJob = range scheduleTable {
				if cronJob.nextTime.Before(now) || cronJob.nextTime.Equal(now) {
					//启动一个协程，执行这个任务
					go func(jobName string) {
						fmt.Println("执行:", jobName)
					}(jobName)

					//计算下一次执行时间
					cronJob.nextTime = cronJob.expr.Next(now)
					fmt.Println(jobName, "下次执行时间:", cronJob.nextTime)
				}
			}

			//这里模拟一下休眠
			select {
			case <-time.NewTimer(time.Millisecond*100).C: //100毫秒可读，返回

			}
		}
	}()

	time.Sleep(time.Second*100)
}

```
执行结果
```shell
执行: job1
job1 下次执行时间: 2020-02-05 17:08:55 +0800 CST
job2 下次执行时间: 2020-02-05 17:08:55 +0800 CST
执行: job2
执行: job1
job1 下次执行时间: 2020-02-05 17:09:00 +0800 CST
job2 下次执行时间: 2020-02-05 17:09:00 +0800 CST


```

