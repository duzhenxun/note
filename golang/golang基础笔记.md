# golang 基础知识
## 一、协程 goroutine
### 1.1 协程的特点 
- 轻量级“级程”
- ==非抢占式==多任务处理，由协程主动交出控制权
- 编译器/解释器/虚拟机层面的多任务
- 多个协程可能在一个或多个线程上运行
- 任何函数加上go就能送给调度器运行
- 调度器在合适的点进行切换

### 1.2 goroutine 可能的切换点
- I/O,select
- channel
- runtime.Gosched()
- 等待锁
- 函数调用

### 1.3 技巧
> 手动交出控制权
runtime.Gosched() 

>使用 -race 来检测数据访问冲突
go run -race goroutine.go

## 二、channel 通道

## 测试用例
