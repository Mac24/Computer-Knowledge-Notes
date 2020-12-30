## 临安项目网络故障排查总结

### 1.问题描述

k8s集群中master和node节点上的Pod IP是相互之间ping不通的

- master和node的配置如下：

|           | master           | node             |
| --------- | ---------------- | ---------------- |
| eth0      | 10.33.188.138/24 | 10.33.188.142/24 |
| flannel.1 | 10.244.0.0/32    | 10.244.1.0/32    |
| cni0      | 10.244.0.1/24    | 10.244.1.1/24    |
| Pod IP    | 10.244.0.4       | 10.244.1.5       |

- **不同节点之间是可以ping通的**

  [root@k8s-master1 flannel]# ping 10.33.188.142
  PING 10.33.188.142 (10.33.188.142) 56(84) bytes of data.
  64 bytes from 10.33.188.142: icmp_seq=1 ttl=64 time=0.354 ms
  64 bytes from 10.33.188.142: icmp_seq=2 ttl=64 time=0.372 ms
  64 bytes from 10.33.188.142: icmp_seq=3 ttl=64 time=0.365 ms

  

  [root@k8s-node2 ~]# ping 10.33.188.138
  PING 10.33.188.138 (10.33.188.138) 56(84) bytes of data.
  64 bytes from 10.33.188.138: icmp_seq=1 ttl=64 time=1.17 ms
  64 bytes from 10.33.188.138: icmp_seq=2 ttl=64 time=0.574 ms
  64 bytes from 10.33.188.138: icmp_seq=3 ttl=64 time=0.775 ms


- **同一节点上的Pod是可以ping通的**
  [root@k8s-node2 ~]# ping 10.244.1.5
  PING 10.244.1.5 (10.244.1.5) 56(84) bytes of data.
  64 bytes from 10.244.1.5: icmp_seq=1 ttl=64 time=0.125 ms
  64 bytes from 10.244.1.5: icmp_seq=2 ttl=64 time=0.086 ms
  64 bytes from 10.244.1.5: icmp_seq=3 ttl=64 time=0.089 ms


- **master节点上的Pod可以ping通node的IP，但是不能ping通node上pod的IP**
  [root@k8s-master1 flannel]# docker exec -it 0ffdb1809598 sh
  / # ping 10.33.188.142
  PING 10.33.188.142 (10.33.188.142): 56 data bytes
  64 bytes from 10.33.188.142: seq=0 ttl=63 time=0.500 ms
  64 bytes from 10.33.188.142: seq=1 ttl=63 time=0.481 ms
  64 bytes from 10.33.188.142: seq=2 ttl=63 time=0.401 ms

  

  / # ping 10.244.1.5
  PING 10.244.1.5 (10.244.1.5): 56 data bytes

  --- 10.244.1.5 ping statistics ---
  49 packets transmitted, 0 packets received, 100% packet loss

### 2.排查思路

在跨节点不能通信的时候，首先想到的是节点DNS问题，通过排查kube-dns发现没有问题

#### 2.1.排查kube-dns

整个的排查过程，参考[官方文档](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)的排查思路，并没有解决问题。

然后针对列举出的可能存在的问题，进行了逐一的排除

#### 2.2排查可能存在的问题

- 排除了Linux内核版本的问题。在同样的Linux内核版本上，本地部署的k8s集群能够正常工作
- 排除了flannel版本的问题。分别使用了最新的v0.13.0，之前使用的v0.10.0，都没有效果
- 排除了Iptables规则的问题。针对Iptables规则中不同的地方进行修改，仍然没有效果

在多种方法问题还没有解决的情况下，决定研究一下flannel的具体工作流程

#### 2.3排查flannel

通过研究flannel的整个工作流程，通过`tcpdump`对eth0,flannel.1,cni0进行抓包。flannel的具体工作流程，参见`flannel的工作流程`。

我在一个正常工作的k8s集群中，通过master上的Pod ping node上的Pod，对flannel数据包的转发过程进行抓包，具体的流程如下：

`veth -> cni0 -> flannl.1 -> node flannel.1 -> node cni0 -> node veth`

对存在问题的k8s集群，采用同样的方式进行抓包，存在的问题如下：

- 能在发起ping的master机器上抓到包，但目标机器上抓不到数据包

- 虽然在master机器上抓到数据包，但是跟正常集群的数据包转发也存在不同

  - 在cni0上数据包转发是一样的
  - 在flannel.1上不同

- 针对上述存在的不同，并不知道是什么原因造成的。后来杭州同事提醒，说临安虚拟机的网络模式使用的是NAT模式，而公司内部的虚拟机的网络模式使用的是桥接模式。

### 3.解决方案

- **flannel中的`vxlan`协议**：两个节点上的pod借助flannel隧道进行通信。因为它有额外开销(封包和解包)，所以性能有点低。

- **flannel的另一种协议叫`host-gw`**：即Node节点把自己的网络接口当做pod的网关使用，从而使不同节点上的node进行通信。

  这个性能比vxlan高，因为它没有额外开销。缺点就是各node节点必须在同一个网段中 。

- 在本集群中pod所在节点都在同一个网段中 ，可以让`vxlan`也支持`host-gw`的功能， 即直接通过物理网卡的网关路由转发，而不用隧道flannel叠加，从而提高了`vxlan`的性能，这种flannel的功能叫`directrouting`。

- kubeadm init 使用的`--pod-network-cidr=10.244.0.0/16 `参数和 kube-flannel.yml中的network保持一致。并且增加一个"`Directrouting":true`参数

```yaml
{
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "Directrouting": true #加一行这个
      }
    }
```



在k8s集群重置的过程中，执行了`kubeadm reset`后，要执行以下命令

```shell
set -x
rm -rf /etc/cni
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ifconfig flannel.1 down
ip link del flannel.1
ifconfig cni0 down
ip link del cni0
ifconfig
systemctl restart docker
systemctl restart kubelet
set +x
```

