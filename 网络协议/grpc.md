# gRPC
grpc是google开源的一个rpc协议，默认通过protocol buffer来定义接口和交互方式，。服务端实现生成的接口提供服务，客户端通过stub（类似于客户端调用sdk）跟服务端进行交互。

## 问题
1. 为什么grpc快？
2. grpc负载均衡？提供roundrobin，rls，grpclb，weightedroundrobin，weightedtarget方式进行负载均衡。
3. 

## protocol buffers
与语言和平台无关的序列化协议，类似于JSON和XML。

## grpc特性
1. 流模式。可以通过流模式进行接受和发送，达到客户端边发送服务端可以边响应的目的，适用于**实时交互或者持续发送大数据包的场景**。
2. 验证器。
3. 拦截器。
4. 负载均衡。
5. 

## grpc通信模式
1. 一元rpc（unaryRPC）。最简单的模式，客户端发送请求，服务端处理并返回一个响应。
2. 服务端流式RPC（service-side streaming RPC）。客户端发起一次请求，服务端通过流水发送多次响应数据的模式。通过在pb中定义stream类型的response，服务端使用response的send方法进行发送。客户端通过response的recv方法进行接收。
3. 客户端流式RPC（client-side streaming RPC）。客户端发送多次请求数据，服务端接收处理返回响应的模式。通过在pb中定义stream类型的request，客户端在发送时，通过request的send方法进行发送，当发送完毕后，调用**closeAndRecv**方法进行接收。服务端使用request的recv进行接收，当读取到EOF时说明已经发送完毕，这时经过处理，通过调用**SendAndClose**方法响应数据并提示客户端结束recv。
4. 双向流式RPC（idirectional streaming RPC）。在pb文件中，request和response都定义为stream模式，客户端调用多次send方法发送多条请求，服务端在handler中处理完之后通过send发送响应，然后循环调用recv方法接收下一条请求并作出响应。

## 拦截器
拦截器分为两种类型。
1. 一元拦截器（unary interceptor）。拦截处理一元RPC调用。
2. 流拦截器（stream interceptor）。拦截处理流式RPC调用。


## channel
表示和终端的一个虚拟连接，一个channel对应多个HTTP2连接，一个HTTP2连接对应多个RPC

## RPC
一个RPC属于一个HTTP2连接，一个RPC对应多个message

## message

## 服务启动服务
1. 注册信息。我们通过protocol buffers文件定义完接口后生成pb.go文件，文件中会包含message的定义，接口的定义、接口中每个方法的handler，服务的描述信息ServiceDesc。我们调用pb文件的regiseter方法，将desc和我们的具体实现绑定起来，完成注册操作。
2. 监听信息。我们需要创建一个listener，一般会创建一个tcpListener来启动grpc服务。
3. 通过监听信息和注册信息生成server。
4. 将server与接口实现进行绑定，执行serve

## 客户端创建连接
### 


## 名称解析器
解析器将名称转换为地址，然后将地址交给负载均衡器。同时提供定期更新机器列表的能力，以保证在机器列表发生改变时可以知晓。当列表被更新的时候需要告知负载均衡器地址列表改动了。
grpc定义了名称解析器的接口，实现这个接口，grpc的名称解析器就可以使用对应的方法来进行名称解析。
```go
// Resolver creates a Watcher for a target to track its resolution changes.
//
// Deprecated: please use package resolver.
type Resolver interface {
	// Resolve creates a Watcher for target.
	Resolve(target string) (Watcher, error)
}
```
以etcd为例，etcd实现了Resolver接口，只要提供一个etcd的client，就可以使用名称解析功能。
```go
resolver := &naming.GRPCResolver{Client: etcdClient}
```
将命名解析器注册到负载均衡器中
```go
balancer := grpc.RoundRobin(resolver)
```


## 负载均衡器
负载均衡器创建连接，并在连接之前对rpc进行负载均衡，当连接失败时，负载均衡器将用最后已知的地址列表进行重连，同时解析器也会开始尝试解析主机名列表以保证机器信息是最新的。

### 如何识别失效连接
* 可以清除的失效连接：如连接被对方主动断开，这时会有FIN挥手，gRPC会立即开始重新进行连接。
* 不可清除的失效连接：如服务器崩溃，或者网络中间有段时间连不上了，这时其实是没有什么征兆的。为了识别这种情况，gRPC通过定期的心跳包来确认对方是否可达，如果不可达，就将连接关闭，重新建立连接。



# 参考资料
[HTTP/2 简介](https://developers.google.com/web/fundamentals/performance/http2?hl=zh-cn#%E6%95%B0%E6%8D%AE%E6%B5%81%E3%80%81%E6%B6%88%E6%81%AF%E5%92%8C%E5%B8%A7)


