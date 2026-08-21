# 虚机容器网桥的建立与两个容器的互通

## 架构图

```mermaid
flowchart TB
    subgraph 宿主机["宿主机 dw-ap01（10.10.10.22/24）"]
        eth0["eth0<br/>10.10.10.22/24<br/>物理网卡"]

        subgraph 默认网络["默认 Bridge 网络（bridge）"]
            docker0["docker0 网桥<br/>172.17.0.1/16<br/>Docker 安装时自动创建"]
            veth1["veth34a5eca@if201<br/>宿主机侧 veth"]
            subgraph 容器PG["容器网络命名空间 net:[4026532241]"]
                pg["PostgreSQL 容器<br/>eth0@if201: 172.17.0.2/16<br/>端口 5432"]
            end
            veth1 --- pg
            docker0 --- veth1
        end

        subgraph 自定义网络["自定义网络 chery-tools_default（app-net）"]
            br["br-4d44c079d811 网桥<br/>172.18.0.1/16<br/>docker network create / Compose 创建"]
            veth2["veth61d83e0@if2<br/>宿主机侧 veth"]
            subgraph 容器Web["容器网络命名空间 net:[4026532171]"]
                web["chery-tools 容器（uvicorn）<br/>eth0@if2: 172.18.0.2/16<br/>端口 8000"]
            end
            veth2 --- web
            br --- veth2
        end

        eth0 --- docker0
        eth0 --- br
    end

    外部客户端["外部客户端"] -- "DNAT: 443 → 172.18.0.2:8000" --> eth0
    本机进程["宿主机本机进程"] -- "DNAT: 127.0.0.1:55432 → 172.17.0.2:5432" --> docker0

    web -. "跨网桥默认隔离<br/>（iptables DROP 阻断）" .-> pg
    web == "推荐：同一网络互通<br/>docker network connect app-net postgres" ==> pg

    容器出网["外部网络"] <-. "MASQUERADE<br/>172.17.0.0/16 / 172.18.0.0/16 SNAT" .- eth0
```

## 摘要

- 主机上存在两个独立的 Docker Bridge 网络：`docker0`（172.17.0.1/16，Docker 默认）和 `br-4d44c079d811`（172.18.0.1/16，对应自定义网络 `chery-tools_default`）。
- 自定义网桥可通过 `docker network create` 或 Docker Compose 自动创建，Linux 侧接口名规则为 `br-<网络ID前12位>`。
- 不同 Bridge 网络默认互相隔离：Docker 通过 iptables（DOCKER 链中的 DROP 规则）阻断跨网桥访问，避免容器间意外互通。
- 网络互通的推荐方案是把两个容器加入同一个自定义网络（自定义网络自带 DNS，可按容器名互访）；也可让一个容器加入多个网络（双网卡），或用宿主机路由（不推荐）。
- Docker 会自动为网桥配置 NAT：MASQUERADE 实现容器出网，DNAT 实现端口发布（如宿主机 443 → 172.18.0.2:8000）。
- 每个容器拥有独立的网络命名空间（netns），通过 veth pair 与宿主机侧网桥连接，可用 `lsns`、`nsenter` 查看与进入。

## 技术要点

1. **两种网桥的来源**：`docker0` 是 Docker 安装时自动创建的默认 Bridge 网络；`br-4d44c079d811` 是执行 `docker network create app-net` 或 `docker compose up`（自动生成 `<project>_default` 网络）后，Linux 自动创建的网桥接口，名字为 `br-` + 网络 ID 前 12 位。
2. **默认网段分配**：`docker0` 使用 172.17.0.0/16，网关 172.17.0.1；每新建一个 bridge 网络，网段依次递增（172.18.0.0/16、172.19.0.0/16……）。容器地址从 `.2` 开始（`.1` 是网桥自身网关）。可用 `--subnet` / `--gateway` 手动指定。
3. **veth pair 连接容器与网桥**：每个容器启动时创建一对 veth，一端在容器内命名为 `eth0@ifN`，另一端（如 `veth61d83e0@if2`）挂在宿主机网桥上，`master docker0` / `master br-xxx` 即其归属。
4. **网络隔离机制**：iptables 的 DOCKER 链中 `! -i br-xxx -o br-xxx -j DROP` 规则丢弃所有非本网桥进入、发往本网桥的流量，因此 172.17.0.2 与 172.18.0.2 之间默认 ping 不通。
5. **方法一（推荐）——同一网络**：`docker run --network app-net nginx` 让多个容器加入同一自定义网络，Docker 内置 DNS 使它们可直接用容器名/别名互相解析访问。
6. **方法二——容器多网络（双网卡）**：`docker network connect bridge web-container` 给运行中容器追加网卡，容器同时持有 172.17.0.10 和 172.18.0.10，像双网卡服务器一样桥接两个网络。
7. **方法三——宿主机路由（不推荐）**：`sysctl -w net.ipv4.ip_forward=1` 开启转发并放行 iptables，实现 172.17.0.0/16 ↔ 172.18.0.0/16 互通，但管理复杂、绕过了 Docker 的隔离设计。
8. **方法四——Compose 多网络分层**：在 `docker-compose.yml` 中定义 `frontend` / `backend` 等多个网络，nginx 同时接入两层，app、mysql 只在 backend，实现网络分层的中间层拓扑（企业常用）。
9. **NAT 规则自动化**：创建带端口映射的容器时 Docker 自动写入 DNAT（443 → 172.18.0.2:8000、127.0.0.1:55432 → 172.17.0.2:5432）与 MASQUERADE（源 172.17/172.18 出网伪装），可通过 `iptables -t nat -L -n -v` 查看。
10. **排查工具链**：`docker network inspect <net> --format '{{len .Containers}}'` 查网络内容器数；`lsns -t net` 列出网络命名空间；`docker inspect -f '{{.State.Pid}}'` 取容器 PID；`nsenter -t <pid> -n ip addr` 进入容器 netns 查看其内部视角的 eth0。

