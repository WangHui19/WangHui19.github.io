---
title: 银河麒麟 V10 ARM64 离线部署 Elasticsearch 单节点（single-node）
date: 2026-09-14 18:30:00
tags: 
    - 信创环境
    - 麒麟V10系统
    - Elasticsearch
categories: 
    - 技术笔记
---

生产环境实战复盘：在银河麒麟 V10 ARM64 信创内网环境，采用 **elasticsearch\-7\.17\.8\-aarch64\.rpm 官方离线包**部署 Elasticsearch 单节点，开启 Xpack 账号密码认证，并打通业务容器访问的全过程。
<!--more-->

## 一、环境介绍

| 项目 | 说明 |
|------|------|
| 操作系统 | 银河麒麟 V10 ARM64 |
| 网络环境 | 内网隔离、无外网（离线） |
| 服务器配置 | 8 vCPU / 32G 内存 |
| 本机内网 IP | `172.20.145.178` |
| ES 部署方式 | **ARM 二进制包宿主机部署（不使用 Docker）** |
| 节点模式 | 单节点 `single-node` |
| 安全认证 | 开启 Xpack 账号密码认证 |
| 业务部署方式 | Docker 容器，`network_mode: host` 主机网络模式 |

## 二、踩坑记录

### 坑 1：架构不匹配

ARM 服务器不要使用 amd64 的 Docker 镜像，极易引发系统异常。选择 ARM 原生二进制包离线部署最稳妥。

### 坑 2：单节点误配置集群参数（最常见误区）

很多开发者会给单节点 ES 错误配置集群节点参数 `discovery.seed_hosts`，并且混淆两个端口：

- **9200**：HTTP 业务访问端口
- **9300**：集群节点间 TCP 通信端口

**重点：`single-node` 模式下，不需要配置任何集群节点列表（`discovery.seed_hosts` / `elasticsearch_nodes`），配置反而会触发启动校验报错。**

### 坑 3：网络访问被拦截

Bridge 网桥模式容器访问宿主机 ES 会被麒麟防火墙拦截，改用 host 容器 + `127.0.0.1` 访问。

## 三、完整部署流程

### 3.1 RPM 离线安装

上传包：`elasticsearch-7.17.8-aarch64.rpm`

```Plain Text
rpm -ivh elasticsearch-7.17.8-aarch64.rpm
```

安装后自动：创建用户、创建服务、生成配置目录、配置系统权限。

### 3.2 修改主配置文件（生产核心）

路径：`/etc/elasticsearch/elasticsearch.yml`

```Plain Text
# 全局监听所有网卡
network.host: 0.0.0.0
# 业务HTTP端口
http.port: 9200
# 单节点模式（最关键配置）
discovery.type: single-node
# 开启密码登录认证
xpack.security.enabled: true

# ===== 单节点【必须删除】以下所有集群配置 =====
# discovery.seed_hosts: []
# cluster.name:
# node.name:
```

### 3.3 JVM 内存调优

路径：`/etc/elasticsearch/jvm.options`

```Plain Text
-Xms4g
-Xmx4g
```

### 3.4 Systemd 服务启停管理

```Plain Text
systemctl enable elasticsearch
systemctl start elasticsearch
systemctl status elasticsearch
```

### 3.5 初始化 ES 账号密码

```Plain Text
/usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
```

多个账号，密码尽量统一，避免遗忘混乱

### 3.6 端口监听 \+ 业务连通验证

```Plain Text
# 检查端口监听
ss -tlnp | grep 9200

# 带密码访问验证
curl 127.0.0.1:9200 -u elastic:你的密码
```

## 四、SpringBoot 业务 .env 配置

业务容器是 host 网络模式，访问本机 ES 写 `127.0.0.1:9200`；已开启认证，账号密码字段必填。

.env 特殊字符：含 \# $ \& 密码必须双引号包裹

```env
ELASTICSEARCH_HOST=127.0.0.1
ELASTICSEARCH_PORT=9200
ELASTICSEARCH_USERNAME=elastic
ELASTICSEARCH_PASSWORD=你的密码
```

⚠️ 单节点**不需要** `ELASTICSEARCH_NODES` / `discovery.seed_hosts` 这类集群节点参数。

## 五、常见问题排查

| 问题 | 排查方案 |
|------|----------|
| single-node 启动报错 | 删掉 `discovery.seed_hosts` 集群配置，单节点不需要节点发现 |
| 业务连接 ES 超时/拒绝 | 业务 host 容器使用 `127.0.0.1:9200` 访问宿主机 ES |
| 9200/9300 端口混淆 | 9200 是 HTTP 业务端口，9300 是集群通信端口；单节点不需要 9300 |

## 六、总结

1. ES 单节点模式，彻底删除集群节点配置，规避启动报错；
2. 业务 host 容器访问本机 ES，统一走 `127.0.0.1:9200`；
3. 开启 Xpack 认证后，业务配置必须带账号密码；
4. 分清 9200（业务）与 9300（集群）两个端口。

#（注：内容由AI生成）
