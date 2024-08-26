## NVIDIA GPU Operator
Kubernetes 通过设备插件框架提供对特殊硬件资源的访问，例如 NVIDIA GPU、网卡 (NIC)、Infiniband 适配器和其他设备。然而，配置和管理包含这些硬件资源的节点需要设置多个软件组件，如驱动程序、容器运行时或其他库，这些操作复杂且容易出错。NVIDIA GPU Operator 利用 Kubernetes 中的 Operator 框架，自动化管理配置 GPU 所需的所有 NVIDIA 软件组件。这些组件包括：

- NVIDIA 驱动程序（用于启用 CUDA）
- Kubernetes GPU 设备插件
- NVIDIA 容器运行时
- 自动节点标记
- 基于 DCGM 的监控系统
- 以及其他相关组件

GPU Operator 使 Kubernetes 集群的管理员能够像管理 CPU 节点一样管理 GPU 节点。管理员无需为 GPU 节点准备特定的操作系统镜像，而是可以依赖通用的操作系统镜像用于 CPU 和 GPU 节点，然后依赖 GPU Operator 来配置 GPU 所需的软件组件。

需要注意的是，GPU Operator 尤其适用于需要快速扩展 Kubernetes 集群的场景——例如在云端或本地快速添加 GPU 节点并管理其底层软件组件的生命周期。由于 GPU Operator 将所有内容都以容器的形式运行（包括 NVIDIA 驱动程序），管理员可以轻松替换各种组件——只需通过启动或停止容器即可。

## nvidia network operator
nvidia network operator的目标是管理与网络相关的组件，同时在 Kubernetes 集群中支持 RDMA 和 GPUDirect RDMA 工作负载的执行。这包括：

- NVIDIA 网络驱动程序，以启用高级功能（如 enp1 netdcreate 和 NV-IPAM IPPool）。
- Kubernetes 设备插件，提供加速网络所需的硬件资源。
- Kubernetes 辅助网络组件，用于网络密集型工作负载。
Network Operator 与 GPU Operator 协同工作，在兼容的系统上启用 GPU-Direct RDMA 功能。

#### Kubernetes Node Feature Discovery (NFD)
NVIDIA Network Operator 依赖节点标签（Node labeling）来将集群配置为所需状态。默认情况下，通过 HELM chart 安装会部署 Node Feature Discovery (NFD) v0.13.2 或更新版本。NFD 用于为节点添加以下标签：

- PCI 供应商和设备信息
- RDMA 功能
- GPU 特性*

### used CRD
#### NICClusterPolicy CRD
CRD that defines a Cluster state for Mellanox Network devices.
NICClusterPolicy CRD 规范包含以下定义值：
- ofedDriver: 在支持 Mellanox 的节点上部署的 OFED 驱动容器。
    OFED 驱动容器旨在作为主机安装的替代方案，只需在主机上部署该容器镜像即可完成以下任务：
    - 重新加载由 Mellanox OFED 提供的内核模块。
    - 将容器的根文件系统挂载到 /run/mellanox/drivers/ 目录。如果将此目录映射到主机上，容器的内容将可供主机或其他容器共享。一个应用场景是编译 Nvidia Peer Memory 客户端模块。
- rdmaSharedDevicePlugin: RDMA 共享设备插件及相关配置。
    RDMA 共享设备插件，这是一个简单的 RDMA 设备插件，支持 InfiniBand (IB) 和 RoCE HCA（主机通道适配器）。该插件以 DaemonSet 的形式运行，容器镜像位于 mellanox/k8s-rdma-shared-dev-plugin。
    RDMA 共享设备插件应部署在以下节点上：
    - 具备 RDMA 功能的硬件
    - 已加载 RDMA 内核栈
    为了进行正确的节点选择，可以使用 Node Feature Discovery (NFD) 来发现节点的能力，并将其暴露为节点标签。
    rdmaSharedDevicePlugin的功能和sriovdeviceplugin的功能类似
- sriovDevicePlugin: SR-IOV 网络设备插件及相关配置。
- ibKubernetes: InfiniBand Kubernetes 及相关配置。
    InfiniBand Kubernetes 提供了一个名为 ib-kubernetes 的守护进程，它与 InfiniBand SR-IOV CNI 和 Multus CNI 协同工作。ib-kubernetes 监听 Kubernetes Pod 对象的变化（创建/更新/删除），读取 Pod 的网络配置，并获取相应的网络 CRD。然后，它读取 PKey，并将新生成的 GUID 或预定义的 GUID（位于 CRD 的 cni-args 中的 guid 字段）添加到该 PKey 中，适用于带有 mellanox.infiniband.app 注解的 Pods。
    
    InfiniBand SR-IOV CNI将具备 SR-IOV 能力的IB网络接口卡 (NIC) 通过引入物理功能 (PF) 和虚拟功能 (VF) 的概念来工作。
    - PF (Physical Function): 由主机使用，VF 配置通过 PF 应用。PF 是网络接口卡的主要功能，用于管理和配置虚拟功能。
    - VF (Virtual Function): 每个 VF 可以被视为一个独立的物理 NIC，并分配给一个容器。VF 是 PF 创建的虚拟化实例，用于提供隔离的网络连接。
    IB-SRIOV-CNI support Mellanox ConnectX®-4/ConnectX®-5/ConnectX®-6 adapter cards.

    这种架构使得每个 VF 能够作为单独的网络接口使用，从而为容器提供高效的网络访问。