## 原文内容

从你的输出可以确认：确实是两个独立的 Docker 网络。

### 1. 这两个网桥是什么？如何创建的？

#### docker0

这是 Docker 安装时自动创建的默认 Bridge 网络。

| 网络 ID | 名称 | 驱动 |
|---------|------|------|
| 4c5132dbffdd | bridge | bridge |

对应的 Linux 网桥就是 `docker0`。容器如果没有指定网络（如 `docker run nginx`），默认加入 `bridge` 网络，网段为 `172.17.0.0/16`，网关 `172.17.0.1/16`。

#### br-4d44c079d811

这是 Docker 创建的自定义 Bridge 网络，通常由以下方式产生：

**Docker 命令：**

```bash
docker network create app-net
```

查看：

```bash
docker network inspect app-net
```

结果：

```text
Subnet: 172.18.0.0/16
```

Linux 会自动生成 `br-4d44c079d811`。

**Docker Compose：**

```yaml
services:
  web:
    image: nginx

networks:
  default:
```

执行 `docker compose up -d` 后，Docker 会自动创建 `project_default` 网络，对应 Linux 中通常变成 `br-xxxxxxxx`。

#### 查看实际网络名

```bash
docker network ls
```

例如：

```text
NETWORK ID          NAME
4d44c079d811        app-net
```

这样就能确认 `app-net` 实际上对应 `br-4d44c079d811`。

### 2. 两个网桥的容器是否默认互通？

答案：**默认不互通**。

网络结构：

```text
docker0 (172.17.0.1/16)
    |
    +--- Container-A (172.17.0.2)

br-4d44c079d811 (172.18.0.1/16)
    |
    +--- Container-B (172.18.0.2)
```

Docker 默认会隔离不同 Bridge 网络。因此在 Container-A 中执行 `ping 172.18.0.2`，大概率失败。

#### 为什么隔离？

Docker 默认添加防火墙规则（`DOCKER-ISOLATION` / DOCKER 链 DROP 规则），目的是避免容器间意外访问：Network A 不能直接访问 Network B。

### 3. 如何实现两个网络互通？

有多种方法。

#### 方法一：将容器加入同一个网络（推荐）

```bash
docker network create app-net
docker run --network app-net nginx
docker run --network app-net redis
```

结果：容器分别获得 `172.18.0.2`、`172.18.0.3`，直接互通。

#### 方法二：一个容器加入多个网络

```bash
docker network connect bridge web-container
```

查看：

```bash
docker inspect web-container
```

结果：

```text
bridge   172.17.0.10
app-net  172.18.0.10
```

拓扑变成（双网卡容器）：

```text
       docker0
    172.17.0.0/16
          |
   +--------------+
   | Container A  |
   | 172.17.0.10  |
   | 172.18.0.10  |
   +--------------+
          |
    172.18.0.0/16
       app-net
          |
   +--------------+
   | Container B  |
   | 172.18.0.20  |
   +--------------+
```

此时 A ↔ B 可以通信，就像 Windows 服务器有两块网卡。

#### 方法三：宿主机路由（不推荐）

```bash
sysctl -w net.ipv4.ip_forward=1
```

再放开 iptables，实现 `172.17.0.0/16 ↔ 172.18.0.0/16` 路由互通，但管理复杂。

#### 方法四：Docker Compose 网络设计（企业常用）

```yaml
services:
  nginx:
    networks:
      - frontend
      - backend

  app:
    networks:
      - backend

networks:
  frontend:
  backend:
```

