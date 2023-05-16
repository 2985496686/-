http://localhost:2379# etcd的出现

**CoreOS公司**在构建自己产品时遇到一个痛点，在管理多节点项目时，节点变更会导致用户服务的宕机。因此他们需要一个协调服务来存储服务的配置信息，提供分布式锁。

他们需要设计一个满足以下五点要求的协调服务：

- 可用性角度：高可用。协调服务作为集群的控制面存储，它保存了各个服务的部署、运行信息。若它故障，可能会导致集群无法变更、服务副本数无法协调。业务服务若此时出现故障，无法创建新的副本，可能会影响用户数据面。

- 数据一致性角度：提供读取“最新”数据的机制。通过raft保证了数据的强一致性，但是也同时降低了读写性能。
- 功能：增删改查，监听数据变化的机制。协调服务保存了服务的状态信息，若服务有变更或异常，相比控制端定时去轮询检查一个个服务状态，若能快速推送变更事件给控制端，则可提升服务可用性、减少协调服务不必要的性能开销。
- 运维复杂度：可维护性。通过api平滑的修改协调服务保存的信息。

**etcd和redis的区别**

1. etcd和redis 提供的服务层面是不一样的，redis主要是通过的k-v的方式存储大量用户信息数据等，而ectd是用来存储服务节点的信息，相比于redis存储的信息量少很多。
2. 为了适应不同的服务，二者的实现会有所不同。
3. redis通过主备异步复制，而etcd使用raft的强一致性复制，前者可能会造成数据不一致，但同样的后者的读写性能要差一些。
4. 数据分片上Redis有各种集群版解决方案，可以承载上T数据，存储的一般是用户数据，而etcd定位是个低容量的关键元数据存储，db大小一般不超过8g。
5. 存储引擎和API上Redis内存实现了各种丰富数据结构，而etcd仅是kv API, 使用的是持久化存储boltdb。


# docker 上安装并使用etcd


**拉取镜像**
```shell
docker pull bitnami/etcd:latest
```
**运行镜像**
```shell
docker run -d --name etcd -p 2379:2379 --env ALLOW_NONE_AUTHENTICATION=yes --env ETCD_ADVERTISE_CLIENT_URLS=http://0.0.0.0:2379 bitnami/etcd
```
- ``ALLOW_NONE_AUTHENTICATION=yes ``: 允许无身份验证。
- ``ETCD_ADVERTISE_CLIENT_URLS=http://0.0.0.0:2379`` : 客户端能够访问etcd的url


**集群部署etcd集群**

