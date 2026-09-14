---
title: 银河麒麟 V10 ARM64 离线部署 Redis｜信创环境踩坑实录
date: 2026-09-14 18:30:00
tags: 
    - 信创环境
    - 麒麟V10系统
    - redis
categories: 
    - 技术笔记
---

生产环境实战复盘：在银河麒麟 V10 ARM64 信创内网环境，从一次"服务器失联事故"到最终稳定落地 Redis 的全过程，包含完整部署步骤与终极避坑规范。
<!--more-->

## 一、环境介绍

| 项目 | 说明 |
|------|------|
| 操作系统 | 银河麒麟 V10 ARM64 |
| 网络环境 | 内网隔离、无外网（离线） |
| 服务器配置 | 8 vCPU / 32G 内存 |
| 本机内网 IP | `172.20.145.178` |
| Redis 部署方式 | **宿主机 YUM 安装（不使用 Docker）** |
| 业务部署方式 | Docker 容器，`network_mode: host` 主机网络模式 |

## 二、前期重大踩坑记录

### 坑 1：Docker 镜像架构不匹配，整机失联

最初尝试使用 Docker 部署 Redis，直接拉取了 `amd64` 架构的镜像，与 ARM64 服务器架构不匹配。启动后**服务器 CPU 满载、SSH 连接直接卡死、通道断开、整机失联**，差点造成生产事故。

**结论：信创 ARM 离线环境，基础中间件优先宿主机原生安装，规避镜像架构不兼容风险。**

### 坑 2：Bridge 网桥模式访问宿主机被拦截

Docker 默认 bridge 网桥模式下，业务容器通过内网 IP `172.20.145.178` 访问宿主机 Redis 时，麒麟系统 SELinux / firewalld 会拦截物理网卡流量，**随机出现 `Connection refused`（连接拒绝）**，服务时通时断。

**解决方案：业务容器开启 `network_mode: host`，访问本机 Redis 使用回环地址 `127.0.0.1`，绕开物理网卡拦截。**

## 三、Redis 完整部署步骤

### 3.1 查看 YUM 源 ARM 版本 Redis 包

```bash
yum list redis
# 返回结果示例
# redis.aarch64  7.2.14-1.p01.ky10
```

### 3.2 执行安装

```bash
yum install -y redis
```

⚠️ **麒麟 V10 特别提醒**：YUM 安装的 Redis，配置文件路径是 `/etc/redis.conf`，**`/etc` 下没有 `redis` 文件夹，配置文件直接就在 `/etc` 目录下**。

### 3.3 修改核心配置文件 `/etc/redis.conf`

```ini
# 监听所有网卡，允许外部访问
bind 0.0.0.0

# 监听端口
port 6379

# 设置访问密码
requirepass 你的密码

# 内存控制，防止 OOM（32G 服务器推荐配置）
maxmemory 16g
maxmemory-policy allkeys-lru
```

⚠️ **重要**：修改 Redis 配置文件后，**必须重启 Redis 服务才能生效**，Redis 不支持此类参数热加载。

### 3.4 启停与开机自启

```bash
# 设置开机自启
systemctl enable redis

# 启动 Redis
systemctl start redis

# 修改配置后重启
systemctl restart redis
```

### 3.5 验证监听状态

```bash
ss -tlnp | grep 6379
```

确认 Redis 监听 `0.0.0.0:6379` 即部署成功。

## 四、业务项目 .env 配置（SpringBoot）

业务容器是 host 网络模式，访问本机中间件统一写 `127.0.0.1`，**禁止写内网 IP**。

.env 特殊字符：含 \# $ \& 密码必须双引号包裹

```env
SPRING_REDIS_HOST=127.0.0.1
SPRING_REDIS_PORT=6379
SPRING_REDIS_DATABASE=8
SPRING_REDIS_PASSWORD=你的密码
```

## 五、常见问题排查

| 问题 | 排查方案 |
|------|----------|
| 业务连接 Redis 报 Connection refused | ① 确认 Redis 监听 `0.0.0.0`；② 业务配置不要写内网 IP，改用 `127.0.0.1` |
| 修改 redis.conf 不生效 | 内存、密码、bind 等参数**必须重启 Redis**，无法热更新 |
| 服务器 CPU 打满、卡死 | 检查是否误用 amd64 镜像/二进制，改用 aarch64 原生安装 |

## 六、总结

1. 信创麒麟 ARM64 离线环境，Redis 优先宿主机 YUM 原生安装；
2. 业务容器统一 host 网络模式，本机中间件访问走 `127.0.0.1` 回环网卡；
3. 修改 Redis 关键配置必须重启服务；
4. 避免在 ARM 服务器使用 amd64 镜像，防止整机失联事故。

#（注：内容由AI生成）