拓扑：

```text
frontend
    |
  nginx
    |
backend
  /    \
app   mysql
```

这样 `nginx` 作为中间层，`app`、`mysql` 不直接暴露在 frontend。

### 你的环境实际情况

根据之前的 NAT 规则推测：

```text
docker0
└─ PostgreSQL (172.17.0.2:5432)

app-net (br-4d44c079d811)
└─ Web Application (172.18.0.2:8000)
```

如果 Web 容器需要连接 PostgreSQL，最好的方式是把 PostgreSQL 也加入 `app-net`：

```bash
docker network connect app-net postgres
```

这样无需开放端口，也无需配置额外路由。

---

### 附：实际环境排查记录

#### 容器信息（docker inspect 92b35d1d75b3 摘要）

```json
{
  "Name": "/chery-tools",
  "Path": "uvicorn",
  "Args": ["app.main:app", "--host", "0.0.0.0", "--port", "8000",
           "--ssl-keyfile", "/certs/key.pem", "--ssl-certfile", "/certs/cert.pem"],
  "HostConfig": {
    "NetworkMode": "chery-tools_default",
    "PortBindings": { "8000/tcp": [ { "HostPort": "443" } ] }
  },
  "NetworkSettings": {
    "Networks": {
      "chery-tools_default": {
        "Gateway": "172.18.0.1",
        "IPAddress": "172.18.0.2",
        "MacAddress": "e6:5c:66:85:49:61",
        "IPPrefixLen": 16,
        "DNSNames": ["chery-tools", "invoice-api", "92b35d1d75b3"]
      }
    }
  }
}
```

（完整 inspect 输出还包含 Binds 挂载 `certs:/certs:ro`、卷 `chery-tools_dedup-data:/data:rw`、restart policy `unless-stopped` 及大量业务环境变量。）

#### 网络列表

```bash
docker network ls
```

```text
NETWORK ID     NAME                  DRIVER    SCOPE
4c5132dbffdd   bridge                bridge    local
4d44c079d811   chery-tools_default   bridge    local
ee53c0d10f11   host                  host      local
1ba1dd80738a   none                  null      local
```

#### 宿主机接口（ip addr 摘要）

```text
2: eth0: inet 10.10.10.22/24        # 宿主机物理网卡
3: br-4d44c079d811: inet 172.18.0.1/16   # 自定义网络网桥
4: docker0: inet 172.17.0.1/16           # 默认网桥
201: veth34a5eca@if2: master docker0          # PostgreSQL 容器的 veth
209: veth61d83e0@if2: master br-4d44c079d811  # chery-tools 容器的 veth
```

#### 容器使用的地址

```text
docker inspect postgres      →  172.17.0.2  (PostgreSQL)
docker inspect 92b35d1d75b3  →  172.18.0.2  (chery-tools)
```

#### 网络详情（docker network inspect chery-tools_default 摘要）

```json
{
  "Name": "chery-tools_default",
  "Driver": "bridge",
  "IPAM": {
    "Config": [ { "Subnet": "172.18.0.0/16", "Gateway": "172.18.0.1" } ]
  },
  "Containers": {
    "c4dec471136b...": { "Name": "chery-tools", "IPv4Address": "172.18.0.2/16" }
  },
  "Status": { "IPAM": { "Subnets": { "172.18.0.0/16": { "IPsInUse": 4, "DynamicIPsAvailable": 65532 } } } }
}
```

#### NAT 规则（iptables -t nat -L -n -v 摘要）

```text
Chain POSTROUTING:
  MASQUERADE  --  *  !docker0          172.17.0.0/16  0.0.0.0/0
  MASQUERADE  --  *  !br-4d44c079d811  172.18.0.0/16  0.0.0.0/0

Chain DOCKER:
  DNAT  tcp  !docker0          0.0.0.0/0  127.0.0.1     tcp dpt:55432  to:172.17.0.2:5432
  DNAT  tcp  !br-4d44c079d811  0.0.0.0/0  0.0.0.0/0     tcp dpt:443    to:172.18.0.2:8000
```

#### 自定义网桥时指定网段与网关

创建 Docker 自定义网桥时，可以通过 `--subnet` 和 `--gateway` 参数手动指定网段和网关：

```bash
docker network inspect app-net

docker network create \
  --driver bridge \
  --subnet <网段/掩码> \
  --gateway <网关IP> \
  app-net

# 示例
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  app-net
```

在 docker-compose.yml 中配置：

```yaml
version: '3.8'
services:
  web:
    image: nginx
    networks:
      - app-net

networks:
  app-net:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.100.0/24
          gateway: 192.168.100.1
```

#### 为什么容器地址从 172.17.0.2 开始？如何查看网络下的容器

