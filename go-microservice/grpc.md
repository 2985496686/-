
# protobuf

- grpc 框架使用protobuf进行数据的传输。
- 相比与json和xml，这种方式的序列化和反序列化效率远优于Json
- 占用内存少。
- 可读性差。

```shell
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative script/rpc.proto
```


# grpc 负载均衡和服务注册


##  builder和resolver

下面是一个写死的负载均衡例子（client端）：

**main**
```go
func main() {  
   b := &UserBuilder{addrs: map[string][]string{"api": []string{"localhost:9999", "localhost:9998", "localhost:9997"}}}  
   resolver.Register(b)  
   conn, err := grpc.Dial("user:///api", grpc.WithTransportCredentials(insecure.NewCredentials()), grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`))  
   if err != nil {  
      log.Fatal(err)  
   }  
   client := pb.NewUserClient(conn)  
   for {  
      if response, err := client.Login(context.Background(), &pb.UserRequest{Name: "张三", Password: "111111"}); err != nil {  
         fmt.Println(err)  
      } else {  
         fmt.Println(response)  
      }  
   }  
}
```
**builder**
```go
  
type UserBuilder struct {  
   addrs map[string][]string  
}  
  
func (b *UserBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {  
   r := &UserResolver{  
      target:     target,  
      cc:         cc,  
      addrsStore: b.addrs,  
   }  
   r.start()  
   return r, nil  
}  
  
func (b *UserBuilder) Scheme() string {  
   return "user"  
}
```

**resolver**

```go
type UserResolver struct {  
   target     resolver.Target  
   cc         resolver.ClientConn  
   addrsStore map[string][]string  
}  
  
func (r *UserResolver) ResolveNow(options resolver.ResolveNowOptions) {  
   fmt.Println("1111")  
}  
  
func (r *UserResolver) Close() {  
}  
  
func (r *UserResolver) start() {  
   addrStrs := r.addrsStore[r.target.Endpoint()]  
   addrs := make([]resolver.Address, len(addrStrs))  
   for i, s := range addrStrs {  
      addrs[i] = resolver.Address{Addr: s}  
   }  
   err := r.cc.UpdateState(resolver.State{Addresses: addrs})  
   fmt.Println(err)  
}
```

- 首先我们创建了builder对象，并初始化它的属性字段(自定义的)。
builder实现了两个重要方法，一个是Sheme
- 将builder注册到resolver中。
resolver包下有一个全局变量，如下：
![输入图片说明](https://raw.githubusercontent.com/2985496686/-/master/imgs/grpc/J1A83ROL9RxsXSZ6.png)
这里所谓的注册就是将Sheme和builder以相对应的关系存储到m中。

- 当调用Dail方法时，就会将``"user:///api"``解析成一个Target对象，并且通过``user``找到builder，调用build方法，生成一个resolver对象。
注意这里我们需要调用``r.cc.UpdateState(resolver.State{Addresses: addrs})  `` ，将地址传入(这里并没有马上更地址建立网络连接)。
- 至此，当我们调用rpc方法后，会通过轮询的方法访问每一个服务器。


# 基于etcd进行服务发现

![输入图片说明](https://raw.githubusercontent.com/2985496686/-/master/imgs/grpc/ex4SkmmJ9aensrTF.png)

## 客户端

```go
package main  
  
import (  
   "context"  
   pb "etcd-discovery/client/rpc"  
   "fmt"  
   clientv3 "go.etcd.io/etcd/client/v3"  
   "go.etcd.io/etcd/client/v3/naming/resolver"   "google.golang.org/grpc"   "google.golang.org/grpc/credentials/insecure"   "log")  
  
func main() {  
  
   //创建访问etcd的客户端连接  
   cli, err := clientv3.NewFromURL("http://localhost:2379")  
   if err != nil {  
      log.Fatal(err)  
   }  
   //通过该连接创建builder  
   builder, err := resolver.NewBuilder(cli)  
   if err != nil {  
      log.Fatal(err)  
   }  
   //注意这里的target，需要写成etcd:///api，其中etcd://是指定协议类型为etcd，第三个/表示服务在etcd的根目录下  
   conn, err := grpc.Dial("tcp:///api", grpc.WithTransportCredentials(insecure.NewCredentials()), grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`), grpc.WithResolvers(builder))  
   if err != nil {  
      log.Fatal(err)  
   }  
   client := pb.NewUserClient(conn)  
   for {  
      if response, err := client.Login(context.Background(), &pb.UserRequest{Name: "张三", Password: "111111"}); err != nil {  
         fmt.Println(err)  
      } else {  
         fmt.Println(response)  
      }  
   }  
}
```
- 客户端每次在进行远端请求的时候，都会通过etcd发现服务，然后获取有效的服务端地址列表(string数组)。
- 客户端会有个map，存储addr(服务端地址)和connection(服务端连接)的映射关系。
- 客户端完成服务发现后，会通过自己的负载均衡算法(grpc支持轮询算法)，选择一个服务端地址(服务发现后的地址列表，确保是有效的)查询map，若在map中不存在，建立连接。

**注意：**
- 这里的连接是懒连接，长连接。



## 服务端

**etcd.go**

```go
package main  
  
import (  
   "context"  
   "errors"   clientv3 "go.etcd.io/etcd/client/v3"  
   "go.etcd.io/etcd/client/v3/naming/endpoints"   "log")  
  
const serviceName = "api"  
const etcdUrl = "http://localhost:2379"  
const ttl = 10  
  
var etcdClient *clientv3.Client  
  
func etcdRegister(addr string) error {  
  
   var err error  
   etcdClient, err = clientv3.NewFromURL(etcdUrl)  
   if err != nil {  
      log.Fatal("failed to register:", err)  
   }  
   log.Println(addr, "register server")  
  
   //endpoints.Manager 是一个用于管理服务地址列表的组件，它提供了添加、删除、获取地址列表等功能  
   m, err := endpoints.NewManager(etcdClient, serviceName)  
   if err != nil {  
      log.Fatal("failed to register:", err)  
   }  
  
   //授权一个租约，并且10s过期  
   lease, err := etcdClient.Grant(context.TODO(), ttl)  
   if err != nil {  
      log.Fatal(err)  
   }  
  
   //将指定地址（addr）的服务添加到指定路径（/api/addr）的服务端点（endpoint）中。  
   //注意这里的服务名和服务端点名不一样，一个服务名下会有多个服务端点  
   err = m.AddEndpoint(context.TODO(), serviceName+"/"+addr, endpoints.Endpoint{Addr: addr}, clientv3.WithLease(lease.ID))  
   if err != nil {  
      log.Fatal("failed to register:", err)  
   }  
  
   //在 etcd 中续租指定 ID 的租约，防止该租约过期，默认每三秒续租一次  
   alive, err := etcdClient.KeepAlive(context.Background(), lease.ID)  
  
   go func() {  
      for {  
         select {  
         case _, ok := <-alive:  
            if !ok {  
               log.Println("keep alive chan has closed")  
               return  
            } else {  
               log.Println("etcd server keep alive")  
            }  
         }  
      }  
   }()  
   return err  
}  
  
func etcdLogOf(addr string) error {  
   log.Println("addr etcd log off")  
   m, err := endpoints.NewManager(etcdClient, etcdUrl)  
   if err != nil {  
      return errors.New("etcdLogOf error:" + err.Error())  
   }  
   err = m.DeleteEndpoint(context.TODO(), serviceName+"/"+addr)  
   if err != nil {  
      return errors.New("etcdLogOf error:" + err.Error())  
   }  
   return nil  
}
```

****
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA0MzY2MjA2NSwxNjIyODMxOTIwLDE4Nz
A1NzcyNjldfQ==
-->