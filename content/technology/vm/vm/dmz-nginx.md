# DMZ 反向代理（dmz-nginx）

## 架构图

```mermaid
flowchart TD
    U["互联网用户 / 客户端"] -->|HTTPS 443| FG["FortiGate 防火墙<br/>外网 IP: 203.0.113.10<br/>只做 L3/L4 端口转发/DNAT"]
    FG -->|HTTPS 443 (VIP → 10.0.1.100)| NGX["DMZ: Nginx 反向代理<br/>10.0.1.100<br/>持有 SSL 证书，SSL 卸载"]
    NGX -->|HTTP 80 (X-Forwarded-*)| BE["内网后端业务服务器<br/>192.168.10.50:8080<br/>无需配置 SSL 证书"]
```

## 摘要

- FortiGate 工作在网络层（L3/L4），负责 IP/端口级别的访问控制；反向代理工作在应用层（L7），深入理解 HTTP/HTTPS 报文，两者是纵深防御的互补关系而非替代关系。
- 反向代理的核心价值：隐蔽内部真实网络架构、统一 Web 入口、收敛暴露面（无需映射 10 个外网 IP/端口）。
- SSL/TLS 证书统一管理与卸载（SSL Termination）：证书只在 Nginx 上配置一次，后端服务器可使用纯 HTTP，显著降低后端 CPU 压力并简化证书续期运维。
- 提供完整的 Nginx 配置实例（HTTP 强制 301 跳转 HTTPS、TLSv1.2/1.3、proxy_pass 转发、X-Forwarded-* 透传真实客户端 IP）与 FortiGate VIP/防火墙策略配置。
- 提供四步验证方案：curl 证书握手验证、301 重定向验证、后端 tcpdump 抓包验证 SSL 已卸载、后端日志验证真实 IP 传递。
- 证书更新零中断：只需更新 Nginx 的 certs 目录并 `nginx -s reload`，无需重启后端业务或变动 FortiGate。

## 技术要点

1. FortiGate 是"保安门卫"（查验身份、放行端口），反向代理是"前台接待员"（引导访客到正确房间并精细管理行为）。
2. FortiGate 配置虚拟 IP（VIP）：External IP 203.0.113.10 → Mapped IP 10.0.1.100，TCP 443→443 端口转发。
3. FortiGate 防火墙策略：wan1 → dmz，Destination 为 VIP-HTTPS-To-Nginx，Service HTTPS，Action ACCEPT，NAT 关闭以保留客户端真实 IP。
4. Nginx 80 端口 server 块使用 `return 301 https://$host$request_uri;` 强制跳转 HTTPS。
5. Nginx 443 server 开启 `listen 443 ssl http2`，配置 `ssl_protocols TLSv1.2 TLSv1.3`、`ssl_ciphers HIGH:!aNULL:!MD5`、`ssl_session_cache shared:SSL:10m`。
6. SSL 卸载核心：`proxy_pass http://192.168.10.50:8080;` 后端无需证书，同时通过 `proxy_set_header X-Real-IP / X-Forwarded-For / X-Forwarded-Proto https` 传递真实客户端信息。
7. 验证 SSL 卸载：在后端 `tcpdump -i any port 8080 -A` 抓包，应看到明文 HTTP 请求头及 `X-Forwarded-Proto: https` 报头。
8. 运维收益：证书续期零中断（仅 reload Nginx）；CPU 密集的 SSL 加解密全部由 Nginx 承载，内网业务服务器 CPU 占用显著降低。

## 原文内容

已经使用了fortigate搭建了DMZ，为什么还需要搭建反向代理？

FortiGate 防火墙和反向代理（Reverse Proxy）解决的是不同层面（网络层 vs. 应用层）的安全与管理问题。

简单来说：FortiGate 是"保安门卫"，负责查验身份和放行车辆；而反向代理是"前台/接待员"，负责把访客引导到正确的房间，并对其具体行为进行精细管理。

以下是即使有了 FortiGate DMZ，依然建议/需要部署反向代理的几个核心原因：

### 1. 业务防护维度的差异（网络层 vs 应用层）