```shell
# For each machine
ETCD_VERSION=latest
TOKEN=my-etcd-token
CLUSTER_STATE=new
NAME_1=etcd-node-0
NAME_2=etcd-node-1
NAME_3=etcd-node-2
THIS_IP=0.0.0.0

CLUSTER=${NAME_1}=http://${THIS_IP}:2389,${NAME_2}=http://${THIS_IP}:2390,${NAME_3}=http://${THIS_IP}:2391

# For node 1
THIS_NAME=${NAME_1}
sudo docker run  -d --network host --name ${THIS_NAME} \
--env ALLOW_NONE_AUTHENTICATION=yes \
--env ETCD_NAME=${THIS_NAME} \
--env ETCD_ADVERTISE_CLIENT_URLS=http://${THIS_IP}:2379 \
--env ETCD_LISTEN_CLIENT_URLS=http://${THIS_IP}:2379 \
--env ETCD_INITIAL_ADVERTISE_PEER_URLS=http://${THIS_IP}:2389 \
--env ETCD_LISTEN_PEER_URLS=http://${THIS_IP}:2389 \
--env ETCD_INITIAL_CLUSTER=${CLUSTER} \
--env ETCD_INITIAL_CLUSTER_STATE=${CLUSTER_STATE} \
--env ETCD_INITIAL_CLUSTER_TOKEN=${TOKEN} \
bitnami/etcd:${ETCD_VERSION}
    
# For node 2
THIS_NAME=${NAME_2}
sudo docker run  -d --network host --nami/script…"   2 days ago   Up 30 seconds             etcd-node-2
b116ec073c93   bitnami/etcd:latest   "/opt/bitnami/script…"   2 days ago   Up 30 seconds             etcd-node-1
2f90aed4c9c7   bitnami/etcd:latest   "/opt/bitnami/script…"   2 days ago   Up 30 seconds             etcd-node-0
name ${THIS_NAME} \
--env ALLOW_NONE_AUTHENTICATION=yes \
--env ETCD_NAME=${THIS_NAME} \
--env ETCD_ADVERTISE_CLIENT_URLS=http://${THIS_IP}:2380 \
--env ETCD_LISTEN_CLIENT_URLS=http://${THIS_IP}:2380 \
--env ETCD_INITIAL_ADVERTISE_PEER_URLS=http://${THIS_IP}:2390 \
--env ETCD_LISTEN_PEER_URLS=http://${THIS_IP}:2390 \
--env ETCD_INITIAL_CLUSTER=${CLUSTER} \
--env ETCD_INITIAL_CLUSTER_STATE=${CLUSTER_STATE} \
--env ETCD_INITIAL_CLUSTER_TOKEN=${TOKEN} \
bitnami/etcd:${ETCD_VERSION}

# For node 3
THIS_NAME=${NAME_3}
sudo docker run  -d --network host --name ${THIS_NAME} \
--env ALLOW_NONE_AUTHENTICATION=yes \
--env ETCD_NAME=${THIS_NAME} \
--env ETCD_ADVERTISE_CLIENT_URLS=http://${THIS_IP}:2381 \
--env ETCD_LISTEN_CLIENT_URLS=http://${THIS_IP}:2381 \
--env ETCD_INITIAL_ADVERTISE_PEER_URLS=http://${THIS_IP}:2391 \
--env ETCD_LISTEN_PEER_URLS=http://${THIS_IP}:2391 \
--env ETCD_INITIAL_CLUSTER=${CLUSTER} \
--env ETCD_INITIAL_CLUSTER_STATE=${CLUSTER_STATE} \
--env ETCD_INITIAL_CLUSTER_TOKEN=${TOKEN} \
bitnami/etcd:${ETCD_VERSION}
```

# etcd的整体架构(V3版本)

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/VaiB6LwxjK9vyUKN.png)


## 一条读请求的执行

**Client层**抽离
执行一条命令：
```shell
docker exec -it etcd-node-2 etcdctl get hello world --endpoints localhost:2379,localhost:2380,localhost:2381
```

etcdctl 创建一个client3对象，在指定了多个endpoint时会通过自带的负载均衡算法(Round-robin轮询)选择一个可用的节点进行连接。

**kvserver层**

成功建立连接后，client就可以通过调用gRPC API 来访问kvserver。此时进入核心的读流程，读数据分为两种模式：串行读和线性读。

**串行读**

节点在收到读请求时直接执行，不经过raft层。

- ``优点：`` 读效率高。
- ``缺点：`` 数据一致性较差。leader的写请求日志已经追加到了过半节点，leader提交了日志(应用到状态机上)，但是收到读请求的节点还未来得及提交日志，此时就造成了数据不一致。

**线性读**
线性也就代表了强一致性，也是etcd默认使用的读取方式，具体流程如下：
1. follower收到读请求，会向leader发送ReadIndex请求，来获取leader提交的日志下标commitIndex。
2. leader收到ReadIndex请求后会向自己的节点发送心跳，当过半节点承认自己是leader时才会给follower发送自己的commitIndex。
3. follower此时会一直等待，直到自己的提交状态赶上了leader才会执行读请求。

- ``优点：`` 保证了数据的强一致性。
- ``缺点：`` 读请求效率较低。



**MVCC**

MVCC：多版本并发控制 (Multiversion concurrency control) 模块支持保存 key 的历史版本、支持多 key 事务等。

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/EF0zKI6rJPNV7fqQ.png)

它核心由内存树形索引模块 (treeIndex) 和嵌入式的 KV 持久化存储库 boltdb 组成。

``treeIndex:`` 使用B-tree数据结构，只保存用户 key 与版本号之间的映射关系。


``boltdb:`` 是一个持久化存储的B+ Tree ，key是版本号，而value是用户信息的key-value。