- secondaryNetwork: 指定要部署的组件，以便在 Kubernetes 中支持二级网络。它包含以下可选部署的组件：
    - Multus-CNI: 支持 Kubernetes 二级网络的 CNI 代理插件。
    - CNI plugins: 目前仅支持 container-networking-plugins。
    - IP Over Infiniband (IPoIB) CNI Plugin: 允许用户创建 IPoIB 子链路并将其移动到 pod 中。
        给IB网卡的接口配置IP地址
    - IPAM CNI: 包括 Whereabouts IPAM CNI 及相关配置。
- nvIpam: NVIDIA Kubernetes IPAM 及相关配置。
    NVIDIA IPAM plugin supports allocation of IP ranges and Network prefixes for nodes.
    NVIDIA IPAM plugin currently support only IP allocation for K8s Secondary Networks. e.g Additional networks provided by multus CNI plugin.

##### Example for NICClusterPolicy resource
In the example below we request OFED driver to be deployed together with RDMA shared device plugin.
```
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    image: mofed
    repository: nvcr.io/nvidia/mellanox
    version: 23.04-0.5.3.3.1
    startupProbe:
      initialDelaySeconds: 10
      periodSeconds: 10
    livenessProbe:
      initialDelaySeconds: 30
      periodSeconds: 30
    readinessProbe:
      initialDelaySeconds: 10
      periodSeconds: 30
  rdmaSharedDevicePlugin:
    image: k8s-rdma-shared-dev-plugin
    repository: ghcr.io/mellanox
    version: sha-fe7f371c7e1b8315bf900f71cd25cfc1251dc775
    # The config below directly propagates to k8s-rdma-shared-device-plugin configuration.
    # Replace 'devices' with your (RDMA capable) netdevice name.
    config: |
      {
        "configList": [
          {
            "resourceName": "rdma_shared_device_a",
            "rdmaHcaMax": 63,
            "selectors": {
              "vendors": ["15b3"],
              "deviceIDs": ["1017"],
              "ifNames": ["ens2f0"]
            }
          }
        ]
      }
  secondaryNetwork:
    cniPlugins:
      image: plugins
      repository: ghcr.io/k8snetworkplumbingwg
      version: v1.2.0-amd64
    multus:
      image: multus-cni
      repository: ghcr.io/k8snetworkplumbingwg
      version: v3.9.3
      # if config is missing or empty then multus config will be automatically generated from the CNI configuration file of the master plugin (the first file in lexicographical order in cni-conf-dir)
      config: ''
    ipamPlugin:
      image: whereabouts
      repository: ghcr.io/k8snetworkplumbingwg
      version: v0.6.1-amd64
```

NicClusterPolicy with NVIDIA Kubernetes IPAM configuration
```
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    image: mofed
    repository: nvcr.io/nvidia/mellanox
    version: 23.04-0.5.3.3.1
    startupProbe:
      initialDelaySeconds: 10
      periodSeconds: 10
    livenessProbe:
      initialDelaySeconds: 30
      periodSeconds: 30
    readinessProbe:
      initialDelaySeconds: 10
      periodSeconds: 30
  rdmaSharedDevicePlugin:
    image: k8s-rdma-shared-dev-plugin
    repository: ghcr.io/mellanox
    version: sha-fe7f371c7e1b8315bf900f71cd25cfc1251dc775
    # The config below directly propagates to k8s-rdma-shared-device-plugin configuration.
    # Replace 'devices' with your (RDMA capable) netdevice name.
    config: |
      {
        "configList": [
          {
            "resourceName": "rdma_shared_device_a",
            "rdmaHcaMax": 63,
            "selectors": {
              "vendors": ["15b3"],
              "deviceIDs": ["101b"]
            }
          }
        ]
      }
  secondaryNetwork:
    cniPlugins:
      image: plugins
      repository: ghcr.io/k8snetworkplumbingwg
      version: v1.2.0-amd64
    multus:
      image: multus-cni
      repository: ghcr.io/k8snetworkplumbingwg
      version: v3.9.3
      config: ''
  nvIpam:
    image: nvidia-k8s-ipam
    repository: ghcr.io/mellanox
    version: v0.1.2
    enableWebhook: false
```
