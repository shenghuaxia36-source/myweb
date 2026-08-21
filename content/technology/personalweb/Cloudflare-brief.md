# Cloudflare 业务简介

## 架构图

```mermaid
graph TD
    subgraph Cloudflare Connectivity Cloud
        subgraph 应用安全与加速
            CDN[CDN 全局加速]
            DDoS[DDoS 防御 L3/4 & L7]
            WAF[WAF / API安全 / 机器人管理]
            DNS[权威 DNS / 1.1.1.1]
        end
        subgraph 网络与 SASE
            ZTNA[零信任网络访问 ZTNA]
            SWG[安全 Web 网关 SWG]
            NaaS[网络即服务 Magic WAN]
            MAIL[企业邮件安全]
        end
        subgraph 开发者与边缘计算
            WORKERS[Cloudflare Workers - V8 Isolate]
            R2[R2 对象存储 - 免流量费]
            D1[D1 Serverless SQL 数据库]
            KV[Workers KV / Durable Objects]
        end
        subgraph 边缘 AI 平台
            WAI[Workers AI - 边缘 GPU]
            AIGW[AI Gateway - 监控/限流/缓存]
        end
    end
    EDGE[全球 330+ 城市边缘网络 Anycast] --> Cloudflare Connectivity Cloud
```

## 摘要

- Cloudflare 已从 CDN/DDoS 防护厂商演进为全球连接云（Connectivity Cloud）平台，覆盖安全、网络、计算与 AI 四大板块。
- 免费版极其慷慨，包含 DNS 解析、CDN 缓存、DDoS 防护、SSL 证书、WAF 规则、Page Rules 等网站功能。
- 开发者免费额度涵盖 Workers（每日 10 万次请求）、Pages（无限静态托管 + 每月 500 次构建）、R2（10GB 存储）、D1（每日 500 万次读取）、Workers AI。
- Zero Trust 免费支持最多 50 个用户，提供 ZTNA、DNS 过滤和基础防护规则。
- 免费版的主要限制在于机器人防护（仅基础筛查）、大文件/视频传输限制以及无 SLA 保障。

## 技术要点

1. **Anycast 边缘网络**：Cloudflare 在全球 330 多个城市部署边缘节点，利用 Anycast 架构实现就近接入和流量分发。
2. **DDoS 防御**：提供 L3/4 网络层和 L7 应用层的无上限容量抗 DDoS 攻击，免费版即可享受。
3. **Cloudflare Workers**：基于 V8 Isolate 技术的无服务器计算平台，极速冷启动，直接在边缘节点运行 JavaScript/Wasm，每日免费 10 万次请求。
4. **R2 对象存储**：兼容 S3 API，最大的特点是免收数据出流量费（No Egress Fees），每月免费 10GB 存储。
5. **D1 数据库**：Serverless 边缘关系型 SQL 数据库，每天免费 500 万次读取和 10 万次写入。
6. **Cloudflare Pages**：免费提供无限的静态网站托管以及每月 500 次 Build 编译次数，适合 Hugo/Hexo/前端项目。
7. **Zero Trust 免费版**：支持最多 50 个用户/终端，包含 ZTNA、DNS 域名过滤、基础防护规则，可替代传统 VPN。
8. **SSL/TLS 证书**：免费提供通配符 SSL 证书，支持自动续期，并支持 HTTP/2 和 HTTP/3 协议。
9. **免费 WAF 规则**：5 条自定义 WAF 规则和 3 条 Page Rules，满足基础防护和缓存/重定向策略需求。
10. **升级触发点**：复杂机器人防护需付费版机器学习与指纹验证；大文件传输需配合 R2 或 Stream；企业级需 SLA 保障和专属客户经理。

## 原文内容

Cloudflare 从最初的"CDN 与 DDoS 防护厂商"已经逐步演进为集应用程序安全、网络与 SASE、边缘计算及 AI 平台于一体的全球连接云（Connectivity Cloud）平台。

整体业务架构、关键产品及免费功能明细如下：

## 一、 总体业务架构与关键产品

Cloudflare 的业务围绕其分布于全球 330 多个城市的庞大边缘网络（Anycast 架构）展开，主要划分为以下四大核心板块：

```
┌─────────────────────────┐
│ Cloudflare Connectivity │
└────────────┬────────────┘
             │
  ┌──────────┬───────────────┴───────────────┬──────────────────┐
  ▼          ▼                               ▼                  ▼
应用安全与加速   网络与 SASE (Zero Trust)    开发者与边缘计算     边缘 AI 平台
(WAF/CDN/DDoS) (Cloudflare One)            (Workers/R2/D1)     (Workers AI/Gateway)
```

### 1. 应用程序交付与安全（App Performance & Security）