## 读请求什么时候会经过磁盘IO？
- 实际上，etcd 在启动的时候会通过 mmap 机制将 etcd db 文件映射到 etcd 进程地址空
间，并设置了 mmap 的 MAP_POPULATE flag，它会告诉 Linux 内核预读文件，Linux
内核会将文件内容拷贝到物理内存中，此时会产生磁盘 I/O。节点内存足够的请求下，后
续处理读请求过程中就不会产生磁盘 I/IO 了。

-  	若 etcd 节点内存不足，可能会导致 db 文件对应的内存页被换出，当读请求命中的页未在
内存中时，就会产生缺页异常，导致读过程中产生磁盘 IO，你可以通过观察 etcd 进程的
majflt 字段来判断 etcd 是否产生了主缺页中断。



## 一条写请求的执行



# etcd中的raft

这里只介绍etcd中raft的不同之处。
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/c7IjLn6jOBkZm0t8.png)

这里只关注etcd的Raft层和逻辑层。etcd对Raft层进行了高度抽离，这里的raft不负责网络传输和日志持久化，而相应的逻辑放在了逻辑层。


## 领导人选举

有这样一个场景：三个节点的raft集群，其中一个follower节点与其他两个节点出现了网络分区，这个follower因为竞选超时，由follower转变为了candidate，term加1，但是他的这个网络分区没有过半的节点，所以他会一直重复这个过程。当网络恢复后，由于他的term领先于其它节点，会导致集群开启了一场新的选举，选出新的leader后集群恢复正常工作。

上面是raft论文中的实现，但是这种选举只是由少数节点引发的无效选举，并且影响了集群的稳定性，etcd在这里进行了优化，引入了一个 PreVote 参数（默认 false），可以用来启用PreCandidate 状态解决此问题。Follower 在转换成 Candidate 状态前，先进入 PreCandidate 状态，不自增任期号， 发起预投票。若获得集群多数节点认可，确定有概率成为 Leader 才能进入 Candidate 状态，发起选举流程。这样就算少数节点出问题了，但是但网络恢复后也不会开启一个新的选举。


## 日志复制

**做为follower的节点，是什么时候将日志落盘到WAL文件中，是在收到leader节点同步过来的日志时，还是在leader节点通知某个日志已经在集群达成一致？为什么以及流程是怎样的？**


leader会在收到leader节点同步过来的日志时将日志落盘到wal文件中，落盘成功后，告知leader日志已经在集群达到一致。

理由：采用第一种方式，先将日志追加到未持久化数据缓冲区，然后告知leader日志已经一致，并在此时通知应用层将日志持久化落盘wal文件。在日志持久化成功之前follower宕机，但是，leader已经收到了他的一致性响应，并且此时恰好收到了过半节点的一致性响应(实际情况并没有过半的节点同步该日志)，leader将该日志提交了，此时leader宕机，原先的follower恢复，未同步刚才那条日志的节点可能会成为新leader，并且会覆盖掉其它节点的日志，这就会导致用户提交成功的数据，在集群中不存在。



# 租约Lease

**什么是租约？**
client 和 etcd server 之间存在一个约定，内容是 etcd server 保证在约定的有效
期内（TTL），不会删除你关联到此 Lease 上的 key-value。

**leassor模块**

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/RgNqNFj8peon0BYi.png)

etcd的lessor负责管理租约，在启动etcd时会创建该模块，并启动一个协程，定时的去完成两个任务一个是 RevokeExpiredLease 任务，定时检查是否有过期 Lease，发起撤销过期的Lease 操作。一个是 CheckpointScheduledLease，定时触发更新 Lease 的剩余到期时间的操作。

```go
func (le *lessor) runLoop() {  
   defer close(le.doneC)  
	//每五百毫秒会执行一次定时任务
   delayTicker := time.NewTicker(500 * time.Millisecond)  
   defer delayTicker.Stop()  
  
   for {  
      le.revokeExpiredLeases()  
      le.checkpointScheduledLeases()  
  
      select {  
      case <-delayTicker.C:  
      case <-le.stopC:  
         return  
      }  
   }  
}
```

**Lease创建** 
```shell
# 创建一个TTL为600秒的lease，etcd server返回LeaseID
etcdctl lease grant 600
lease 326975935f48f814 granted with TTL(600s)

# 查看lease的TTL、剩余时间
etcdctl lease timetolive 326975935f48f814
lease 326975935f48f814 granted with TTL(600s)， remaining(590s)
```

