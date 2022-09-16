---
title: "Kubernetes (Microk8s) 快速搭建指南"
date: 2022-08-29
last_modified_at: 2022-09-16
categories:
  - blog
tags:
  - Kubernetes
  - SJTU SE2320 互联网产品设计与开发
---
轻松搞定 Ubuntu 系统 Kubernetes 环境配置

## 安装 **Microk8s**

### TL; DR

1. 准备一个可以使用 clash 规则的科学上网服务
2. 执行 `curl -sL https://raw.githubusercontent.com/f1yby/se2320-miscellaneous/master/k8s/setup.sh > setup.sh`，或是将[setup.sh](https://github.com/f1yby/se2320-miscellaneous/blob/master/k8s/setup.sh)手动保存到本地
3. 执行 `sudo sh setup.sh ${clash-version} ${clash-config}`

### 网络环境

由于众所周知的原因，国内并不能直接访问 Kubernetes 官方镜像源，因此配置科学上网是必须的工作，参考[官方指南](https://microk8s.io/docs/install-proxy)

如果你是用的是 clash ，以下代码会为你配置`/etc/environment`

```bash
proxy_server=http://127.0.0.1:7890

echo "HTTPS_PROXY=$proxy_server
HTTP_PROXY=$proxy_server
NO_PROXY=10.0.0.0/8,192.168.0.0/16,127.0.0.1,172.16.0.0/16
https_proxy=$proxy_server
http_proxy=$proxy_server
no_proxy=10.0.0.0/8,192.168.0.0/16,127.0.0.1,172.16.0.0/16
" | sudo tee -a /etc/environment
```

### 安装 [snap](https://snapcraft.io/docs/installing-snapd)

### 安装 microk8s

```bash
sudo snap install microk8s --channel=1.24/stable --classic
```

### microk8s 初始化

在这一步之前请确保**网络环境**已经正确配置。

```bash
microk8s.start && microk8s.status -w
```

**Done!**

