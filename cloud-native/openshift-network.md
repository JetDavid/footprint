# Openshift网络初学

## 核心问题

解决openshift本身，以及多云场景下的网络互通问题。具体关注：

## pod间相互访问

- pod可获得单独ip，意味着在很多操作上如同主机和虚拟机一致：端口管理，网络，名字解析，服务发现，LB，应用配置，迁移
- 单集群内部，pod间保持原有k8s的方式。openshift推荐自身的SDN（ovs）方案;
- 跨集群：利用ingress（http或https）互访service，帮助service对外暴露，route做切分

## 利用dns解耦前後端服務可用性依賴

具体场景：故障，升级，改配置

- 基于dns的服务发现，使用coreDNS

## 運維管理多k8s集群：堡壘機的架設

- 利用堡垒机管理openshift集群

## 多租戶

- openshift自身提供project概念，可以利用network policy完成多租户隔离；
- 基于istio的多租户隔离[方案](https://www.openshift.com/blog/connecting-multiple-openshift-sdns-with-a-network-tunnel)

## 網絡流量管理和安全管理

multiple networks用来解决流量管理，安全隔离场景下的网络需求。基于openshift-sdn基础上可以增加macvlan, host-device, bridge, ipvlan的4种其他网络模型，协助业务完成流量分离和安全隔离。

## 性能提升

主要利用SRIOV，DPDK，RDMA等技术。

## 參考文獻

- [OpenShift Dedicated 4 -- Networking](https://docs.openshift.com/dedicated/4/getting_started/dedicated-networking.html)
- [OpenShift Container Platform 4.3 -- Networking](https://docs.openshift.com/container-platform/4.3/networking/understanding-networking.html)