Lease server 在收到client创建lease请求后(当前节点如果不是leader，会自动转发给leader)，首先会通过raft模块write log，并同步到其他follower节点，leader收到过半节点的同步响应后，将日志应用到自己的状态机上。

首先 Lessor 的 Grant 接口会把 Lease 保存到内存的 ItemMap 数据结构中，然后它需要
持久化 Lease，将 Lease 数据保存到 boltdb 的 Lease bucket 中，返回一个唯一的
LeaseID 给 client。
```go
func (le *lessor) Grant(id LeaseID, ttl int64) (*Lease, error) {  
   if id == NoLease {  
      return nil, ErrLeaseNotFound  
   }  
  
   if ttl > MaxLeaseTTL {  
      return nil, ErrLeaseTTLTooLarge  
   }  
  
   // TODO: when lessor is under high load, it should give out lease  
   // with longer TTL to reduce renew load.  
   l := &Lease{  
      ID:      id,  
      ttl:     ttl,  
      itemSet: make(map[LeaseItem]struct{}),  
      revokec: make(chan struct{}),  
   }  
  
   if l.ttl < le.minLeaseTTL {  
      l.ttl = le.minLeaseTTL  
   }  
  
   le.mu.Lock()  
   defer le.mu.Unlock()  
  
   if _, ok := le.leaseMap[id]; ok {  
      return nil, ErrLeaseExists  
   }  
  
   if le.isPrimary() {  
      l.refresh(0)  
   } else {  
      l.forever()  
   }  
  
   le.leaseMap[id] = l  
   l.persistTo(le.b)  
  
   leaseTotalTTLs.Observe(float64(l.ttl))  
   leaseGranted.Inc()  
  
   if le.isPrimary() {  
      item := &LeaseWithTime{id: l.ID, time: l.expiry}  
      le.leaseExpiredNotifier.RegisterOrUpdate(item)  
      le.scheduleCheckpointIfNeeded(l)  
   }  
  
   return l, nil  
}
```



**将key绑定Lease**
```shell
$ etcdctl put node healthy --lease 326975935f48f818
OK
$ etcdctl get node -w=json | python -m json.tool
{
    "kvs":[
        {
            "create_revision":24，
            "key":"bm9kZQ=="，
            "Lease":3632563850270275608，
            "mod_revision":24，
            "value":"aGVhbHRoeQ=="，
            "version":1
        }
    ]
}
```
- 当你通过put 等命令新增一个指定了"--lease"的 key 时，MVCC 模块它会通过 Lessor 模块的
Attach 方法，将 key 关联到 Lease 的 key 内存集合 ItemSet 中。

```go
func (le *lessor) Attach(id LeaseID, items []LeaseItem) error {  
   le.mu.Lock()  
   defer le.mu.Unlock()  
   //通过id获取租约
   l := le.leaseMap[id]  
   if l == nil {  
      return ErrLeaseNotFound  
   }  
  
   l.mu.Lock()  
   //item其实就是一个key切片
   for _, it := range items {  
      l.itemSet[it] = struct{}{}  
      le.itemMap[it] = id  
   }  
   l.mu.Unlock()  
   return nil  
}
```

- etcd 的 MVCC 模块在持久化存储 key-value 的时候，保存到 boltdb 的 value 是
个结构体（mvccpb.KeyValue）， 它不仅包含你的 key-value 数据，还包含了关联的
LeaseID 等信息。因此当 etcd 重启时，可根据此信息，重建关联各个 Lease 的 key 集合
列表。


**优化 Lease 续期性能**


在正常情况下，你的节点存活时，需要定期发送 KeepAlive 请求给 etcd 续期健康状态的 Lease，否则你的 Lease 和关联的数据就会被删除。

KeepAlive作为一个高频请求，在etcd v2中使用http1.0 ，这种设计，因不支持连接多路复用、相同 TTL 无法复用导致性能较差，无法支撑较大规模的 Lease 场景。在etcd3.0中，将http1.0改为了grpc，grpc可以使用多路复用减少连接数量，使用长连接减少连接创建和销毁的次数，更加适合于etcd的业务。