- **CDN 与全局加速**：利用全球节点对静态/动态内容进行智能缓存与路由优化（如 Argo Smart Routing）。
- **DDoS 防御**：支持 L3/4（网络层）及 L7（应用层）的无上限容量抗 DDoS 攻击。
- **应用安全套件**：包含 Web 应用防火墙（WAF）、API 安全保护、机器人管理（Bot Management）及 Rate Limiting（限速）。
- **DNS 服务**：提供极速且安全的权威 DNS 解析，以及公共 DNS 解析服务（如著名的 1.1.1.1）。

### 2. 网络与 Zero Trust / SASE（Cloudflare One）

- **零信任网络访问（ZTNA）**：替换传统 VPN，提供基于身份和设备安全状态的内网应用无缝访问。
- **安全 Web 网关（SWG）**：针对企业终端的 DNS 过滤、HTTP 流量拦截与恶意软件防护。
- **网络即服务（NaaS / Magic WAN）**：提供分支机构连接、SD-WAN 替换以及边缘防火墙（FWaaS）。
- **企业邮件安全**：防范钓鱼邮件、身份冒用及供应链攻击。

### 3. 开发者与边缘计算平台（Developer Platform）

- **Cloudflare Workers**：基于 V8 Isolate 技术的无服务器（Serverless）计算平台，极速冷启动，直接在边缘节点运行 JavaScript/Wasm。
- **边缘存储生态**：
  - **R2 Storage**：兼容 S3 的对象存储，免收数据出流量费（No Egress Fees）。
  - **D1**：Serverless 边缘关系型 SQL 数据库。
  - **Workers KV / Durable Objects**：用于键值对存储与高并发有状态状态机的存储组件。

### 4. 边缘 AI 平台（AI Infrastructure）

- **Workers AI**：直接在 Cloudflare 的边缘 GPU 节点上运行开源大语言模型（如 Llama、Mistral）与图像/声音模型。
- **AI Gateway**：提供针对主流 AI 服务的调用监控、速率限制、响应缓存与安全防泄漏控制。

## 二、 免费功能与业务情况（Free Plan）

Cloudflare 秉承"构建更好的互联网"理念，其免费版（Free Tier）非常慷慨，非常适合个人网站、独立开发者、开源项目及中小型企业初创期使用。

### 1. 免费的"网站与网络"功能

| 功能模块 | 免费版包含的权益 | 适用场景 / 限制 |
|---|---|---|
| DNS 解析 | 极速 DNS 服务、无限制域名托管 | 毫秒级解析响应速度 |
| CDN & 基础防护 | 无限 CDN 流量缓存、基础 Web 自动化防护 | 不支持大型视频/纯文件下载节点直连 |
| DDoS 防护 | 无容量限制的 L3/4 及 L7 DDoS 防御 | 自动抵御各种规模的流量攻击 |
| SSL/TLS 证书 | 免费提供通配符 SSL 证书，支持自动续期 | 强制 HTTPS 转换与 HTTP/2、HTTP/3 协议支持 |
| 基础 WAF 规则 | 5 条自定义 WAF 规则、基础 SQL 注入及恶意 IP 封禁 | 满足日常基础防护，高级规则需升级 |
| 页面规则 (Page Rules) | 提供 3 条免费 Page Rules 规则 | 可用于设置特定路径的重定向或缓存策略 |

### 2. 免费的"开发者与 Cloud 边缘"额度

Cloudflare 的免费版并非仅限 CDN，在 Serverless 及 AI 领域也提供了极其有竞争力的每日免费额度（Free Tier Allowances）：

- **Cloudflare Workers**：每日免费 100,000 次请求，支持 10ms CPU 运行时间。
- **Cloudflare Pages**：免费提供无限的静态网站托管以及每月 500 次 Build 编译次数。
- **R2 对象存储**：每月免费 10 GB 存储空间、1,000 万次 Class B 读操作及 100 万次 Class A 写操作（且无流量出站费）。
- **D1 数据库**：每天提供 500 万次读取和 10 万次写入的免费额度。
- **Workers AI**：每天提供一定额度的免费神经元（Neurons），可用于免费调用边缘 GPU 运行小型 AI 模型。

### 3. 免费的"零信任安全"（Zero Trust Free）

- **Cloudflare Zero Trust**：免费支持最多 50 个用户/终端。
- **主要权益**：包含身份认证网关 (ZTNA)、DNS 域名过滤、基础防护规则以及针对团队内网的无 VPN 安全穿透。

## 三、 免费版的主要限制与升级触发点

尽管免费功能十分全面，但在以下场景中需要考量升级到 Pro / Business / Enterprise 计划：

- **机器人防护（Bot Management）**：免费版仅提供基础的 IP 特征和 User-Agent 筛查；若遭遇复杂的黄牛抢购、数据刷虫，需使用付费版的机器学习与指纹验证机制。
- **大文件与视频传输**：免费 CDN 不允许作为单一的视频分发网络，频繁超量传输大文件可能会触发 TOS 限制，需配合 R2 或 Stream 产品使用。
- **支持与 SLA 保障**：免费版仅提供社区支持与工单排队，不提供服务等级协议（SLA）保障；企业级（Enterprise）则提供 15 分钟内的高优先响应和专属客户经理。
