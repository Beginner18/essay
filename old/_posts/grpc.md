---
title: grpc
date: 2019-08-05 18:08:41
tags: grpc
---

1 grpc
===
***特色***:
---
>- 解决语言及环境通信的复杂性：protobuf
>- 跨语言、多语言：插件根据protobuf生成对应语言代码
>- 各种环境：云服务器/平板电脑
>- 高效的序列号：名称用序列代替
>- 简单的 IDL: interface defined language
>- 容易进行接口更新
>- 基于http/2
>- 自动根据proto文件生成对应语言的api，client&server端

***缺点***：
---
> 1. GRPC尚未提供连接池，需要自行实现 
> 2. 尚未提供“服务发现”、“负载均衡”机制 
> 3. 因为基于HTTP2，绝大部多数HTTP Server、Nginx都尚不支持，即Nginx不能将GRPC请求作为HTTP请求来负载均衡，而是作为普通的TCP请求。 （nginx1.9版本已支持） 
> 4. Protobuf二进制可读性差（貌似提供了Text_Fromat功能） 
默认不具备动态特性（可以通过动态定义生成消息类型或者动态编译支持）

***GRPC使用流程***
---
> 1. 写.proto文件
> 2. 生成对应语言接口 pb.go; proto-gen-go插件中插件grpc
> 3. 实现客户端及服务端(可用不同语言)
>
>```
//////服务端  
实现pb.go中相关方法
生成grpc服务端server grpc.NewServer()并开始监听lis, err := net.Listen(“tcp”, port)
newServer建立server结构体，其中包含server的相关配置：最大读/写缓冲区大小，是否安全认证
利用pb.go中方法，注册grpc方法函数需要用到server及实现3.1.1中方法的结构体&struct{}
packagePbGo.NewExportXXServer(server, &struct{})
注册server：reflection.Register(server)
server.Serve(lis)
///////客户端
// 基于http2服务，需要先建立连接，设置options, 客户端将req编码为对应的编码格式，然后通过连接发送至server端
发起链接：conn := grpc.Dial(address, grpc.WithInSecure()) //非https 无tls ttl
defer conn.Close
建立客户端：c := packagePbGo.NewExportXXClient(conn)
方法调用: &res, err := c.MethodName(contex.Background(), &req)
``` 

***GRPC 中间件***
---
>- 创建cliConn或者server的时候通过options设置中间件
>
```
// 只能添加一个中间件;多个中间件通过grpc middlerware组合成一个中间件
// WithUnaryInterceptor returns a DialOption that specifies the interceptor for
// unary RPCs.
func WithUnaryInterceptor(f UnaryClientInterceptor) DialOption {
	return newFuncDialOption(func(o *dialOptions) {
		o.unaryInt = f
	})
}
```
>- grpc-middlerWare将grpc中间件函数组合成chain，链式中间件
>
```
// ChainUnaryClient creates a single interceptor out of a chain of many interceptors.
//
// Execution is done in left-to-right order, including passing of context.
// For example ChainUnaryClient(one, two, three) will execute one before two before three.
func ChainUnaryClient(interceptors ...grpc.UnaryClientInterceptor) grpc.UnaryClientInterceptor {
	switch len(interceptors) {
	case 0:
		// do not want to return nil interceptor since this function was never defined to do so/for backwards compatibility
		return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
			return invoker(ctx, method, req, reply, cc, opts...)
		}
	case 1:
		return interceptors[0]
	default:
		return func(ctx context.Context, method string, req, reply interface{}, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
			buildChain := func(current grpc.UnaryClientInterceptor, next grpc.UnaryInvoker) grpc.UnaryInvoker {
				return func(currentCtx context.Context, currentMethod string, currentReq, currentRepl interface{}, currentConn *grpc.ClientConn, currentOpts ...grpc.CallOption) error {
					return current(currentCtx, currentMethod, currentReq, currentRepl, currentConn, next, currentOpts...)
				}
			}
			chain := invoker
			for i := len(interceptors) - 1; i >= 0; i-- {
				chain = buildChain(interceptors[i], chain)
			}
			return chain(ctx, method, req, reply, cc, opts...)
		}
	}
}
```
// 语法糖