**优化高效淘汰过期**

etcd3.5在创建lease时，会将租约按照过期时间创建一个最小堆，在前面有说过，lessor会有一个协程来负责定时淘汰过期的租约。

```go
func (le *lessor) revokeExpiredLeases() {  
   var ls []*Lease  
  
   // rate limit  
   revokeLimit := leaseRevokeRate / 2  
  
   le.mu.RLock()  
   if le.isPrimary() {  
	  //找到所有过期的lease
      ls = le.findExpiredLeases(revokeLimit)  
   }  
   le.mu.RUnlock()  
  
   if len(ls) != 0 {  
      select {  
      case <-le.stopC:  
         return  
      case le.expiredC <- ls:  
      default:  
         // the receiver of expiredC is probably busy handling  
         // other stuff         // let's try this next time after 500ms      }  
   }  
}
```

下面分析``findExpiredLeases``


```go
func (le *lessor) findExpiredLeases(limit int) []*Lease {  
   leases := make([]*Lease, 0, 16)  
   for {  
	   //通过expireExists方法获取一个堆定元素
      l, next := le.expireExists()  
      if l == nil && !next {  
         break  
      }  
      if next {  
         continue  
      }  
  
      if l.expired() {  
         leases = append(leases, l)  
         // reach expired limit  
         if len(leases) == limit {  
            break  
         }  
      }  
   }  
   return leases  
}

//两个返回值，第一个是过期的lease，第二个表示后续是否有可能还有过期的lease
func (le *lessor) expireExists() (l *Lease, next bool) {  
   //堆中已经没有lease
   if le.leaseExpiredNotifier.Len() == 0 {  
      return nil, false  
   }  
	//获取堆定顶元素
   item := le.leaseExpiredNotifier.Peek()  
   l = le.leaseMap[item.id]  
   if l == nil {  
      // lease 已经被lessor revoke，但是此时还存在于堆中(后面会解释为什么会出现这种情况)
      //调用这个方法会先pop，再调整堆log(N)
      le.leaseExpiredNotifier.Unregister()
      return nil, true  
   }  
   now := time.Now()  
   if now.Before(item.time) /* item.time: expiration time */ {  
      //还未排序      
      return nil, false  
   }  
  
   // 调整过期lease的time
   item.time = now.Add(le.expiredLeaseRetryInterval)  
   //进入RegisterOrUpdate方法后会调用Fix方法，重新调整堆
   le.leaseExpiredNotifier.RegisterOrUpdate(item)  
   return l, false  
}
```
在检测过期lease时，并不会立即将过期的lease 从堆中pop，而是延长它的过期时间，等待下一轮检测时再将其pop。

使用小根堆来管理lease的过期十分高效。

查询过期lease时间复杂度：``O(1)``
删除，插入，更新复杂度：``O(logN)``


etcd server 收到删除lease任务(通过expiredC)，同样是先write log，然后同步到follower节点，收到一半的回复后，应用到自己的状态机上。


**checkpoint 机制**

上述的过程检查 Lease 是否过期、维护最小堆、针对过期的 Lease 发起 revoke 操作都是由leader进行的，并且小根堆的维护是放在内存上的。那么当 Leader 因重启、crash、磁盘 IO 等异常不可用时，Follower 节点就会发起Leader 选举，新 Leader 要完成以上职责，必须按照持久化的lease ttl时间重建 Lease 过期最小堆等管理数据结构，这个过程就会将所有lease的过期时间刷新了一遍，若较频繁出现 Leader 切换，切换时间小于 Lease 的 TTL，这会导致 Lease 永远无法
删除，大量 key 堆积，db 大小超过配额等异常。



为了解决这个问题，etcd 引入了检查点机制，也就是下面架构图中黑色虚线框所示的
CheckPointScheduledLeases 的任务。

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/kNyYoPKaU4PoIkqV.png)

etcd 启动的时候，Leader 节点后台会运行此异步任务，定期批量地将 Lease 剩
余的 TTL 基于 Raft Log 同步给 Follower 节点，Follower 节点收到 CheckPoint 请求
后，更新内存数据结构 LeaseMap 的剩余 TTL 信息。