- **FortiGate (防火墙 / DMZ)**：主要工作在网络层（L3/L4）和传输层。它负责访问控制（IP、端口、协议），比如只允许公网的 443 端口流量进入 DMZ。虽然 FortiGate 带有 WAF/IPS 模块，但它的核心强项依然是流量筛选和网络隔离。
- **反向代理**：工作在应用层（L7）。它可以深入理解 HTTP/HTTPS 报文的具体内容（如 Header、Cookie、URL 路径、请求参数），从而执行更精细的逻辑过滤和路由。

### 2. 核心功能补充

即使 FortiGate 隔离了 DMZ，反向代理还能提供以下防火墙无法替代或做不好功能：

**① 隐蔽内部真实网络架构与收敛暴露面**

- 不暴露多台服务器：如果 DMZ 内有 10 个不同的 Web 应用（比如 OA、CRM、官网），不需要在 FortiGate 上映射 10 个外网 IP 或开放 10 个不同端口。
- 统一入口：只需将反向代理暴露给 FortiGate，所有外网请求都发给反向代理，再由它根据域名或路径转发给内部真正的服务器。外部完全看不到后端具体有哪些服务器及其真实 IP。

**② SSL/TLS 证书统一管理与卸载 (SSL Termination)**

如果没有反向代理，你需要在后端的每一台 Web 服务器上配置和更新 SSL 证书。

部署反向代理后，所有证书只需在反向代理上一键配置。解密工作在代理层完成，代理与后端服务器之间可以使用 HTTP 或内部轻量加密，大幅减轻后端服务器的计算压力，同时也极大地简化了证书续期与运维。

**③ 灵活的负载均衡与平滑发布**

反向代理（如 Nginx、HAProxy、Traefik）具备极强的应用层负载均衡能力（如轮询、IP Hash、权重分发）。

支持无缝轮换（Rolling Update）与灰度发布：在后端服务器升级或维护时，只需在反向代理中摘除节点，前端用户完全无感知。

**④ 高级路由与请求重写**

- 可以轻松实现基于 URL 路径的路由（例如 /api 转发给 API 集群，/static 转发给静态资源服务器）。
- 轻松处理请求头修改、跨域（CORS）配置、URL 重写、重定向（HTTP 强制转 HTTPS）等纯业务逻辑需求。

### 3. 典型的联合架构（最佳实践）

在企业级安全架构中，两者通常是协同工作而非替代关系：

```text
[ 互联网 Users ]
│
▼
[ FortiGate 防火墙 ]  <--- 1. 过滤非法 IP/端口、防止 DDoS、基础 IPS 扫描
│ (只放行 80/443 端口)
▼
[ DMZ 区: 反向代理 ] <--- 2. SSL 卸载、统一域名入口、应用路由、WAF防护
│
▼
[ DMZ/内网: 业务服务器集群 ] (OA、CRM、API 等)
```

### 总结

- 只有 FortiGate：你的应用服务器直接暴露在网络前沿（哪怕在 DMZ），需要自己处理 SSL 证书、暴露多个 IP/端口，且难以做灵活的 HTTP 路由和负载均衡。
- FortiGate + 反向代理：FortiGate 挡住网络层的恶意攻击和非法端口扫描，反向代理收敛 Web 入口并提供高效的应用层分发与管理，达成纵深防御（Defense in Depth）。

---

## SSL/TLS 证书统一管理与卸载 (SSL Termination) —— 部署实例和验证方案

在企业网络架构中，将 FortiGate 防火墙 与 反向代理（以 Nginx 为例） 结合，实现 SSL 卸载（SSL Termination）与证书统一管理 是非常标准的最佳实践。

### 架构拓扑与流量路径

```text
[ 客户端 (公网) ]
│  HTTPS (443)
▼
[ FortiGate 防火墙 ]  (只做 L3/L4 端口转发/DNAT，将 443 转发至 Nginx)
│  HTTPS (443)
▼
[ DMZ: Nginx 反向代理 ] (持有 SSL 证书，完成解密，解密为 HTTP)
│  HTTP (80)
▼
[ 内网/后端 业务服务器 ] (如 192.168.10.50:8080，无需配置 SSL 证书)
```

### 一、部署实例

#### 1. 环境参数准备

