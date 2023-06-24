# 架构设计

## 协议的设计

**实现目标：**  保证消息的可靠性和一致性。

- 可靠性： 消息不会丢失也不会重复，不会出现发送方显示发送成功，但是接收方并未收到消息的情况或者接收方连续收到两条相同的消息。
``问题：`` 传输层已经使用tcp协议了，为什么还会产生消息丢失的问题？
tcp只能保证运输层消息的可靠性，当消息成功到达用户进程后tcp的任务已经完成，此时如果用户进程崩溃，消息就会丢失。


- 一致性：消息有着全局的顺序，不能出现消息乱序的问题。
``消息乱序产生原因: `` 多发送方，多接收方，多进程线程，网络传输，没有全局时钟。


阅读：
[零基础IM开发入门(四)：什么是IM系统的消息时序一致性？](http://www.52im.net/thread-3189-1-1.html)

**方案选型**

将可靠性和一致性细分为四点：
1. 消息及时：服务端实时接收消息，并转发。
2. 消息可达：超时重传，ACK确认。
3. 消息幂等(不重复)：分配seqID，服务端存储seqID。
4. 消息有序：seqId可比较，接收端可根据seqId给消息排序。





**上行消息**
上行消息为客户端发送给服务端的消息，为了保证消息的可靠性和一致性，客户端在给服务端发送消息时，保证消息有一个唯一递增的id，这里称为clientId。

实现方案：
参考raft leader同步日志的流程，客户端不需要维护一个严格递增的clientId，可以将客户端的时间戳当作消息的clientId，保证消息的趋势递增。客户端在发送消息时需要维护两个字段preCId和CId，preCId表示前一条消息的Id，同时服务端也需要维护前一条消息的preCId，服务端就可以根据这两个字段来判断消息是否有丢失。


**下行消息(如何分配seqId?)**

只有上行消息的CId是不够的，CId只能保证在一次长连接中消息的序号是唯一且递增的，服务端收集消息然后交付给客户端需要保证来自一个客户端的消息有顺序(期间可能多次建立长连接)，所以除了有CId，对于同一个用户的消息，还需要有一个全局递增的Id

- 方案一：用户的每条消息都会通过redis获取一个全局有序的seqId。
缺点：所有消息都需要通过单点redis生成唯一ID，redis压力过大，效率低。

- 方案二：消息的seqId并不需要保证在所有用户消息中唯一有序，只需要保证对于同一个用户发送的消息的seqId是有序唯一的，不同用户之间的seqId可以相同。在用户注册时可以通过雪花算法为用户分配一个全局唯一的userId，消息的id可以为``{userId}:{seqId}`` 。对于seqId的生成不再依赖单点的redis，可以通过对userId哈希，匹配一个redis节点，生成对应的seqId。
缺点：redis的从节点在同步主节点``{userId}:{seqId}``之前崩溃，从节点变成新的主节点后，消息的seqId可能会重复。

- 方案三：为解决方案二所存在的问题，可以引入一个runId，runId为redis主节点的nodeId。当主节点崩溃之后，从节点变成了新的主节点，它保存的runId和nodeId已经不一致。此时它会让seqId进行一个跃变(保证此时的Id不会重复)，并且更新runId。此时的seqID由严格递增变为了趋势递增，客户端收到seqId跃变的消息后，并不知道消息是丢失了还是seqId产生了跃变，客户端需要向服务器发送pull请求。


**问题1：** 既然下行消息使用了全局递增的seqId，为什么上行消息不直接使用seqId？

1. 因为上行消息的Id需要由客户端进行分配。
2. 



# 长连接网关

## 为什么需要长连接
1.  基于tcp实现长连接，可以减少每次发送消息时DNS解析，tcp三次握手的开销。
2. 长连接可以实现服务端主动向客户端推送消息，保证消息的及时性。
3. 长连接是一个有状态的服务，难以维护和管理。如果将长连接网关和server层放在一起，每次业务更新重启服务时，势必会导致长连接的断开和重连，严重影响用户的体验，所以将长连接网关抽离出来，形成接入层，从而保证服务的变更不会导致长连接的重新建立。




## 设计目标

1. 

## 方案选型

### ip的选择
通过自建http server 作为ip config server 
1. 客户端通过http请求访问ipConfig 服务。
2. ipConfig会将长连接网关的ip列表依据负载，网络等因素排序，然后返回给客户端。
3. 客户端ip列表的顺序，依次尝试进行tcp长连接，直到连接成功。
4. ipConfig会定期通过etcd进行服务发现，并剔除不可用服务。



connectNums




# 2. IpConfig 实现

ipConfig 需要实现三个目标：
1. 服务发现，及时地发现长连接网关服务的变更。
2. 负载均衡，根据长连接网关的负载，为ip列表排序。
3. 对外暴露获取ip列表的http接口。



## 2.1 IpConfig 核心流程

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/iJNe1EIeMapl9Oll.png)
这里自下向上的分析Config服务：
1. data 模块通过访问discovery 服务，进行服务发现与监听，并将服务的变更封装成事件通知serviceManage模块。
2. serviceManage收到来自data的事件后变更自己的服务，并提供服务调度的方法(负载均衡)。
3. gin web 对外暴露获取ip列表的接口，client访问时，调用serviceManage模块，获取负载均衡排序后的ip列表。



## 2.2 ipConfig 服务发现与监听

etcd的服务发现可能会被多个服务需要，所以将它抽离成discovery服务，这里会一并讲解。

ipConfig的服务发现借助了 ``etcd 的watch机制`` ，以满足服务发现的极时性需求。当ipConfig服务启动后会开启一个服务发现的协程，该协程首先会初始化服务，调用etcd client的get方法，发现已有的服务。然后将发现的服务数据封装成添加服务事件，通过event chan 通知serviceManage。初始化完毕后，协程通过watch机制监听etcd服务的变更，同时将变更的数据封装成事件，通知上层。




