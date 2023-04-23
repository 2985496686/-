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
sudo docker run  -d --network host --name ${THIS_NAME} \
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
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA3MDc1ODkzNiwtMTM5NTA2NjYxMywtMj
YxODYwNjNdfQ==
-->