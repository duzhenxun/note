### 新建proto文件
```GO

syntax = "proto3";
//生成go文件 
//protoc --go_out=plugins=grpc:. *.proto

package hello;

service HelloService {
    rpc Fun1 (Request) returns (Response);
    rpc Fun2 (Request) returns (Response);
}

message Request {
    string name = 1;
}

message Response{
    string message =1;
}
```

### client端

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"go-demo/grpc/proto/hello"
	"google.golang.org/grpc"
	"time"
)

//go run client/hello/main.go -addr=127.0.0.1:5001 -name=asdfasdf

var addr = flag.String("addr", "127.0.0.1:9080", "register address")
var name = flag.String("name", "duzhenxun", "要发送的名称")


func main() {
	flag.Parse()
	auth := Authentication{
		appKey:    "duzhenxun",
		appSecret: "password",
	}
	conn, err := grpc.Dial(*addr, grpc.WithInsecure(), grpc.WithPerRPCCredentials(&auth))
	if err != nil {
		//log.Fatal(err)
	}
	defer conn.Close()

	//使用服务
	client := hello.NewHelloServiceClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)
	defer cancel()

	r, err := client.Fun1(ctx, &hello.Request{Name: *name})
	if err != nil {
		fmt.Printf("%v %s\n", time.Now().Format("2006-01-02 15:04:05"), err.Error())
	}
	fmt.Printf("%v %s\n", time.Now().Format("2006-01-02 15:04:05"), r.Message)

	//无需认证
	r2, err := client.Fun2(ctx, &hello.Request{Name: *name})
	if err != nil {
		fmt.Printf("%v %s\n", time.Now().Format("2006-01-02 15:04:05"), err.Error())
	}
	fmt.Printf("%v %s\n", time.Now().Format("2006-01-02 15:04:05"), r2.Message)

	//循环请求
	ticker := time.NewTicker(time.Second * 2)
	for range ticker.C {
		client := hello.NewHelloServiceClient(conn)
		ctx, cancel := context.WithTimeout(context.Background(), time.Second*2)
		defer cancel()
		r, _ := client.Fun1(ctx, &hello.Request{Name: *name})
		fmt.Printf("%v %s\n", time.Now().Format("2006-01-02 15:04:05"), r.Message)
	}

}



type Authentication struct {
	appKey    string
	appSecret string
}
func (a *Authentication) GetRequestMetadata(context.Context, ...string) (
	map[string]string, error,
) {
	return map[string]string{"app_key": a.appKey, "app_secret": a.appSecret}, nil
}
func (a *Authentication) RequireTransportSecurity() bool {
	return false
}

```

### server 端首先在server端要开启反射

```go
package main

import (
	"context"
	"errors"
	"flag"
	"fmt"
	"go-demo/grpc/proto/hello"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/metadata"
	"google.golang.org/grpc/status"
	"google.golang.org/grpc/reflection"
	"log"
	"net"
)

var (
	port = flag.Int("port", 5000, "listening port")
)

func main() {
	//解析传入参数
	flag.Parse()

	//注册可用服务,服务中的fun1需要token验证,fun2可以直接访问
	grpcServer := grpc.NewServer()
	hello.RegisterHelloServiceServer(grpcServer, &helloService{})

	//监听端口
	log.Printf("starting  service at %d", *port)
	lis, err := net.Listen("tcp", fmt.Sprintf("0.0.0.0:%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	// Register reflection service on gRPC server.
	reflection.Register(grpcServer)
	
	grpcServer.Serve(lis)
}

// used to implement hello.HelloServiceServer.
type helloService struct {
}

//需要token认证
func (this *helloService) Fun1(ctx context.Context, in *hello.Request) (*hello.Response, error) {
	auth := Auth{}
	if err := auth.Check(ctx); err != nil {
		return &hello.Response{Message: err.Error()}, nil
	}
	//设置时间防止客户端已断开,服务端还在傻傻的执行
	//https://book.eddycjy.com/golang/grpc/deadlines.html
	if ctx.Err()==context.Canceled{
		return nil, errors.New("客户端已断开")
	}
	fmt.Printf("fun1 name:%v\n",in.Name)
	return &hello.Response{Message: "fun1 hello " + in.Name}, nil
}

//直接可以访问
func (this *helloService) Fun2(ctx context.Context, in *hello.Request) (*hello.Response, error) {

	fmt.Printf("fun2 name:%v\n",in.Name)
	return &hello.Response{Message: "fun2 hello " + in.Name}, nil
}

type Auth struct {
	appKey    string
	appSecret string
}

func (a *Auth) Check(ctx context.Context) error {
	md, ok := metadata.FromIncomingContext(ctx)

	if !ok {
		return status.Errorf(codes.Unauthenticated, "metadata.FromIncomingContext err")
	}
	var (
		appKey    string
		appSecret string
	)
	if value, ok := md["app_key"]; ok {
		appKey = value[0]
	}
	if value, ok := md["app_secret"]; ok {
		appSecret = value[0]
	}

	if appKey != a.GetAppKey() || appSecret != a.GetAppSecret() {
		return errors.New("Token有误!")
	}

	return nil
}

func (a *Auth) GetAppKey() string {
	return "duzhenxun"
}

func (a *Auth) GetAppSecret() string {
	return "password"
}

```
#### 功能测试

➜  ~ grpcurl -plaintext 127.0.0.1:9080 list
grpc.reflection.v1alpha.ServerReflection
hello.HelloService
➜  ~ grpcurl -plaintext 127.0.0.1:9080 list hello.HelloService
hello.HelloService.Fun1
hello.HelloService.Fun2
➜  ~ grpcurl -plaintext 127.0.0.1:9080 describe hello.HelloService.Fun1
hello.HelloService.Fun1 is a method:
rpc Fun1 ( .hello.Request ) returns ( .hello.Response );
➜  ~ grpcurl -plaintext 127.0.0.1:9080 describe hello.Request
hello.Request is a message:
message Request {
  string name = 1;
}
➜  ~ grpcurl -plaintext 127.0.0.1:9080 describe hello.HelloService.Fun2
hello.HelloService.Fun2 is a method:
rpc Fun2 ( .hello.Request ) returns ( .hello.Response );
➜  ~ grpcurl -plaintext -d '{"name": "gopher"}' 127.0.0.1:9080 hello.HelloService.Fun1
{
  "message": "Token有误!"
}
➜  ~ grpcurl -plaintext -d '{"name": "gopher"}' 127.0.0.1:9080 hello.HelloService.Fun2
{
  "message": "fun2 hello gopher"
}
➜  grpcurl -plaintext -authority duzhenxun -H app_key:duzhenxun -H app_secret:password -d '{"name":"duzhenxun"}' 127.0.0.1:9080 hello.HelloService.Fun1

➜  grpcurl -v -plaintext -authority=xxx  -d '{"sql_query":"id=456"}'   feature1-service-uwms.fat.n.com:8000 wms_truck_task.TruckTaskService.GetTruckTask

 ➜  grpcurl -plaintext -d '{"sql_query":"id=1"}' service-uwms.fat.n.com:8000  wms_repository.RepositoryService.GetRepository
{
  "code": 1,
  "data": {
    "id": 1,
    "name": "合肥仓",
    "provinceId": 1,
    "cityId": 101,
    "address": "站前路1号",
    "longitude": "117.316957",
    "latitude": "31.885144",
    "repositoryType": 2,
    "status": 2,
    "mastername": "v-chenjingsheng",
    "updateTime": "2019-09-20 16:56:58",
    "createTime": "1980-01-01 00:00:00"
  }
}

