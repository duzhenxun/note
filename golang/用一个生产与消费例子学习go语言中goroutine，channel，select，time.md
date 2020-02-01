近最学习了一下Go语言的 goroutine（协程），channel（通道），select，time等相关的知识，通过一个例子来说一下对它们的理解与使用。我们先来看一个生产与消费的例子，最后再去细看一些理论知识，这对于一个新手来说可能会更容易理解。

### 队列生产与消费的例子

这里使用2个goroutine往n大小的通道中模拟新型冠状病毒的生产。select中的case哪个可以读取则打印出数据，每隔5秒我们来看一下生产的消息还有多少没有被打印过。

```golang
func main() {
	var t1 = makeTask("adoJob", 1000)
	var t2 = makeTask("xs25Job", 500)
	var tick = time.Tick(time.Second * 5)
	for {
		select {
		case task:=<-t1:
			log.Println(task)
		case task:=<-t2:
			log.Println(task)
		case <-tick:
			log.Println(fmt.Sprintf("队列挤压数量t1:%v个，t2:%v个", len(t1), len(t2)))
		}
		time.Sleep(time.Second * 1)
	}
}

//生产数据
func makeTask(queueName string, n int) chan string {
	ch := make(chan string, n)
	go func() {
		i := 1
		for {
			time.Sleep(time.Millisecond * time.Duration(rand.Intn(2000))) //假设生产任务占用时间
			ch <- fmt.Sprintf("%s,生产数据 %d", queueName, i)
			i++
		}
	}()
	return ch
}

```
运行后的结果：
```shell
2020/01/23 00:25:09 adoJob,生产数据 1
2020/01/23 00:25:10 xs25Job,生产数据 1
2020/01/23 00:25:11 xs25Job,生产数据 2
2020/01/23 00:25:12 adoJob,生产数据 2
2020/01/23 00:25:13 xs25Job,生产数据 3
2020/01/23 00:25:14 队列挤压数量 t1:7个，t2:3个
2020/01/23 00:25:15 adoJob,生产数据 3
2020/01/23 00:25:16 adoJob,生产数据 4
2020/01/23 00:25:17 adoJob,生产数据 5
2020/01/23 00:25:18 adoJob,生产数据 6
2020/01/23 00:25:19 adoJob,生产数据 7
2020/01/23 00:25:20 xs25Job,生产数据 4
2020/01/23 00:25:21 xs25Job,生产数据 5
2020/01/23 00:25:22 adoJob,生产数据 8
2020/01/23 00:25:23 队列挤压数量 t1:10个，t2:9个
```
以上这段代码我模拟了2个生产端，一个定时任务。

这里先对代码做一个简单的解释：
> go func() 开启一个goroutine（协程）

> ch := make(chan string, n)
创建一个string的channel（通道），指定大小则是一个非阻塞，没有指定大小则是一个阻塞通道

> <- 这个符号表示从通道里读数据或往通道中写数据。通道是先进先出原则。

> makeTask("adoJob", 1000)
 adoJob这个任务，创建一个1000大小的通道，开启一个goroutine随机时间往里写数据

> makeTask("xs25Job", 500)
xs25Job这个任务，创建一个500大小的通道，开启一个goroutine随机时间往里写数据

> time.Tick(time.Second * 5)
定时任务，每5秒运行一次，我们在这里主要是为了练习，每隔5秒钟有可能执行不到这个case。原因是多个case都满足时随机执行其中一个。

