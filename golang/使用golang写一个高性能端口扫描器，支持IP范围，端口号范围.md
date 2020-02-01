功能：可以快速扫描指定端口范围，ip地址范围。将扫描结果保存到本地！先来体验一下运行后的效果：
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146322090-1579146322095.png)

![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/16/1579146348365-1579146348370.png)

### 帮助信息
```
scanPort -h 
Options:
  -h    帮助信息
  -ip string
        ip地址 例如:-ip 192.168.0.1-255 或直接输入域名 xs25.cn (default "127.0.0.1")
  -n int
        进程数 例如:-n 10 (default 100)
  -p string
        端口号范围 例如:-p 80,81,88-1000 (default "80")
  -path string
        日志地址 例如:-path log (default "log")
  -t int
        超时时长(毫秒) 例如:-t 200 (default 200)
```
### 例1：指定端口号扫描，如我们要扫描xs25.cn这台服务的开放端口，使用1000个协程进行
scanport -p 80,81,88-3306 -ip xs25.cn -n 1000

### 例2：指定IP范围扫描,如我们扫描 192.168.0.1-255 网段的端口 80-10000
scanPort -ip 192.168.0.1-255 -p 80-10000

#### 注：程序扫描完后开放端口放在log目录中，如想更改目录名请加 -path 参数来指定



