---
title: docker-basic
tags:
  - docker
categories:
  - 容器技术
  - docker
date: 2022-12-02 10:34:20
---

# docker安装

## ubuntu

```shell
# 卸载已安装的docker
sudo apt-get remove docker docker-engine docker.io containerd runc
# 更新apt软件包索引并安装软件包（允许apt通过HTTPS使用仓库）：
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
#  安装
sudo apt-get -y update
sudo apt-get -y install docker-ce

```

2. 设置镜像加速器

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://p6qarvcy.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