## 2.3 ipConfig 对长连接网关服务的负载均衡

负载均衡常用的方式有轮询，随机，一致性哈希等，但这些都不适用于对于长连接网关服务的负载均衡，主要有以下原因：

1. 长连接是持续的资源消耗。
2. 对于每一条连接所消耗的资源是不确定的，有活跃连接和非活跃连接。

3. 节点负载状况具有很强时效性，不稳定。

在短链接的场景下，请求任务很多，但是任务大小相差不大，只要保证任务能够平均分配到每一台机器上，就能实现负载均衡。但在长连接的情况下，就需要考虑每台服务器的平均负载情况，我们可以采用以下方案。


**平均负载**

ipConfig在派发任务时会统计长连接网关的负载情况，这里实现为了简单，只统计连接数量和单位时间内传输的数据量，然后根据这两个值得到两个分值：静态分和动态分，以这两个分值为依据进行负载均衡。

这样实现还有一个问题，忽略了上面的第三点。为了得到一个稳定的负载数据，我们可以维护一个窗口，假如这里窗口大小为4，那么窗口就保存的是最近四次状态，并且会每秒更新一次。同时也要考虑，越靠近当前时间的负载数据可信度越高，在计算负载分值时可根据可信度加权求和。


参考文章： [长连接的负载均衡](https://lushunjian.github.io/blog/2018/07/28/%E9%95%BF%E8%BF%9E%E6%8E%A5%E7%9A%84%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1/)


# 可靠性传输协议的实现

## 为什么不直接使用webSocket



```shell
# 设置系统中允许同时打开的文件描述符的最大数量
sudo sysctl -w fs.file-max=2000500

sysctl -w fs.nr_open=2000500
```



## epoll

**连接前**

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/982YfEDsnRTvH9wH.png)



**20万连接**
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/EIdbPhBIAIrQDuYI.png)



[图片上传失败...(image-Umqo5lgvta1LZB9L)]

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/zfjgrX3HS13tS1t2.png)


**22万长连接**
![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/XleFj244DxGmXjXz.png)


## goroutinue-pre-connection

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/TkV7rcE1lDa2TS9g.png)

**10万连接**

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/kyVSg70jNKwfLGQV.png)



![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/409DKFBoYDDN2uAx.png)




**14万连接**

![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/2eKI2LeOWZlKMJzu.png)




![输入图片说明](https://raw.githubusercontent.com/GTianLuo/-/master/imgs/im/7eF1HMnQ6khx5eQw.png)


## 心跳包保活机制

在客户端与服务端建立长连接后，客户端需要定期向服务器发送心跳，告诉服务器连接仍然存活，同时服务器需要回复心跳应答。


具体实现方案：
1. client每隔3分钟发送心跳。
2. server收到心跳后回送心跳应答，若五分钟未收到心跳，则认为连接断开，释放该连接的资源。
3. client发送心跳，20s未收到心跳应答，则连续发送5个心跳(间隔20秒)，均为收到应答则断开连接。

**有tcp的keepAlive机制为什么还需要心跳？**

1. TCP的超时探查报文只能检查连接的死活，但是并不能保证连接可用。
	如果服务器CPU占用率将近100%，已经不能正常处理消息，但是连接仍然存在。
6. TCP的探测间隔时间太长，若两个小时未通信，才会发送探测报文。
7. 因为IM系统的长连接是跨网络的，如果在一定时间内未正常通信，中间的网络运营商会断开连接。


参考：
[《TCP/IP 23章》TCP的保活定时器](http://docs.52im.net/extend/docs/book/tcpip/vol1/23/)

[为什么说基于TCP的移动端IM仍然需要心跳保活？](http://www.52im.net/thread-281-1-1.html)



## 消息存储

采用Redis缓存 + mysql持久化存储



**mysql 表设计**

``im_message_send（msg_id,msg_from,msg_to,msg_seq,msg_content,send_time,msg_type）``

其中：

-   msg_id：消息ID。
-   msg_from：消息发送者UID。
-   msg_to：消息接收者。如果是单聊消息那么就是用户UID，如果是群聊消息就是群ID。
-   msg_seq：客户端发送消息时带上的序列号，主要用于消息排重以及通知客户端消息发送成功之用。
-   msg_content：消息内容。
-   send_time：消息发送时间。
-   msg_type：消息类型，如单聊、群聊消息等。




``im_message_recieve（id,msg_from,msg_to,msg_id,flag）``

其中：

-   id：这个表的ID，自增。
-   msg_from：消息发送者ID。
-   msg_to：消息接收者ID。
-   msg_id：消息ID，对应发送消息表中的ID。
-   flag：标志位，表示该消息是否已读。



**缓存设计**

在redis中对于每一个用户都有一个收信箱，发送消息时要
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTkzNDY5NjQxOCw4MDUyNzE0NzksMTIyOT
g2NjAwMCwtNDY5NjYwNDcwLDc3NzExODA5OCwzNjU5Nzc3MzIs
LTIxMzQwNzI5ODgsMjA3NTQ0NTA5NywxNzkzNTk0OTI5LDU5Nj
kzMDk1OSwtNjEwNTk1MjE0LDIwNTQ5MDI5NTQsLTEzODM3OTg4
MjksLTY2MTc5NjgzNiwtODYyODYyMDQ5LDM4ODI5NDE2Miw0NT
UxMDg0MzcsLTIwOTg1NzYwMTgsMTM1NjM2NzU2MSwtMTA2ODA0
NDM0XX0=
-->