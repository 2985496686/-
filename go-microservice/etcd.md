# etcd的出现

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

- etcd 的 MVCC 模块在持久化存储 key-value 的时候，保存到 boltdb 的 value 是
个结构体（mvccpb.KeyValue）， 它不仅包含你的 key-value 数据，还包含了关联的
LeaseID 等信息。因此当 etcd 重启时，可根据此信息，重建关联各个 Lease 的 key 集合
列表。


**优化 Lease 续期性能**

在v3版本中，lease机制进行了优化，
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTg4ODAzMjE1OCwtMjg3MzkxMTkwLC0xNj
g4ODAzNjE0LDE5MzkzNjE1NDAsMTQ1MDI1NDAyLC0xNTkyODQ0
MjExLDkzNjM1MDkwMiwxMjQwNzA2OTIxLDYyODg4NjU5LDIwNz
A3NTg5MzYsLTEzOTUwNjY2MTMsLTI2MTg2MDYzXX0=
-->