# 创建API示例
我们根据自己业务需要可以选择创建一个接口供别的系统调用。

## 基础知识

我们接口协议使用 protobuf，相关知识可以参考：

* 官方文档：https://developers.google.com/protocol-buffers/docs/proto3

本文中的 EdgeCommon 公共库，EdgeAPI是API接口库，具体的编译逻辑请参考 源码编译 一节。

## 步骤1：准备protoc工具

在使用 build.sh 编译 .proto 文件之前，你需要确保已经为 protoc 安装了对应的插件：

```bash
# install protoc-gen-go plugin
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# install protoc-gen-go-grpc plugin
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

```

如果提示网络连接超时或失败，请尝试在命令行中设置网络代理后再尝试：

`go env-w GOPROXY=https://goproxy.cn,direct`


## 步骤2：定义原型

首先我们需要在`proto`文件中定义接口的原型，接口原型定义在 `EdgeCommon/pkg/rpc/protos` 目录下，如果是在原有服务的基础上添加接口，你只需要编辑对应服务的proto文件即可，这里我们假设要开发一个新的服务HelloWorldService，用来实现根据用户输入的ID来返回信息。

在 `protos/` 目录下创建一个 `service_hello_world.proto` 文件：

```
protos/
  service_a.proto
  servie_...proto
  ...
  service_hello_world.proto    <- 我们创建的服务定义文件
```

里面写入内容：

```
syntax = "proto3";
option go_package = "./pb";

package pb;

service HelloWorldService {
	rpc SayHello(SayHelloRequest) returns (SayHelloResponse);
}

message SayHelloRequest {
	int64 id = 1;
	string name = 2;
}

message SayHelloResponse {
	string result = 1;
}
```

其中：

> syntax、 option、package选项都是固定的，不需要修改
> service 定义了服务的名称
> rpc 定义了服务中包含的方法
> message 定义了调用服务方法所需要的参数信息

* int64 和 string 是两种基本类型，完整说明请参考：https://protobuf.dev/programming-guides/proto3/#scalar
* 参数值后面的 1、2 是参数顺序，而不是参数值

服务定义好之后，需要执行编译：

```bash
# 必须到build目录下
cd EdgeCommon/build/
./build.sh
```

如果出现以下提示表示成功：

```
> ./build.sh
starting ...
ok
```

如果不成功，会有错误提示，请根据错误提示进行修改。

编译成功后，会在 EdgeCommon/pkg/rpc/pb/ 目录下生成两个文件：

```
pb/
  ...
  service_hello_world.pb.go
  service_http_access_log_grpc.pb.go
  ...

```


至此我们的接口定义已经完成，下面就可以实现这个接口了。


## 步骤3：实现接口

在 `EdgeAPI/internal/rpc/services` 目录下创建一个 `service_hello_world.go` 文件，注意这个文件名和原型定义的文件名是一致的：


```
services/
  ...
  service_hello_world.go
  ...
```

里面写入内容：


```go
package services

import (
	"context"
	"github.com/TeaOSLab/EdgeCommon/pkg/rpc/pb"
	"strconv"
)

type HelloWorldService struct {
	BaseService
}

func (this *HelloWorldService) SayHello(ctx context.Context, req *pb.SayHelloRequest) (*pb.SayHelloResponse, error) {
	// 校验用户权限
	_, _, err := this.ValidateAdminAndUser(ctx, true)
	if err != nil {
		return nil, err
	}

	// 返回信息
	return &pb.SayHelloResponse{
		Result: "Id: " + strconv.FormatInt(req.Id, 10) + ", Name: " + req.Name,
	}, nil
}

```


其中：

* HelloWorldService - 需要同原型定义保持一致，并集成我们的 BaseService 中已经定义好的功能函数
* SayHello 就是我们方法的实现，这里我们只做了一个简单的返回用户输入信息处理，这里注意首字母必须大写
* ValidateAdminAndUser - 用来校验用户权限，更多校验方法，可以参考 BaseService 中的定义
* SayHelloRequest和SayHelloResponse - 同我们原型中定义的message一致

这样我们的实现代码已经写好了，然后就可以注册接口。

## 步骤4：注册接口

在 `EdgeAPI/internal/nodes/api_node_services.go` 文件中可以注册一个接口，找到 `registerServices` 方法，在 `APINodeServicesRegister` 之前写入：

```go
...
{
	var instance = this.serviceInstance(&services.HelloWorldService{}).(*services.HelloWorldService)
	pb.RegisterHelloWorldServiceServer(server, instance)
	this.rest(instance)
}

APINodeServicesRegister(this, server)
```


其中要注意：


* `ervice.HelloWorldService` - 是我们写的服务实现代码
* `RegisterHelloWorldServiceServer` - 是根据原型自动生成的HelloWorld服务专门的注册方法

保存文件并重启EdgeAPI开发环境，如果没有提示错误的话，说明注册成功；然后你可以在别的组件中使用grpc调用此服务，也可以使用HTTP方法调用。


## 步骤5：测试接口

这里我们使用HTTP方法测试我们的服务：

先调用 APIAccessTokenService/getAPIAccessToken 获取一个Token：

```
curl -XPOST http://192.168.2.41:8004/APIAccessTokenService/getAPIAccessToken -d ' {"type":"admin", "accessKeyId":"vFxVRScklDiWP8E5", "accessKey":"ajtpfYRddMbevOdS6YRe87lsxqDfj2T6" }'
```


其中的API地址、accessKeyId、accessKey都使用你自己的设置（如果不知道怎么设置HTTP调用，请参考 API调用概述 一节）。
然后利用获取到的 token 来调用HelloWorld服务的sayHello方法：

```
curl -XPOST -H "X-Edge-Access-Token: dvQvl85OVJkZjGOSgmvnlT9WkxiG6kRuAwJIMryat3ZB59GHd84fjLpA9WkFIlNG4ZAfiE64cg8bqfQwX5bWOJjKdKf4y67xmGPSJ1KH4FlFPnsMenSkOozApj96H4JT" http://192.168.2.41:8004/HelloWorldService/sayHello -d  '{ "id":123, "name":"Lily" }'
```


其中：
* X-Edge-Access-Token - 是我们调用方法时传递的 token
* /HelloWorldService/sayHello - 是方法地址，命名方式为 /服务名/方法名
* -d 是 curl 命令参数用来传递我们需要向接口传递的参数，这里我们传递了一个id、一个name共两个参数
不出意外的话，调用的结果为：

`{"code":200,"data":{"result":"Id: 123, Name: Lily"},"message":"ok"}`

其中的 result 就是我们在接口实现中的 SayHelloResponse 定义的内容。

总结

本文介绍了如何自定义一个API接口，显示利用proto文件定义一个接口，然后再使用Go语言代码实现，然后注册后即可使用此API。

了解了简单的整个过程后，你可以在 `HelloWorldService.SayHello()` 方法中实现更复杂的逻辑。

服务一旦定义后，你就可以轻松在其中添加更多的方法。




