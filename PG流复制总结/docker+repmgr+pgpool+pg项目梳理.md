# 流复制集群搭建以及功能测试

项目需求

启动三个容器作为PG的3个节点，1主2从，实现主节点故障，备节点能够提升为主节点，继续对外提供服务。

1、搭建流程

原理

部署完成后，每个节点都有自己的元数据表，记录所有集群节点的信息；每个节点都有自己的repmgrd守护进程来监控节点数据库状态。其中主节点守护进程主要用来监控本节点数据库服务状态，备节点守护进程主要用来监控主节点和本节点数据库服务状态。在发生Auto Failover时，备节点在尝试N次连接主节点失败后，repmgrd会在所有备节点中选举一个候选备节点（选举机制参考以下Tips）提升为新主节点，然后其他备节点去Follow到该新主上，至此，形成一个新的集群状态。

Repmgr选举候选备节点会以以下顺序选举：LSN ， Priority， Node_ID。
系统会先选举一个LSN比较大者作为候选备节点；如LSN一样，会根据Priority优先级进行比较，该优先级是在配置文件中进行参数配置；如优先级也一样，会比较节点的Node ID，小者会优先选举。

步骤：拉取镜像，保存为文件，文件上传至本地仓库，启动容器。

2、功能验证

停止主，查看其他两个从节点有一个被提升为了新的主节点，变为一主一从；再停止当前的主节点，从节点又被提升为新的主节点，变为只有一个主节点了。

3、遇到的问题及应对方法

