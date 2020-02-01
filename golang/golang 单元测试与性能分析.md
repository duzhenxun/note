## golang 单元测试与性能分析
平时我们写了的一些方法，想测试时一般在main包中的main函数中去调用我们写好的函数，这样测试不是很专业。golang自带test工具非常好用，我们可以手动写测试代码，也可以在ide中使用快捷键先创建,我们使用下面的例子来说一下 代码测试，性能压测，性能分析等。

**例子 demo.go**
```golang
package demo

import "time"
//字符串长度
func strCount(str string) int  {
	var l int
	l = len([]rune(str))
	return l
}
//计算函数
func algorithm(a int,b int) int {
	time.Sleep(time.Millisecond*100)
	return a+b
}

type Demo struct {

}
// 注释
//	e.g. t.Test1(123)
func(d *Demo) Test1(v int)int{
	return v+v
}
```
### 一、测试结果 
我们要知道测试的函数需要传什么参数，应返回值是什么参数。按需写测试的cases。文件命名要以_test.go结尾，文件中的函数要以Test开头。使用testing.T 下面的方法，使用这个包的相关方法即可。
**demo_test.go**
#### 1. 表格数据测试
```golang
func Test_strCount(t *testing.T) {
	type args struct {
		str string
	}
	tests := []struct {
		name string
		args args
		want int
	}{
		// TODO: Add test cases.
		{"name1", args{"阿杜zhenxun"}, 9},
		{"name2", args{"ado"}, 3},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := strCount(tt.args.str); got != tt.want {
				t.Errorf("strCount() = %v, want %v", got, tt.want)
			}
		})
	}
}

```
**IDE中运行结果**
```shell
=== RUN   Test_strCount
--- PASS: Test_strCount (0.00s)
=== RUN   Test_strCount/name1
    --- PASS: Test_strCount/name1 (0.00s)
=== RUN   Test_strCount/name2
    --- PASS: Test_strCount/name2 (0.00s)
PASS
```
如果你不想手写你也可以使用golang ide的开发工具直接生成此代码。将光标放到你想测试的方法上面，mac按 command+n ，windown 应是ctrl+n.会出现以下图片所示：
![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/31/1580444613879-1580444613881.png)

#### 2. 简单例子测试
```golang
// 测试例子
func ExampleDemo_Test1() {
	d := Demo{}
	fmt.Println(d.Test1(11))
	fmt.Println(d.Test1(22))

	// Output:
	// 11
	// 44

}

```
**IDE中运行结果**
```shell
=== RUN   ExampleDemo_Test1
--- FAIL: ExampleDemo_Test1 (0.00s)
got:
22
44
want:
11
44
```
**注:** 在命名行下可以直接运行 go test 即可看到结果，其它相关命令查看 ==go help test==

### 二、代码覆盖率
在golang的ide中运行时选 "... with Coverage" 可以显示代码覆盖率
或是在命令行下执行如下2条命令在html中查看覆盖率
> go test -coverprofile=c.out

> go tool cover -html=c.out

### 三、测试性能
性能测试我们需要将函数命名以 Benchmark开头，使用testing.B结构体下面的方法

**demo_test.go**
```golang
package demo

import (
	"testing"
)

func Benchmark_algorithm(b *testing.B) {
	args1 := 1
	args2 := 2
	want := 3
	b.Run("name1", func(bb *testing.B) {
		for i := 0; i < 10; i++ {
			if got := algorithm(args1, args2); got != want {
				b.Errorf("algorithm() = %v, want %v", got, want)
			}
		}
	})
}

```
**运行结果**
可以使用 go test -bench . -count=10 命令指定执行10次
```shell
➜  测试 git:(master) ✗ go test -bench . -count=10   
goos: darwin
goarch: amd64
pkg: go-demo/基础知识/测试
Benchmark_algorithm/name1-8                    1        1043681294 ns/op
Benchmark_algorithm/name1-8                    1        1044113890 ns/op
Benchmark_algorithm/name1-8                    1        1044500717 ns/op
Benchmark_algorithm/name1-8                    1        1044609517 ns/op
Benchmark_algorithm/name1-8                    1        1044535417 ns/op
Benchmark_algorithm/name1-8                    1        1044608038 ns/op
Benchmark_algorithm/name1-8                    1        1044479081 ns/op
Benchmark_algorithm/name1-8                    1        1044309623 ns/op
Benchmark_algorithm/name1-8                    1        1044603758 ns/op
Benchmark_algorithm/name1-8                    1        1044389802 ns/op
PASS
ok      go-demo/基础知识/测试   10.456s

```
因为我们在函数中休眠了0.1秒。每次测试我们调用10次此函数，执行结果发现每次1秒多一点。10次一共10秒左右。



### 四、生成代码性能PDF
先对strCount写一个性能分析代码，使用go tool pprof 生成pdf文件
```golang
func Benchmark_strCount(b *testing.B){
	s:="阿杜ado"
	for i:=0;i<25;i++{
		s = s+s
	}
	//b.Logf("len(s) = %d",len(s))
	b.ResetTimer() //这里将生成字符串的时间去除
	for i:=0;i<b.N;i++{
		strCount(s)
	}
}
```

**命令**
1. 先使用命令 go test -bench . -cpuprofile cpu.out 输出文件cpu.out
2. 再运行 go tool pprof cpu.out 安提示输入web

> 注：如无法输出，请先安装graphviz
graphviz官方网址 http://www.graphviz.org/download

**执行结果**
```shell
 ➜  测试 git:(master) ✗ go test -bench . -cpuprofile cpu.out
goos: darwin
goarch: amd64
pkg: go-demo/基础知识/测试
Benchmark_strCount-8           2         705669566 ns/op
PASS
ok      go-demo/基础知识/测试   2.824s

➜  测试 git:(master) ✗ go tool pprof cpu.out               
Type: cpu
Time: Jan 31, 2020 at 2:15am (CST)
Duration: 2.77s, Total samples = 2.64s (95.28%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) pdf
Generating report in profile001.pdf

```
查看 profile001.pdf 发现  ==runtime decoderune 1.32s (50.97%)==，说明我们在decoderune 中用时较长。

![title](https://raw.githubusercontent.com/xs25cn/images/master/note/2020/01/31/1580413145298-1580413145305.png)

