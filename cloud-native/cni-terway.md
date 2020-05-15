# Terway调研

## 设计要点

基本连通性、IP资源高效利用、优化调度（感知IP资源分布）、垃圾回收

- 不同网络方案中的网络连通性考虑，包括pod和pod，pod和service， pod和node，pod和外部网络
- binary和daemon组件职责划分，CNI的模式是通过binary调用，但binary中又做不了所有的事情
- 如何做到高效的IPAM和资源管理，容器的网络短暂异变，Terway网络联通性主要靠编排和配置各种底层的资源，如何高效的利用资源
- 需要考虑资源的限制，云厂商对网络的资源都有配额限制，如何让调度感知
- 异常处理，异常情况下如何避免垃圾资源和配置产生，上下游的调用时存在不一致的情况时如何保障资源状态

### 网络方案

Terway有多种容器网络的配置通信方式：

- VPC（多子网模式）: Pod网段不同于节点的网络的网段，通过Aliyun VPC路由表打通不同节点间的容器网络。
- ENI: 一个弹性网卡一个POD，容器的网卡是Aliyun弹性网卡，Pod的网段和宿主机的网段是一致的。
- ENI多IP：一个Aliyun弹性网卡可以配置多个辅助VPC的IP地址，将这些辅助IP地址映射和分配到Pod中，这种Pod的网段和宿主机网段也是一致的。

### 对ENI多IP的提升

ENI多IP有两种方案：

- 策略路由方式：利用策略路由实现POD间从IP流量转发
- ipvlan I3s: Linux在4.2以上的内核中支持了ipvlan的虚拟网络，可以实现单个网卡虚拟出来多个子网卡用不同的IP地址，而Terway便利用了这种虚拟网络类型，将弹性网卡的辅助IP绑定到IPVlan的子网卡上来打通网络，使用这种模式使ENI多IP的网络结构足够简单，性能也相对veth策略路由较好。

### 资源管理

- 优化集群调度效率和避免IP浪费：增加device-plugin 让集群感知IP的分布来进行有效调度。
- IP池有MAX和MIN水位，满足快速增删POD时，避免由于访问openapi带来的响应损耗。有些预留IP可以直接使用
- IPAMD中维护了每个POD的IP租期，到期后会自动释放IP。避免kubelet删除POD时没有调用CNI释放IP。也满足statefulset在租期内保持IP。

### NetworkPolicy

采用calico felix

### QoS

Terway会取到Pod上的ingress和egress的annotation配置，然后通过配置Pod的网卡上的TC的tbf规则来实现对速度的限制。

## 总结

以下几个方面做的比较好：

1. IP池化，减轻对SDN访问访问压力，减少访问openapi，提升pod创建效率
2. device-plugin 让集群感知IP数量，IP作为一种调度资源。提升调度效率和IP有效利用率
3. ipvlan I3s提升网络性能
4. QoS增加更多流量控制手段，为资源隔离提供更健壮的保证

## Reference

[Terway插件](https://github.com/AliyunContainerService/terway/blob/master/docs/design.md)