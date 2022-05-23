# 多网卡管理

从 v1.1 版本开始，Kube-OVN 可以为其他 CNI 网络插件，例如 macvlan、vlan、host-device 等插件提供集群级别的 IPAM 能力，其他网络插件也可以使用到 Kube-OVN 中子网以及固定 IP 功能。从 1.7.1 版本之后容器内的多块网卡可以都使用 Kube-OVN 提供网络和 IPAM 能力。

## 工作原理

通过使用 [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni), 我们可以给一个 Pod 添加多块不同网络的网卡。然而我们仍然缺乏对集群范围内不同网络的 IP 地址进行管理的能力。在 Kube-OVN 中，我们已经能够通过 Subnet 和 IP 的 CRD 来进行 IP 的高级管理，例如子网管理，IP 预留，随机分配，固定分配等。现在我们对子网进行扩展，来接入其他不同的网络插件，使得其他网络插件也可以使用 Kube-OVN 的IPAM功能。

### 工作流程

![work-flow](https://raw.githubusercontent.com/alauda/kube-ovn/master/docs/multi-nic.png)

上图展示了如何通过 Kube-OVN 来管理其他网络插件的 IP 地址。其中容器的 eth0 网卡接入 OVN 网络，net1 网卡接入其他 CNI 网络。net1 网络的网络定义来自于 multus-cni 中的 NetworkAttachmentDefinition CRD。

当 Pod 创建时，kube-ovn-controller 会监听到 Pod 添加事件，并根据 Pod 中的 annotation 去寻找到对应的 Subnet CRD 并从中进行 IP  的分配和管理，并将 Pod 所分配到的地址信息写回到 Pod annotation 中。

在容器所在机器的 CNI 可以通过在配置中配置 kube-ovn-cni 作为 ipam 插件, kube-ovn-cni 将会读取 Pod annotation 并将地址信息通过CNI 协议的标准格式返回给相应的 CNI 插件。

## 使用方法

### 安装 Kube-OVN 和 Multus

请参考 [Kube-OVN installation](https://github.com/alauda/kube-ovn/wiki/%E4%B8%80%E9%94%AE%E5%AE%89%E8%A3%85) 和 [Multus how to use](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md) 来安装 Kube-OVN 和 Multus-CNI。

### 附属网卡为非 Kube-OVN 类型 CNI 提供的网络

#### 创建 network attachment definition

这里我们使用 macvlan 作为容器网络的第二个网络，并将其 ipam 设置为 kube-ovn

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan
  namespace: default
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "kube-ovn",
        "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
        "provider": "macvlan.default"
      }
    }'
```
`spec.config.ipam.type`: 需要为 `kube-ovn` 来调用 kube-ovn 的插件来获取地址信息

`server_socket`: Kube-OVN 通信使用的 socket 文件。 默认位置为 `/run/openvswitch/kube-ovn-daemon.sock`

`provider`: 当前 NetworkAttachmentDefinition 的 `<name>.<namespace>` , Kube-OVN 将会使用这些信息找到对应的 Subnet 资源。

### 附属网卡为 Kube-OVN 类型网卡

#### 创建 network attachment definition, 并将 provider 的后缀设置为 ovn
```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: attachnet
  namespace: default
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "kube-ovn",
      "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
      "provider": "attachnet.default.ovn"
    }'
```

`spec.config.type`: 设置为 `kube-ovn` 来触发 CNI 插件使用 Kube-OVN 子网

`server_socket`: Kube-OVN 通信使用的 socket 文件。 默认位置为 `/run/openvswitch/kube-ovn-daemon.sock`

`provider`: 当前 NetworkAttachmentDefinition 的 `<name>.<namespace>.ovn` , Kube-OVN 将会使用这些信息找到对应的 Subnet 资源，注意后缀需要设置为 ovn。

### 创建一个 Kube-OVN Subnet

创建一个 Kube-OVN Subnet,设置对应的 CIDR 和 exclude_ips, `provider` 应该设置为对应的 NetworkAttachmentDefinition的`<name>.<namespace>`, 例如用macvlan提供附加网卡，创建subnet如下
```yaml
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: macvlan
spec:
  protocol: IPv4
  provider: macvlan.default
  cidrBlock: 172.17.0.0/16
  gateway: 172.17.0.1
  excludeIps:
  - 172.17.0.0..172.17.0.10
```

gateway, private, nat 只对 provider 类型为 ovn 的网络生效，不适用于 attachment network。

如果以kube-ovn作为附加网卡，则
provider 应该设置为对应的 NetworkAttachmentDefinition的`<name>.<namespace>.ovn`,要以`ovn`作为后缀结束，用kube-ovn提供附加网卡，创建subnet示例如下
```yaml
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: attachnet
spec:
  protocol: IPv4
  provider: attachnet.default.ovn
  cidrBlock: 172.17.0.0/16
  gateway: 172.17.0.1
  excludeIps:
  - 172.17.0.0..172.17.0.10
```

### 创建一个多网络的 Pod

对于地址随机分配的 Pod，只需要添加如下 annotation `k8s.v1.cni.cncf.io/networks`,取值为对应的 NetworkAttachmentDefinition的`<name>`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: samplepod
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine

```

### 创建固定 IP 的 Pod

对于固定 IP 的 Pod，添加 `<networkAttachmentName>.<networkAttachmentNamespace>.kubernetes.io/ip_address` annotations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-ip
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan
    ovn.kubernetes.io/ip_address: 10.16.0.15
    ovn.kubernetes.io/mac_address: 00:00:00:53:6B:B6
    macvlan.default.kubernetes.io/ip_address: 172.17.0.100
    macvlan.default.kubernetes.io/mac_address: 00:00:00:53:6B:BB
spec:
  containers:
  - name: static-ip
    image: nginx:alpine
```

### 创建使用固定 IP 的工作负载

对于使用 ippool 的工作负载, 添加 `<networkAttachmentName>.<networkAttachmentNamespace>.kubernetes.io/ip_pool` annotations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: static-workload
  labels:
    app: static-workload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-workload
  template:
    metadata:
      labels:
        app: static-workload
      annotations:
        k8s.v1.cni.cncf.io/networks: macvlan
        ovn.kubernetes.io/ip_pool: 10.16.0.15,10.16.0.16,10.16.0.17
        macvlan.default.kubernetes.io/ip_pool: 172.17.0.200,172.17.0.201,172.17.0.202
    spec:
      containers:
      - name: static-workload
        image: nginx:alpine
```