### 具体代码如下
详细代码见：[https://github.com/xs25cn/scanPort](https://github.com/xs25cn/scanPort)
main.go 文件：
```golang
package main
import (
	"flag"
	"fmt"
	"os"
	"github.com/xs25cn/scanPort/lib"
	"github.com/xs25cn/scanPort/scan"
	"time"
)

var (
	startTime = time.Now()
	ip        = flag.String("ip", "127.0.0.1", "ip地址 例如:-ip=192.168.0.1-255 或直接输入域名 xs25.cn")
	port      = flag.String("p", "80-1000", "端口号范围 例如:-p=80,81,88-1000")
	path      = flag.String("path", "log", "日志地址 例如:-path=log")
	timeout   = flag.Int("t", 200, "超时时长(毫秒) 例如:-t=200")
	process   = flag.Int("n", 100, "进程数 例如:-n=10")
	h         = flag.Bool("h", false, "帮助信息")
)
//go run main.go -h
func main() {
	flag.Parse()
	//帮助信息
	if *h == true {
		lib.Usage("scanPort version: scanPort/1.10.0\n Usage: scanPort [-h] [-ip ip地址] [-n 进程数] [-p 端口号范围] [-t 超时时长] [-path 日志保存路径]\n\nOptions:\n")
		return
	}

	fmt.Printf("========== Start %v ip:%v,port:%v ==================== \n", time.Now().Format("2006-01-02 15:04:05"), *ip, *port)

	//创建目录
	lib.Mkdir(*path)

	//初始化
	scanIP := scan.ScanIp{
		Debug:   true,
		Timeout: *timeout,
		Process: *process,
	}
	ips, err := scanIP.GetAllIp(*ip)
	if err != nil {
		fmt.Println(err.Error())
		return
	}
	//扫所有的ip
	fileName := *path + "/" + *ip + "_port.txt"
	for i := 0; i < len(ips); i++ {
		ports := scanIP.GetIpOpenPort(ips[i], *port)
		if len(ports) > 0 {
			f, err := os.OpenFile(fileName, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
			if err != nil {
				if err := f.Close(); err != nil {
					fmt.Println(err)
				}
				continue
			}
			var str = fmt.Sprintf("%v ip:%v,开放端口:%v \n", time.Now().Format("2006-01-02 15:04:05"), ips[i], ports)
			if _, err := f.WriteString(str); err != nil {
				if err := f.Close(); err != nil {
					fmt.Println(err)
				}
				continue
			}
		}
	}
	fmt.Printf("========== End %v 总执行时长：%.2fs ================ \n", time.Now().Format("2006-01-02 15:04:05"), time.Since(startTime).Seconds())

}
```
scan.go 文件
````golang
package scan

import (
	"errors"
	"fmt"
	"math"
	"net"
	"os"
	"strconv"
	"strings"
	"sync"
	"time"
)

//ip 扫描
type ScanIp struct {
	Debug   bool
	Timeout int
	Process int
}

//获取开放端口号
func (s *ScanIp) GetIpOpenPort(ip string, port string) []int {
	var (
		total     int
		pageCount int
		num       int
		openPorts []int
		mutex     sync.Mutex
	)
	ports, _ := s.getAllPort(port)
	total = len(ports)
	if total < s.Process {
		pageCount = total
	} else {
		pageCount = s.Process
	}
	num = int(math.Ceil(float64(total) / float64(pageCount)))

	s.sendLog(fmt.Sprintf("%v 【%v】需要扫描端口总数:%v 个，总协程:%v 个，每个协程处理:%v 个，超时时间:%v毫秒", time.Now().Format("2006-01-02 15:04:05"), ip, total, pageCount, num, s.Timeout))
	start := time.Now()
	all := map[int][]int{}
	for i := 1; i <= pageCount; i++ {
		for j := 0; j < num; j++ {
			tmp := (i-1)*num + j
			if tmp < total {
				all[i] = append(all[i], ports[tmp])
			}
		}
	}

	wg := sync.WaitGroup{}
	for k, v := range all {
		wg.Add(1)
		go func(value []int, key int) {
			defer wg.Done()
			var tmpPorts []int
			for i := 0; i < len(value); i++ {
				opened := s.isOpen(ip, value[i])
				if opened {
					tmpPorts = append(tmpPorts, value[i])
				}
			}
			mutex.Lock()
			openPorts = append(openPorts, tmpPorts...)
			mutex.Unlock()
			if len(tmpPorts) > 0 {
				s.sendLog(fmt.Sprintf("%v 【%v】协程%v 执行完成，时长： %.3fs，开放端口： %v", time.Now().Format("2006-01-02 15:04:05"), ip, key, time.Since(start).Seconds(), tmpPorts))
			}
		}(v, k)
	}
	wg.Wait()

	s.sendLog(fmt.Sprintf("%v 【%v】扫描结束，执行时长%.3fs , 所有开放的端口:%v", time.Now().Format("2006-01-02 15:04:05"), ip, time.Since(start).Seconds(), openPorts))
	time.Sleep(time.Second * 1)
	return openPorts
}

//获取所有ip
func (s *ScanIp) GetAllIp(ip string) ([]string, error) {
	var (
		ips []string
	)

	ipTmp := strings.Split(ip, "-")
	firstIp, err := net.ResolveIPAddr("ip", ipTmp[0])
	if err != nil {
		return ips, errors.New(ipTmp[0] + "域名解析失败" + err.Error())
	}
	if net.ParseIP(firstIp.String()) == nil {
		return ips, errors.New(ipTmp[0] + " ip地址有误~")
	}
	//域名转化成ip再塞回去
	ipTmp[0] = firstIp.String()
	ips = append(ips, ipTmp[0]) //最少有一个ip地址

	if len(ipTmp) == 2 {
		//以切割第一段ip取到最后一位
		ipTmp2 := strings.Split(ipTmp[0], ".")
		startIp, _ := strconv.Atoi(ipTmp2[3])
		endIp, err := strconv.Atoi(ipTmp[1])
		if err != nil || endIp < startIp {
			endIp = startIp
		}
		if endIp > 255 {
			endIp = 255
		}
		totalIp := endIp - startIp + 1
		for i := 1; i < totalIp; i++ {
			ips = append(ips, fmt.Sprintf("%s.%s.%s.%d", ipTmp2[0], ipTmp2[1], ipTmp2[2], startIp+i))
		}
	}
	return ips, nil
}



//记录日志
func (s *ScanIp) sendLog(str string) {
	if s.Debug == true {
		fmt.Println(str)
	}
}


//获取所有端口
func (s *ScanIp) getAllPort(port string) ([]int, error) {
	var ports []int
	//处理 ","号 如 80,81,88 或 80,88-100
	portArr := strings.Split(strings.Trim(port, ","), ",")
	for _, v := range portArr {
		portArr2 := strings.Split(strings.Trim(v, "-"), "-")
		startPort, err := s.filterPort(portArr2[0])
		if err != nil {
			continue
		}
		//第一个端口先添加
		ports = append(ports, startPort)
		if len(portArr2) > 1 {
			//添加第一个后面的所有端口
			endPort, _ := s.filterPort(portArr2[1])
			if endPort > startPort {
				for i := 1; i <= endPort-startPort; i++ {
					ports = append(ports, startPort+i)
				}
			}
		}
	}
	//去重复
	ports = s.arrayUnique(ports)

	return ports, nil
}

//端口合法性过滤
func (s *ScanIp) filterPort(str string) (int, error) {
	port, err := strconv.Atoi(str)
	if err != nil {
		return 0, err
	}
	if port < 1 || port > 65535 {
		return 0, errors.New("端口号范围超出")
	}
	return port, nil
}



//查看端口号是否打开
func (s *ScanIp) isOpen(ip string, port int) bool {
	conn, err := net.DialTimeout("tcp", fmt.Sprintf("%s:%d", ip, port), time.Millisecond*time.Duration(s.Timeout))
	if err != nil {
		if strings.Contains(err.Error(),"too many open files"){
			fmt.Println("连接数超出系统限制！"+err.Error())
			os.Exit(1)
		}
		return false
	}
	_ = conn.Close()
	return true
}

//数组去重
func (s *ScanIp) arrayUnique(arr []int) []int {
	var newArr []int
	for i := 0; i < len(arr); i++ {
		repeat := false
		for j := i + 1; j < len(arr); j++ {
			if arr[i] == arr[j] {
				repeat = true
				break
			}
		}
		if !repeat {
			newArr = append(newArr, arr[i])
		}
	}
	return newArr
}

````
详细代码见：[https://github.com/xs25cn/scanPort](https://github.com/xs25cn/scanPort)