另一方面，当 Leader 节点收到 KeepAlive 请求的时候，它也会通过 checkpoint 机制把
此 Lease 的剩余 TTL 重置，并同步给 Follower 节点，尽量确保续期后集群各个节点的
Lease 剩余 TTL 一致性。

chekpoint机制虽然解决了问题，但是很明显了带来了性能的下降(每一次同步都设计磁盘IO)，所以该功能在etcd3.6以下都是默认关闭的。


# ETCD实现分布式锁


## 分布式锁具备特点

-   互斥性：在同一时刻，只有一个客户端能持有锁

-   安全性：避免死锁，如果某个客户端获得锁之后处理时间超过最大约定时间，或者持锁期间发生了故障导致无法主动释放锁，其持有的锁也能够被其他机制正确释放，并保证后续其它客户端也能加锁，整个处理流程继续正常执行
-   可用性：也被称作容错性，分布式锁需要有高可用能力，避免单点故障，当提供锁的服务节点故障（宕机）时不影响服务运行，这里有两种模式：一种是分布式锁服务自身具备集群模式，遇到故障能自动切换恢复工作；另一种是客户端向多个独立的锁服务发起请求，当某个锁服务故障时仍然可以从其他锁服务读取到锁信息(Redlock)
-   可重入性：对同一个锁，加锁和解锁必须是同一个线程，即不能把其他线程持有的锁给释放了
-   高效灵活：加锁、解锁的速度要快；支持阻塞和非阻塞；支持公平锁和非公平锁


## Redis实现分布式锁的缺点

1. **客户端长时间阻塞导致锁失效问题**
客户端A获取锁后在处理业务时长时间阻塞，导致锁过期释放。当阻塞恢复时，就会出现多客户端同时持有锁的情况。
redis的解决方案：
-- 延长锁的过期时间。
-- java redission 提供了看门狗机制，在业务处理完之前不断给锁续期。

2. **单点实例安全性问题**
为了保证分布式锁的高可用，需要部署redis的主从节点，在数据同步之前发生主从切换，可能就会丢失原先master上的锁信息，导致同一时间两个客户端同时持有锁。
redis的解决方案：
参考官方文档[redlock](https://redis.io/docs/manual/patterns/distributed-locks/#the-redlock-algorithm)


## ETCD实现分布式锁的思路

### prefix

etcd支持前缀查找，所以可以用一个前缀表示锁资源，前缀 + 唯一id的方式表示锁资源的持有者。


### lease机制

租约机制可以保证锁的活性，持有锁的客户端宕机，key自动过期，避免宕机。etcd客户端提供的lease续租机制解决客户端长时间阻塞导致锁失效问题。

### watch机制
redis采用忙轮询的方式来获取锁，etcd可以使用watch机制监听锁的删除事件，更加高效。

### 实现策略

etcd实现分布式锁的方案有很多种，可以通过判断是否存在一个固定的key来实现分布式锁，但是这种实现策略有很大的问题。当多客户端同时获取锁时，只有一个成功获得，其余多个客户端监听key的删除事件，一旦锁被释放，多个客户端同时收到锁删除事件(无论尝试加锁的顺序)进行加锁，这就是 **“惊群问题”** ，所以etcd官网提供了另外一种实现策略。

不再将一个固定的key当作锁资源，而是将一个前缀当作锁资源，每一个客户端尝试加锁的时候都会以该前缀创建一个key，并且监听前一个创建该前缀key的版本号。

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/xK458iHxdTijC6yN.png)

具体的实现方式会在源码解析时讲解。

### etcd的强一致性

etcd是基于raft实现的，对外提供的是强一致性的kv存储，不会存在类似于redis主从切换导致的不一致问题。


## etcd客户端concurrency包提供的分布式锁


### 锁的使用

```go
func NewLock() sync.Locker {  
	//创建客户端
   cli, err := clientv3.New(clientv3.Config{Endpoints: ip1,ip2...})  
   if err != nil {  
      log.Fatal(err)  
   }  
   //授权租约
   resp, err := cli.Grant(context.TODO(), 5)  
   if err != nil {  
      log.Fatal(err)  
   }  
   //创建会话
   //会话会创建一个租约，并在客户端生存期内保证租约的活性
   session, err := concurrency.NewSession(cli, concurrency.WithLease(resp.ID))  
   if err != nil {  
      log.Fatal(err)  
   }  
   //利用会话，指定一个前缀创建锁
   return concurrency.NewLocker(session, "/myLock/")  
}
```
通过以上方式就可以创建一个分布式锁，通过锁的Lock()和UnLock()方法加锁解锁。



