---
title: Unbutu 使用
tags:
  - linux
  - Unbutu
categories:
  - 操作系统
  - linux
  - Unbutu
date: 2022-11-27 19:21:30
---



# deb包

## 查看默认安装位置

查看deb包的安装位置：

```plain
sudo dpkg-deb -c jdk-19_linux-x64_bin.deb 
```

## 安装deb包

```powershell
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_9.2.4_amd64.deb
sudo dpkg -i grafana-enterprise_9.2.4_amd64.deb
```