- 域名：app.example.com
- FortiGate 外网 IP：203.0.113.10
- Nginx（反向代理）IP：10.0.1.100（位于 DMZ 区）
- 后端业务服务器 IP：192.168.10.50:8080（位于内网 LAN）
- SSL 证书文件：
  - 证书公钥：/etc/nginx/certs/app.example.com.crt
  - 证书私钥：/etc/nginx/certs/app.example.com.key

#### 2. Nginx 反向代理配置（SSL 卸载核心）

在 Nginx 所在服务器上配置 app.example.com.conf：

```nginx
server {
    listen 80;
    server_name app.example.com;
    # 将所有 HTTP 请求强制重定向至 HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    # 1. 统一管理 SSL 证书
    ssl_certificate      /etc/nginx/certs/app.example.com.crt;
    ssl_certificate_key  /etc/nginx/certs/app.example.com.key;

    # 2. SSL 加密协议与套件优化
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # 3. 反向代理与 SSL 卸载配置（解密后发送 HTTP 给后端）
    location / {
        proxy_pass http://192.168.10.50:8080;    # 后端服务器无需配置证书

        # 传递客户端真实信息给后端应用
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;  # 告知后端实际访问使用的是 HTTPS

        # 超时设置
        proxy_connect_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

#### 3. FortiGate 防火墙配置（只做端口放行与映射）

**① 配置虚拟 IP（VIP / 端口映射）**

- Name: VIP-HTTPS-To-Nginx
- External IP: 203.0.113.10
- Mapped IP: 10.0.1.100
- Port Forwarding: 开启
- Protocol: TCP
- External Port: 443
- Map to Port: 443

**② 配置防火墙策略（Firewall Policy）**

- Incoming Interface: wan1 (外网口)
- Outgoing Interface: dmz (DMZ 口)
- Source: All
- Destination: VIP-HTTPS-To-Nginx
- Service: HTTPS
- Action: ACCEPT
- NAT: 关闭（保留客户端真实 IP）

### 二、验证与测试方案

配置完成后，分步骤执行以下验证，确认 SSL 卸载和代理路由正常。

**验证 1：基本联通性与 SSL 握手验证**

```bash
# 查看 SSL 握手过程与证书是否生效
curl -vI https://app.example.com
```

预期输出标志：

- 显示 SSL connection using TLSv1.3。
- 证书颁发者与域名正确无误。
- 响应状态码为 HTTP/2 200 或相应的正常状态。

**验证 2：验证 HTTP 强制跳转 HTTPS**

```bash
curl -I http://app.example.com
```

预期输出：

```text
HTTP/1.1 301 Moved Permanently
Location: https://app.example.com/
```

**验证 3：验证 SSL 确实在 Nginx 层被卸载（核心验证）**

要在后端确认 SSL 已解密，可在后端业务服务器（192.168.10.50）上抓包分析：

```bash
# 在后端服务器上抓取 8080 端口的流量
tcpdump -i any port 8080 -A
```

然后再在外部发起请求 `curl https://app.example.com`。

预期输出：

- 抓包结果中可以直接看到明文的 HTTP 请求头（如 GET / HTTP/1.1）以及包含 X-Forwarded-Proto: https 报头。
- 说明流量进入 Nginx 前是 HTTPS 加密流，通过 Nginx 后被彻底解密为 HTTP 明文发送给后端，验证 SSL 卸载成功。

**验证 4：客户端真实 IP 传递验证**

在后端应用的日志（如 Tomcat、Node.js、Nginx 后端日志）中查看日志格式。

预期输出：

- 日志记录的访问者 IP 为公网客户端的真实 IP，而不是 DMZ 区 Nginx（10.0.1.100）或 FortiGate 的 IP，证明 X-Forwarded-For 传递正常。

### 运维总结

通过这种方案：

- **证书更新零中断**：未来续期 SSL 证书时，仅需更新 Nginx 的 /etc/nginx/certs/ 目录并运行 `nginx -s reload`，无需重启任何后端业务服务，也不需要变动 FortiGate 防火墙。
- **性能优化**：CPU 密集的 SSL 加解密计算全部由 Nginx 承载，内网业务服务器 CPU 占用显著降低。
