## K8s



docker   和   虚拟机

物理机是一栋楼，虚拟机就是一个个套间，容器技术就是一个个隔断。

![image-20210407160722299](C:\Users\Seven\AppData\Roaming\Typora\typora-user-images\image-20210407160722299.png)



Master 主控节点  node 工作节点

Api  server :集群统一入口，以restful风格进行请求的分发和处理，交于etcd存储

Controller manager：处理集群中常规后台任务，一个资源对应一个控制器，

scheduler：节点调度器，选择一个node节点进行应用的部署

etcd ： 存储系统，用于保存集群中相关的数据



Pod ： 最小部署单元，共享网络，一组容器的集合，生命周期是短暂的

controller：确保预期的pod副本的数量，(无状态应用部署 有状态应用部署)