### 源码解析

```go
type Mutex struct {  
   s *Session  //会话 
 
   pfx   string  //锁名称，key的前缀
   myKey string  //key
   myRev int64  //创建key的版本号
   hdr   *pb.ResponseHeader  
}
```


**加锁**
```go
func (m *Mutex) Lock(ctx context.Context) error {  
   //尝试获取锁
   resp, err := m.tryAcquire(ctx)  
   if err != nil {  
      return err  
   }
   ......
 }
```

```go
func (m *Mutex) tryAcquire(ctx context.Context) (*v3.TxnResponse, error) {  
   s := m.s  
   client := m.s.Client()  
  
   //拼接完整的key，prefix + leaseId
   m.myKey = fmt.Sprintf("%s%x", m.pfx, s.Lease())  
   // 比较操作，判断当前key的创建版本号是否为0,版本号为0表示key还未创建
   cmp := v3.Compare(v3.CreateRevision(m.myKey), "=", 0)  
   // 创建key操作(将当前key存储到etcd)
   p**加锁**ut := v3.OpPut(m.myKey, "", v3.WithLease(s.Lease()))  
   // 获取当前key的版本号操作  
   get := v3.OpGet(m.myKey)  
   // 获取第一个创建该前缀key的版本号(不会获取到已经删除的key的版本号)，这个key就是当前持有锁的key  
   getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)  
   //通过一个事务执行操作
   //若当前key不存在，创建key，并获取持有锁的key的版本号
   //若当前key存在，获取key的版本号，并获取持有锁的key
   resp, err := client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()  
   if err != nil {  
      return nil, err  
   }  
   //将当前key的把版本号赋值给myRev字段
   m.myRev = resp.Header.Revision  
   if !resp.Succeeded {  
      m.myRev = resp.Responses[0].GetResponseRange().Kvs[0].CreateRevision  
   }  
   return resp, nil  
}
```

```go
func (m *Mutex) Lock(ctx context.Context) error {  
   resp, err := m.tryAcquire(ctx)  
   if err != nil {  
      return err  
   }  
   // 将当前持有锁的key赋值给ownerKey
   ownerKey := resp.Responses[1].GetResponseRange().Kvs
   //ownerKey不存在，或者版本号等于自己创建key的版本号，表示当前key正持有锁，可直接返回  
   if len(ownerKey) == 0 || ownerKey[0].CreateRevision == m.myRev {  
      m.hdr = resp.Header  
      return nil  
   }  
   client := m.s.Client()  
   //等待锁的释放
   _, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)  
   // release lock key if wait failed  
   if werr != nil {  
      m.Unlock(client.Ctx())  
      return werr  
   }  
  
   // make sure the session is not expired, and the owner key still exists.  
   gresp, werr := client.Get(ctx, m.myKey)  
   if werr != nil {  
      m.Unlock(client.Ctx())  
      return werr  
   }  
  
   if len(gresp.Kvs) == 0 { // is the session key lost?  
      return ErrSessionExpired  
   }  
   m.hdr = gresp.Header  
  
   return nil  
}
```

```go
//等待锁的释放
func waitDeletes(ctx context.Context, client *v3.Client, pfx string, maxCreateRev int64) (*pb.ResponseHeader, error) {  
   //获取最新的该前缀的key的操作，WithMaxCreateRev(maxCreateRev)对返回值进行了限制，返回值版本号必须是小于等于maxCreateRev的，通过这个操作就可以获取仅小于自己key版本号的key
   getOpts := append(v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev))  
   for {  
      resp, err := client.Get(ctx, pfx, getOpts...)  
      if err != nil {  
         return nil, err  
      }  
      //不存在大于自己版本号的key，可以获取锁了
      if len(resp.Kvs) == 0 {  
         return resp.Header, nil  
      }  
      lastKey := string(resp.Kvs[0].Key)  
      //等待 lastKey 被删除
      if err = waitDelete(ctx, client, lastKey, resp.Header.Revision); err != nil {  
         return nil, err  
      }  
   }  
}
```