```
// Chain creates a single interceptor out of a chain of many interceptors.
//
// WithUnaryServerChain is a grpc.Server config option that accepts multiple unary interceptors.
// Basically syntactic sugar.
func WithUnaryServerChain(interceptors ...grpc.UnaryServerInterceptor) grpc.ServerOption {
	return grpc.UnaryInterceptor(ChainUnaryServer(interceptors...))
}
```
[grpc实用工具github](https://github.com/grpc-ecosystem/awesome-grpc)
[go-grpc-metrics中间件](https://github.com/grpc-ecosystem/go-grpc-prometheus)
[grpc中间件](https://github.com/grpc-ecosystem/go-grpc-middleware)
1.1 grpc内部实现
---
server端：
>- grpc关键组件处理
>- 安全认证：TLS等
>- http/2
>
```
// Server is a gRPC server to serve RPC requests.
type Server struct {
	opts options
	mu     sync.Mutex // guards following
	lis    map[net.Listener]bool
	conns  map[io.Closer]bool
	serve  bool
	drain  bool
	cv     *sync.Cond          // signaled when connections close for GracefulStop
	m      map[string]*service // service name -> service info,注册的service放置在该map中
	events trace.EventLog
	quit               chan struct{}
	done               chan struct{}
	quitOnce           sync.Once
	doneOnce           sync.Once
	channelzRemoveOnce sync.Once
	serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop
	channelzID int64 // channelz unique identification number
	czData     *channelzData
}
// RegisterService registers a service and its implementation to the gRPC
// server. It is called from the IDL generated code. This must be called before
// invoking Serve.
func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
	ht := reflect.TypeOf(sd.HandlerType).Elem()
	st := reflect.TypeOf(ss)
	if !st.Implements(ht) {
		grpclog.Fatalf("grpc: Server.RegisterService found the handler of type %v that does not satisfy %v", st, ht)
	}
	s.register(sd, ss)
}
func (s *Server) register(sd *ServiceDesc, ss interface{}) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.printf("RegisterService(%q)", sd.ServiceName)
	if s.serve {
		grpclog.Fatalf("grpc: Server.RegisterService after Server.Serve for %q", sd.ServiceName)
	}
	if _, ok := s.m[sd.ServiceName]; ok {
		grpclog.Fatalf("grpc: Server.RegisterService found duplicate service registration for %q", sd.ServiceName)
	}
	srv := &service{
		server: ss,
		md:     make(map[string]*MethodDesc),
		sd:     make(map[string]*StreamDesc),
		mdata:  sd.Metadata,
	}
	for i := range sd.Methods {
		d := &sd.Methods[i]
		srv.md[d.MethodName] = d
	}
	for i := range sd.Streams {
		d := &sd.Streams[i]
		srv.sd[d.StreamName] = d
	}
	s.m[sd.ServiceName] = srv
}
```

2 grpc-gateway： 
===
It reads protobuf service definitions and generates a reverse-proxy server which translates a RESTful HTTP API into gRPC. 
>- 实现server端http请求
>- .proto文件中options中给出method, path等
>- 实现方式：建立与grpc conn的连接，gateway server建立自己的restful server,存在自己的路由函数，根据cli请求，找到对应路由函数，进而通过与grpc conn的连接调用grpc server对应的方法，实现http请求
>
```
定义grpc server proto文件
your_service.proto:
syntax = "proto3";
package example;
message StringMessage {
  string value = 1;
}
service YourService {
  rpc Echo(StringMessage) returns (StringMessage) {}
}
添加google.api.http 注解到 .proto file
your_service.proto:
 syntax = "proto3";
 package example;
+
+import "google/api/annotations.proto";
+
 message StringMessage {
   string value = 1;
 }
 service YourService {
-  rpc Echo(StringMessage) returns (StringMessage) {}
+  rpc Echo(StringMessage) returns (StringMessage) {
+    option (google.api.http) = {
+      post: "/v1/example/echo" // method path
+      body: "*"
+    };
+  }
 }
```
[参考github](https://github.com/grpc-ecosystem/grpc-gateway)

3 grpc服务注册服务发现
---
>- [服务注册服务发现负载均衡的实现](https://segmentfault.com/a/1190000008672912)
>- [基于etcd实现服务发现负载均衡](https://juejin.im/post/5a66046c51882535a47cfc8d)
>- [熔断降级限流](https://juejin.im/post/5cced96e6fb9a032514bbf94)