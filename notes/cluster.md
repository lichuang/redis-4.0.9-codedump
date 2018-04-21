# 集群

## 初始化
如果配置为集群模式，将调用clusterInit函数进行初始化，其中会再创建一个端口为基础port+10000的监听端口，负责接收集群内部的请求，其回调函数是clusterAcceptHandler，接收连接之后读取请求数据的回调函数是clusterReadHandler，处理请求数据的函数clusterProcessPacket。

## meet指令添加一个node到集群中

1.  客户端发送cluster meet命令到server，server收到时的处理（函数clusterStartHandshake）：尝试连接cport（port+10000），连接成功后创建一个clusterNode结构体，将状态标记为CLUSTER_NODE_MEET。
2.  在clusterCron函数中，判断如果是CLUSTER_NODE_MEET状态，将发送meet命令。
3.  对端收到之后应答pong命令(函数clusterProcessPacket)。

## 更新slot配置

通过定时的ping、pong消息clusterMsg中的myslots数据来向其他节点同步本节点管理的slots信息。

## 向其他节点发送gossip消息的逻辑
并不是随时都会向所有节点发送gossip消息。

在clusterCron函数中，每10次调用该函数，才会尝试向一个节点发送gossip消息。

选择发送目标节点的逻辑是：
1.  随机选择集群中的5个节点。
2.  选择其中最长时间没有进行通信的节点。

发送gossip消息时也有讲究，并不会一次性全部通过所有节点信息到该节点。同样是随机选出节点进行同步（函数clusterSendPing）

## 迁移slot数据

1.  cluster setslot importing 命令告知目标节点准备导入数据，将修改importing_slots_from数据结构
2.  cluster setslot migrate 命令告知源节点准备迁移数据，将修改migrating_slots_to数据结构。
3.  调用migrate命令开始迁移数据。

在迁移过程中，如果仍然有请求到源节点查询在迁移中的slot中的key而又查询不到，将返回ASK错误，告知这些数据已经迁移到哪些节点了。


