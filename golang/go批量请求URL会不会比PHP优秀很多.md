为了凑够3篇原创只好把以前测试代码给找出来，看看能凑够字数不？
以前使用PHP做批量抓取过淘宝的数据，公司内部的接口数据..反正挺好用。特别爽~ PHP使用的是自带函数curl_multi*

### PHP 批量请求码代演示
```php
/**
 * 批量curl请求
 * @author Zhenxun Du <5552123@qq.com>
 * @date   2017-8-9 17:08:32
 * @param array $curl_data
 * @param int $read_timeout
 * @param int $connect_timeout
 * @return array
 *
 */
function my_curl_multi($curl_data, $read_timeout = 30, $connect_timeout = 30)
{
    //加入子curl
    $mh = curl_multi_init();
    $curl_array = array();

    foreach ($curl_data as $k => $info) {
        $curl_array[$k] = curl_init($info['url']);
        curl_setopt($curl_array[$k], CURLOPT_RETURNTRANSFER, true);
        curl_setopt($curl_array[$k], CURLOPT_HEADER, 0);

        if ($read_timeout) {
            curl_setopt($curl_array[$k], CURLOPT_TIMEOUT, $read_timeout);
        }
        if ($connect_timeout) {
            curl_setopt($curl_array[$k], CURLOPT_CONNECTTIMEOUT, $connect_timeout);
        }

        if (!empty($info['headers'])) {
            curl_setopt($curl_array[$k], CURLOPT_HTTPHEADER, $info['headers']);
        }
        //发送整个body
        if (!empty($info['post_fields'])) {
            curl_setopt($curl_array[$k], CURLOPT_POSTFIELDS, $info['post_fields']);
        }

        curl_multi_add_handle($mh, $curl_array[$k]);
    }


    //执行curl
    $running = null;
    do {
        $mrc = curl_multi_exec($mh, $running);
    } while ($mrc == CURLM_CALL_MULTI_PERFORM);


    while ($running && $mrc == CURLM_OK) {
        if (curl_multi_select($mh) == -1) {
            usleep(100);
        }
        do {
            $mrc = curl_multi_exec($mh, $running);
        } while ($mrc == CURLM_CALL_MULTI_PERFORM);
    }

    //获取执行结果
    $response = [];
    foreach ($curl_array as $key => $val) {
        $response[$key] = curl_multi_getcontent($val);
    }

    //关闭子curl
    foreach ($curl_data as $key => $val) {
        curl_multi_remove_handle($mh, $curl_array[$key]);
    }

    //关闭父curl
    curl_multi_close($mh);

    return $response;
}

//使用
//curl批量获取 
$curls_data = [
['url'=>'http://www.baidu.com'],
['url'=>'http://www.58.com'],
['url'=>'http://xs25.cn'],
];
//批量curl 获得所有请求的返回信息
$multi_arr = my_curl_multi($curls_data, 180, 60)

```

相信大多数学GO的对PHP语言都比较了解。PHP的使用也比较灵活，现在我们来看一下使用GO写起来是不是更简单，效率更高效！写这文章属于是为了凑字，拼凑出一篇原创...


## GO 协程批量请求 第一种写法
```go
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"time"
)

func main()  {
	start :=time.Now()
	ch :=make(chan string)

	var urls = []string{"http://www.baidu.com",
		"http://www.qq.com",
		"http://www.58.com",
		"http://xs25.cn",
	}
	for _,url:=range urls[:len(urls)]{
		go fetch(url,ch)
	}

	for range urls[:len(urls)]{
		fmt.Println(<-ch)
	}
	fmt.Printf("%.2fs elapsed \n",time.Since(start).Seconds())

}

func fetch(url string,ch chan<- string){
	start:=time.Now()
	res,err:=http.Get(url)
	if err!=nil{
		ch <- fmt.Sprint(err)
		return
	}
	nbytes,err:=io.Copy(ioutil.Discard,res.Body)
	if err!=nil{
		ch <- fmt.Sprintf("while reading %s:%v",url,err)
		return
	}
	secs:=time.Since(start).Seconds()
	ch <- fmt.Sprintf("%.2fs %7d %s",secs,nbytes,url)

}
```
## GO 协程批量请求 第二种写法

```go
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"sync"
	"time"
)

func main() {
	var urls = []string{"http://www.baidu.com",
		"https://www.qq.com",
		"https://lf.58.com",
		"http://xs25.cn"
	}
	start := time.Now()
	wg := sync.WaitGroup{}
	wg.Add(len(urls))
	for _, val := range urls {
		go func(url string) {
			start := time.Now()
			res, err := http.Get(url)
			if err != nil {
				fmt.Println(err)
				return
			}
			nbytes, err := io.Copy(ioutil.Discard, res.Body)
			if err != nil {
				fmt.Printf("while reading %s:%v\n", url, err)
				return
			}
			fmt.Printf("%.2fs %7d %s\n", time.Since(start).Seconds(), nbytes, url)
			wg.Done()
		}(val)
	}

	wg.Wait()

	fmt.Printf("%.2fs elapsed \n", time.Since(start).Seconds())

}

```

0.12s  156965 http://www.baidu.com
0.44s  235408 http://www.qq.com
0.51s   63305 https://www.xs25.cn
0.56s  183305 https://lf.58.com


### 最后再凑几个文字
上面2种方法就是使用GO协程的批量请求方式。是不是比PHP简单了许多。
第一种方案用的是go协程+通道（chan），如果你的量比较大使用通道不太合适，所以有了第二种方案用的是go协程+sync.WaitGroup。WaitGroup 对象内部有一个计数器，最初从0开始，它有三个方法：Add(), Done(), Wait() 用来控制计数器的数量。Add(n) 把计数器设置为n ，Done() 每次把计数器-1 ，wait() 会阻塞代码的运行，直到计数器地值减为0。