```go
//通过watch机制监听指定version 的key的删除。
func waitDelete(ctx context.Context, client *v3.Client, key string, rev int64) error {  
   cctx, cancel := context.WithCancel(ctx)  
   defer cancel()  
  
   var wr v3.WatchResponse  
   wch := client.Watch(cctx, key, v3.WithRev(rev))  
   for wr = range wch {  
      for _, ev := range wr.Events {  
	     //监听Delete事件
         if ev.Type == mvccpb.DELETE {  
            return nil  
         }  
      }  
   }  
   if err := wr.Err(); err != nil {  
      return err  
   }  
   if err := ctx.Err(); err != nil {  
      return err  
   }  
   return fmt.Errorf("lost watcher waiting for delete")  
}
```

**释放锁**

```go
func (m *Mutex) Unlock(ctx context.Context) error {  
   client := m.s.Client()  
   //直接删除key
   if _, err := client.Delete(ctx, m.myKey); err != nil {  
      return err  
   }  
   m.myKey = "\x00"  
   m.myRev = -1  
   return nil  
}
```

## 总结

基于etcd实现的分布式锁基本上使用到了etcd的全部性质，并且保证了分布式锁的互斥性，安全性和可用性。官方实现的分布式锁并不支持可重入性，但是要实现可重入性锁也很简单，对这个锁在封装一层，并增加一个计数器。


**参考资料**
极客时间- etcd实战课
[etcd分布式锁的实现原理](https://juejin.cn/post/7062900835038003208)






# ETCD 的事务机制

## 事务的隔离级别
-   **未提交读（Read Uncommitted）**：能够读取到其他事务中还未提交的数据，这可能会导致脏读的问题。
-  **读已提交（Read Committed）**：只能读取到已经提交的数据，即别的事务一提交，当前事务就能读取到被修改的数据，这可能导致不可重复读的问题。
-   **可重复读（Repeated Read）**：一个事务中，同一个读操作在事务的任意时刻都能得到同样的结果，其他事务的提交操作对本事务不会产生影响。
-   **串行化（Serializable）：** 串行化的执行可能冲突的事务，即一个事务会阻塞其他事务。它通过牺牲并发能力来换取数据的安全，属于最高的隔离级别。


## etcd中的微事务
etcd v3 中引入了一个微事务的概念（mini-transaction），允许用户在一次修改中批量执行多个操作，这意味着这一组操作被绑定成一个原子操作，并共享同一个修订号。它的写法有点类似一个 CAS

```go
Txn().If(cond1, cond2, …).Then(op1, op2, …).Else(op1’, op2’, …).commit()
```

如果`If`语句中的条件全部为真，则整个事务的返回值为`true`，并且`Then`中的操作会被执行，否则返回`false`并执行`Else`中的操作。

## 从ACID角度分析微事务

### 读写事务
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/cAFVRZ1PenyJ9RGp.png)
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/etcd/8s4nLBjWTOAWRpr3.png)

读事务首先会在buffer访问，若buffer中没有才会到boltdb中访问，这里访问boltdb并不会访问磁盘，etcd在启动时就会将磁盘中的内容映射到内存中。

写事务会先将数据写入buffer和boltdb，持久化机制会定时将数据刷到磁盘中去，这种异步提交的方式很大的提高了写书


### 持久化(Durability)
etcd在事务提交时，就会将blotdb
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg2MjYzNTk1MiwxMTgxMjE3NDEyLDkyNj
U4NTM2OSwxMTE1ODQxMzI3LDQxNDM3NzUwNiw4MjM3MjgzOSwt
MTAyNzYzOTgxNSwtOTU0OTExNTE1LDc1ODA2ODQ5MCwxMTY0MD
U0MzQ5LC0xMjQ0NjczMjc0LDQyMzU3NTc4NSwtMTkyMzMwMTIw
LC00OTIwNjY5NTUsMTE1Nzc5Mzk0OSwtMTI4NjA1MTE4MCwyMD
E3NjYxNDYzLC0xODQ1NDQ4MjIyLDE2MDI2NDM1OTYsMjA1MDAw
OTk1XX0=
-->