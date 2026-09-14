---
title: 银河麒麟 V10 ARM64 离线部署 MinIO 对象存储实战
date: 2026-09-14 18:30:00
tags: 
    - 信创环境
    - 麒麟V10系统
    - MinIO
categories: 
    - 技术笔记
---

生产环境实战复盘：在银河麒麟 V10 ARM64 信创内网环境，使用 **minio-20250422221226.0.0-1.aarch64.rpm，aarch64 官方二进制文件**部署 MinIO 对象存储，并打通业务容器访问的全过程。
<!--more-->

## 一、环境介绍

| 项目 | 说明 |
|------|------|
| 操作系统 | 银河麒麟 V10 ARM64 |
| 网络环境 | 内网隔离、无外网（离线） |
| 服务器配置 | 8 vCPU / 32G 内存 |
| 本机内网 IP | `172.20.145.178` |
| MinIO 部署方式 | **ARM 二进制宿主机部署（不使用 Docker）** |
| 部署形态 | 单节点 |
| 业务部署方式 | Docker 容器，`network_mode: host` 主机网络模式 |

## 二、踩坑记录

### 坑 1：架构问题

ARM 服务器不要使用 amd64 镜像，避免架构不兼容导致系统异常。

### 坑 2：网络访问被拦截

Bridge 网桥模式容器访问宿主机 MinIO，会被麒麟系统防火墙拦截。

## 三、部署步骤

### 3.1 RPM 安装

```Plain Text
rpm -ivh minio-20250422221226.0.0-1.aarch64.rpm
```

### 3.2 创建数据目录（root 权限）

```Plain Text
mkdir -p /data/minio
chown -R root:root /data/minio
```

### 3.3 配置文件 /etc/default/minio

```Plain Text
MINIO_VOLUMES="/data/minio"
MINIO_OPTS="--address:9000"
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=Admin@123456
```

### 3.4 修改 systemd 为 root 启动（关键步骤）

编辑服务文件：

```Plain Text
vi /usr/lib/systemd/system/minio.service
```

注释默认 minio 用户，改为 root：

```Plain Text
#User=minio
#Group=minio
User=root
Group=root
```

### 3.5 重载服务、开机自启

```Plain Text
systemctl daemon-reload
systemctl enable minio
systemctl start minio
```

### 3.6 端口验证

```Plain Text
ss -tlnp | grep 9000
```

## 四、SpringBoot 业务 .env 配置参考

业务容器是 host 网络模式，访问本机 MinIO 写 `127.0.0.1:9000`。

.env 特殊字符：含 \# $ \& 密码必须双引号包裹

```env
MINIO_HOST=127.0.0.1
MINIO_PORT=9000
MINIO_ACCESS_KEY=admin
MINIO_SECRET_KEY=Admin@123456
```

## 五、常见问题排查

| 问题 | 排查方案 |
|------|----------|
| 业务无法连接 MinIO | 确认 MinIO 监听 `0.0.0.0:9000`；host 模式业务使用 `127.0.0.1` 访问 |
| 离线环境无法下载 minio | 提前在外部 ARM 环境下载 aarch64 rpm包，上传到内网服务器 |
| AccessKey/SecretKey 不生效 | 确认与启动时 `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` 一致 |

## 六、总结

1. MinIO 使用 aarch64 rpm宿主机部署，避免架构不兼容；
2. 业务 host 容器访问本机 MinIO，统一走 `127.0.0.1:9000`；
3. 提前准备离线rpm包，解决无外网环境的安装问题。

#（注：内容由AI生成）