现在我们再使用goroutine与通道做一个消费端，将代码再改进一下。
```golang
func main() {
	var t1 = makeTask("adoJob", 1000)
	var t2 = makeTask("xs25Job", 1000)
	var allTask []string                  //因为我想只做一个消费端，将2个生产端生产出来的消费都扔到一起
	var tick = time.Tick(time.Second * 5) //每隔一段时间报告队列积压情况
	var workerCh = worker()

	for {
		var taskInfo string //具体任务
		var ch chan<- string
		if len(allTask) > 0 {
			taskInfo = allTask[0] //从所有任务中取出每一个
			ch = workerCh
		}
		select {
		case task := <-t1:
			allTask = append(allTask, task)
		case task := <-t2:
			allTask = append(allTask, task)
		case ch <- taskInfo: //任务详情写入到要消费工作中
			allTask = allTask[1:]
		case <-tick:
			log.Println("队列挤压数量", len(allTask))
		}
	}
}

//生产数据
func makeTask(queueName string, n int) chan string {
	ch := make(chan string, n)
	go func() {
		i := 1
		for {
			time.Sleep(time.Millisecond * time.Duration(rand.Intn(2000))) //假设生产任务占用时间
			ch <- fmt.Sprintf("%s,生产数据 %d", queueName, i)
			i++
		}
	}()
	return ch
}

//消费数据
func worker() chan<- string {
	ch := make(chan string)
	go func(tasks chan string) {
		for t := range tasks {
			time.Sleep(time.Second * 1) //假设我们每次消费任务需要花费1秒钟
			log.Printf("消费任务: %s \n", t)
		}
	}(ch)
	return ch
}

```

运行后的结果：
```shell
2020/01/23 00:28:21 消费任务: adoJob,生产数据 1 
2020/01/23 00:28:23 消费任务: xs25Job,生产数据 1 
2020/01/23 00:28:24 消费任务: adoJob,生产数据 2 
2020/01/23 00:28:25 消费任务: xs25Job,生产数据 2 
2020/01/23 00:28:25 队列挤压数量 8
2020/01/23 00:28:26 消费任务: adoJob,生产数据 3 
2020/01/23 00:28:27 消费任务: adoJob,生产数据 4 
2020/01/23 00:28:28 消费任务: adoJob,生产数据 5 
2020/01/23 00:28:29 消费任务: xs25Job,生产数据 3 
2020/01/23 00:28:30 消费任务: adoJob,生产数据 6 
2020/01/23 00:28:30 队列挤压数量 11
```
我们可以看出因生产速度快，消费速度跟不上，产生了队列挤压。这个代码例子主要是为了练习一下select上面的使用。通过这个例子的实验，对Go语言的goroutine,channel,select有了一个简单的了解。

### 理论知识
看完上面的例子，我们再来看这些枯燥的理论知识会轻松许多
#### 1、goroutine
goroutine 是一种非常轻量级的实现，可在单个进程里执行成千上万的并发任务，它是Go语言并发设计的核心。 

goroutine 其实就是线程，但是它比线程更小，十几个 goroutine 可能体现在底层就是五六个线程，而且Go语言内部也实现了goroutine 之间的内存共享, go 关键字就可以创建 goroutine。

将 go 声明放到一个需调用的函数之前，在相同地址空间调用运行这个函数，这样该函数执行时便会作为一个独立的并发线程，这种线程在Go语言中则被称为 goroutine。

如果两个或者多个 goroutine 在没有相互同步的情况下，访问某个共享的资源，比如同时对该资源进行读写时，就会处于相互竞争。可以使用sync.WaitGroup，sync.Mutex相关的包进行处理。

#### 2、channel
go语言提倡使用通信的方法代替共享内存，当一个资源需要在 goroutine 之间共享时，通道在 goroutine 之间架起了一个管道，并提供了确保同步交换数据的机制，channels是goroutine之间的通信机制，一个channels是一个通信机制，它可以让一个goroutine通过它给另一个goroutine发送值信息。

#### 3、select
select 的用法与 switch 语言非常类似，select 有比较多的限制，其中最大的一条限制就是每个 case 语句里必须是一个 IO 操作。
select用来监听和channel有关的IO操作，当 IO 操作发生时，触发相应的动作。如果有一个或多个IO操作可以完成，系统就会随机的选择一个执行。如果都没有完成有defalut分支就会选择defalut分支，如果defalut也没有，那select语句会一直阻塞，直到有一个IO操作才进行。

#### 4、time.Tick
定时任务，调用Tick函数会返回一个时间类型的channel，在调用Tick方法的过程中，必然又创建了goroutine,负责发送数据，唤醒被阻塞的定时任务。定时任务都会加入timersBucket（时间任务桶），关于time.Tick这里先不做太深入的探讨。这里我们每隔5秒会看看一下当前任务数量。

### 总结
