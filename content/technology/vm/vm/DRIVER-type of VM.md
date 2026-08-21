# 虚机容器DRIVER的形式

## 架构图

```mermaid
flowchart TD
    DOCKER["Docker 网络驱动体系<br/>docker network create --driver xxx"]

    BRIDGE["bridge 桥接<br/>默认驱动 ★★★★★<br/>单机容器互通, NAT 出网"]
    HOST["host 主机网络 ★★★★<br/>直接使用宿主机网络栈<br/>无独立IP, 性能最高"]
    NONE["none 空网络 ★★<br/>只有 lo, 完全隔离"]
    OVERLAY["overlay 跨主机 ★★★<br/>Swarm 集群, VXLAN 封装"]
    MACVLAN["macvlan ★★★★<br/>容器获得真实局域网IP<br/>像真实机器一样在交换机网络中"]
    IPVLAN["ipvlan ★★<br/>与 macvlan 类似但共享网卡 MAC<br/>不大量消耗 MAC 地址"]

    DOCKER --> BRIDGE
    DOCKER --> HOST
    DOCKER --> NONE
    DOCKER --> OVERLAY
    DOCKER --> MACVLAN
    DOCKER --> IPVLAN

    subgraph PLUGINS["第三方网络插件 (CNI)"]
        CALICO["Calico"]
        FLANNEL["Flannel"]
        WEAVE["Weave"]
        CILIUM["Cilium"]
        NSX["VMware NSX"]
    end
    DOCKER -.->|"插件机制 docker plugin ls"| PLUGINS

    subgraph USAGE["生产环境选型"]
        U1["单机部署 → bridge"]
        U2["需要容器真实局域网IP → macvlan"]
        U3["Docker Swarm 集群 → overlay"]
        U4["Kubernetes → Calico/Flannel/Cilium 等 CNI"]
    end
    BRIDGE --> U1
    MACVLAN --> U2
    OVERLAY --> U3
    PLUGINS --> U4

    subgraph BSC["bridge 网络示意"]
        CA["Container A"] --- D0["docker0"] --- CB["Container B"]
        D0 --- HOSTS["Host"]
    end
    BRIDGE --> BSC

    subgraph OVSC["overlay 网络示意"]
        N1["Node1 容器A"] <==|"VXLAN 封装"| N2["Node2 容器B"]
    end
    OVERLAY --> OVSC
```

## 摘要

- Docker 中 `docker network create --driver xxx` 最常见的是 bridge，但实际上支持多种 Network Driver：bridge、host、none、overlay、macvlan、ipvlan，以及通过插件机制接入的第三方驱动。
- 查看当前 Docker 支持的驱动：`docker info` 或 `docker network ls`，常见输出包含 bridge/host/none（null）/ingress（overlay）。
- **bridge**（默认）：单机内使用、容器间互通、通过 NAT 与外部通信，适用于 Web 服务、数据库、中间件。
- **host**：容器直接使用宿主机网络栈，没有独立 IP、不需要端口映射、性能最高（`docker run --network host nginx` 等同于在宿主机直接监听 Host:80）。
- **macvlan**：容器获得真实局域网 IP（如 192.168.1.100），与 192.168.1.101 一样像真实机器存在于交换机网络中；**ipvlan** 与之类似但共享网卡 MAC，不大量消耗 MAC 地址，适合大规模部署。
- 生产环境最常见选择：单机部署 → bridge；需要容器使用真实局域网 IP → macvlan；Docker Swarm 集群 → overlay；Kubernetes 通常不直接使用这些驱动，而是通过 Calico、Flannel、Cilium 等 CNI 网络插件实现。

## 技术要点

1. **驱动查看命令**：
   ```bash
   docker info
   docker network ls
   ```
   常见输出：
   ```text
   NETWORK ID     NAME      DRIVER
   xxxx           bridge    bridge
   xxxx           host      host
   xxxx           none      null
   xxxx           ingress   overlay
   ```
2. **bridge 创建与特点**：`docker network create --driver bridge my-net`；默认驱动，单机容器挂在 docker0（或 br-xxx）下互通，出网靠 NAT，最常用。
3. **host 驱动**：`docker run --network host nginx` 直接监听 `Host:80`，没有独立 IP、无端口映射开销，性能最高；适合高性能场景、网络转发、监控组件。
4. **none 驱动**：`docker run --network none nginx` 后容器没有网卡（除了 lo），完全隔离；用 `docker exec -it nginx ip addr` 只能看到 lo；适合安全隔离和特殊网络配置。
5. **overlay 驱动**：Swarm 集群专用，多节点通信基于 VXLAN 封装（`Node1 容器A <----VXLAN----> Node2 容器B`），创建命令：
   ```bash
   docker network create \
     --driver overlay \
     my-overlay
   ```