`.1` 被网桥自身（网关）占用，容器 IP 从 `.2` 开始分配。

```bash
docker network ls
docker network inspect <network-name>
docker network inspect <network-name> --format '{{len .Containers}}'
docker network inspect <network-name> --format '{{range $id,$c := .Containers}}{{$c.Name}} {{$c.IPv4Address}}{{println}}{{end}}'
```

#### 查看命名空间

```bash
lsns -t net
```

```text
NS TYPE NPROCS     PID USER    NETNSID NSFS                           COMMAND
4026531840 net     193       1 root unassigned /run/docker/netns/default /usr/lib/systemd/systemd --system --deserialize=119
4026532171 net       1 1264590 root          0 /run/docker/netns/14fcccec6b3a /usr/local/bin/python3.13 /usr/local/bin/uvicorn app.main:app --host 0.0.0.0 --
4026532241 net       6 1243997 999           1 /run/docker/netns/207b600e3a93 postgres
```

#### 查看容器对应的 Namespace 并进入

```bash
docker inspect -f '{{.State.Pid}}' tmp-pg
# 1243997

ls -l /proc/1243997/ns/net
# lrwxrwxrwx ... /proc/1243997/ns/net -> 'net:[4026532241]'

# 进入容器网络命名空间查看
nsenter -t 1243997 -n ip addr
```

```text
1: lo: inet 127.0.0.1/8
2: eth0@if201: inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
```

#### Docker 自动生成的隔离规则（iptables -L DOCKER -n -v）

```text
Chain DOCKER (2 references)
pkts bytes target  prot opt in                 out                source       destination
14   840   ACCEPT  6   --  !br-4d44c079d811    br-4d44c079d811    0.0.0.0/0    172.18.0.2   tcp dpt:8000
0    0     ACCEPT  6   --  !docker0            docker0            0.0.0.0/0    172.17.0.2   tcp dpt:5432
0    0     DROP    0   --  !br-4d44c079d811    br-4d44c079d811    0.0.0.0/0    0.0.0.0/0
0    0     DROP    0   --  !docker0            docker0            0.0.0.0/0    0.0.0.0/0
```

其中最后两条 DROP 规则就是不同 bridge 网络之间默认隔离的来源。FORWARD 链的完整链路为 `FORWARD → DOCKER-USER → DOCKER-FORWARD → DOCKER-CT / DOCKER-INTERNAL / DOCKER-BRIDGE → DOCKER`，并与 UFW 规则共存（filter 表 INPUT 默认 DROP，由 ufw-* 链放行 SSH/443/8080 等端口）。

#### iptables-save 关键规则摘录

```text
*raw
-A PREROUTING -d 172.17.0.2/32 ! -i docker0 -j DROP
-A PREROUTING -d 127.0.0.1/32 ! -i lo -p tcp -m tcp --dport 55432 -j DROP
-A PREROUTING -d 172.18.0.2/32 ! -i br-4d44c079d811 -j DROP

*nat
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
-A POSTROUTING -s 172.18.0.0/16 ! -o br-4d44c079d811 -j MASQUERADE
-A DOCKER -d 127.0.0.1/32 ! -i docker0 -p tcp -m tcp --dport 55432 -j DNAT --to-destination 172.17.0.2:5432
-A DOCKER ! -i br-4d44c079d811 -p tcp -m tcp --dport 443 -j DNAT --to-destination 172.18.0.2:8000

*filter
-A DOCKER -d 172.18.0.2/32 ! -i br-4d44c079d811 -o br-4d44c079d811 -p tcp -m tcp --dport 8000 -j ACCEPT
-A DOCKER -d 172.17.0.2/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 5432 -j ACCEPT
-A DOCKER ! -i br-4d44c079d811 -o br-4d44c079d811 -j DROP
-A DOCKER ! -i docker0 -o docker0 -j DROP
```

#### 其他网络模式说明

- 创建自定义网桥、创建容器使用指定网桥后，虚机会自动创建 NAT 规则。
- 若希望容器使用指定地址段（而非默认网桥的地址段），或容器 IP 直接出现在虚机的局域网中，可使用 **macvlan**。
- 不使用 NAT、容器直接获取与主机一样的网络配置的方式是 **MACVLAN** 或 **IPVLAN**。

#### 网桥与网段对照表

| 序号 | 网桥 | 网段 |
|------|------|------|
| 1 | docker0 | 172.17.0.1/16 |
| 2 | br-4d44c079d811 | 172.18.0.1/16 |

---

> 注意：原文档的 `docker inspect` 输出中包含大量真实的数据库连接串、SFTP 凭据、API Key 等敏感环境变量，本整理稿已略去，仅保留网络相关的关键字段。