6. **macvlan 创建示例**：容器直接使用物理网络、获得真实局域网 IP：
   ```bash
   docker network create -d macvlan \
     --subnet=192.168.1.0/24 \
     --gateway=192.168.1.1 \
     -o parent=eth0 \
     macvlan-net

   docker run -it --network macvlan-net alpine
   ```
   容器可能获得 `192.168.1.100`，与 `192.168.1.101` 像真实机器一样存在于交换机网络中；适用于网络设备模拟、需要真实 IP 的服务、广播/组播场景。
7. **ipvlan 创建示例**：与 macvlan 类似但共享父网卡 MAC，不大量消耗 MAC 地址，大规模部署更友好，适用于数据中心、云环境：
   ```bash
   docker network create -d ipvlan \
     --subnet=192.168.1.0/24 \
     -o parent=eth0 \
     ipvlan-net
   ```
8. **第三方驱动插件机制**：`docker plugin ls` 查看已装插件，常见插件有 Calico、Flannel、Weave、Cilium、VMware NSX；例如：
   ```bash
   docker network create \
     --driver calico \
     calico-net
   ```
9. **使用频率对照表**：

   | Driver | 使用频率 | 场景 |
   |---|---|---|
   | bridge | ★★★★★ | 单机容器 |
   | host | ★★★★ | 高性能应用 |
   | macvlan | ★★★★ | 容器直连物理网络 |
   | overlay | ★★★ | Swarm 集群 |
   | ipvlan | ★★ | 大规模网络 |
   | none | ★★ | 隔离环境 |

## 原文内容

在 Docker 中，`docker network create --driver xxx` 最常见的是 bridge，但实际上支持多种 Network Driver。

先查看当前 Docker 支持的驱动：`docker info`，或者：

```bash
docker network ls
```

常见输出：

```text
NETWORK ID     NAME      DRIVER
xxxx           bridge    bridge
xxxx           host      host
xxxx           none      null
xxxx           ingress   overlay
```

### 1. bridge（桥接）

默认驱动。

特点：

- 单机内使用
- 容器间互通
- 通过 NAT 与外部通信
- 最常用

创建命令：

```bash
docker network create --driver bridge my-net
```

网络示意：

```text
Container A
      |
   docker0
      |
Container B
      |
   Host
```

适用于：Web 服务、数据库、中间件。

### 2. host（主机网络）

容器直接使用宿主机网络栈。

特点：

- 没有独立 IP
- 不需要端口映射
- 性能最高

例如：

```bash
docker run --network host nginx
```

等同于在宿主机直接监听：`Host:80`

适用于：高性能场景、网络转发、监控组件。

### 3. none（空网络）

特点：

- 没有网卡（除了 lo）
- 完全隔离

启动：

```bash
docker run --network none nginx
```

查看：

```bash
docker exec -it nginx ip addr
```

只能看到：`lo`

适用于：安全隔离、特殊网络配置。

### 4. overlay（跨主机网络）

Swarm 集群专用。

特点：

- 多节点通信
- VXLAN 封装
- 容器跨主机互联

创建：

```bash
docker network create \
  --driver overlay \
  my-overlay
```

示意：

```text
Node1                Node2
容器A <----VXLAN----> 容器B
```

适用于：Docker Swarm、微服务集群。

### 5. macvlan（直接使用物理网络）

容器获得真实局域网 IP。

启动：

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net
```

```bash
docker run -it --network macvlan-net alpine
```

容器可能获得：`192.168.1.100`，与 `192.168.1.101` 像真实机器一样存在于交换机网络中。

适用于：网络设备模拟、需要真实 IP 的服务、广播/组播场景。

### 6. ipvlan

与 macvlan 类似，但共享网卡 MAC。

特点：

- 不大量消耗 MAC 地址
- 大规模部署更友好

创建：

```bash
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  -o parent=eth0 \
  ipvlan-net
```

适用于：数据中心、云环境。

### 7. 第三方网络驱动

Docker 支持插件机制。查看：

```bash
docker plugin ls
```

常见插件：Calico、Flannel、Weave、Cilium、VMware NSX。例如：

```bash
docker network create \
  --driver calico \
  calico-net
```

### 生产环境最常见选择

对于大多数企业环境：

- 单机部署 → bridge
- 需要容器使用真实局域网 IP → macvlan
- Docker Swarm 集群 → overlay

Kubernetes 环境通常不直接使用这些驱动，而是通过 Calico、Flannel、Cilium 等 CNI 网络插件实现。

### Driver 使用频率对照

| Driver | 使用频率 | 场景 |
|---|---|---|
| bridge | ★★★★★ | 单机容器 |
| host | ★★★★ | 高性能应用 |
| macvlan | ★★★★ | 容器直连物理网络 |
| overlay | ★★★ | Swarm 集群 |
| ipvlan | ★★ | 大规模网络 |
| none | ★★ | 隔离环境 |
