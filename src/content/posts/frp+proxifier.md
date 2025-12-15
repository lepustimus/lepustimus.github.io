---
title: frp+proxifier/nps代理搭建
published: 2025-12-15
description: ''
image: ''
tags: []
category: '内网渗透'
draft: false 
lang: ''
---

# 1、实验环境
- 介绍：
  - 攻击机与跳板机可互相通信，内网机与跳板机可互相通信，攻击机和内网机无法通信

![alt text](image-199.png)

# 2、隧道代理方法
## 1、frp(socks)+proxifier
  - frp原理：服务端运行，监听一个主端口，用户和客户端建立稳定连接，同时告诉服务端要监听的端口（通信端口）和需要转发的类型，服务端开启新的进程监听客户端指定的端口，外网用户通过服务端连接到客户端的指定端口，服务端和客户端的连接将数据转发到客户端，客户端再将数据转发到本地服务，实现将内网暴露到外网的目的
  - frps.toml：
  ```
  bindPort = 7000 # 服务端与客户端的稳定连接端口
  ```
  - frpc.toml：
  ```
serverAddr = "127.0.0.1" # 代理服务器
serverPort = 7000 # 服务端与客户端的稳定连接端口

[[proxies]]
name = "test-tcp"
type = "tcp" # 代理类型
localIP = "127.0.0.1" # 本地服务的 IP 地址，即需要被代理的服务所在的IP
localPort = 22 # 本地服务监听的端口，这里是 22 端口
remotePort = 6000 # 服务端和客户端的通信端口
```
  - SOCKS代理和TCP代理区别：
    - SOCKS：SOCKS代理的"动态性"不绑定具体目标，SOCKS是一种通用代理协议，它的作用是为客户端提供一个"入口"，让客户端可以通过这个入口访问任意目标（只要跳板机能到达），而不是固定转发到某个特定机器。
    - TCP：TCP代理是"点对点"的固定转发TCP代理的核心作用是将特定端口的流量固定转发到目标机器的特定端口，它是一种"一对一"的映射关系。

  - 实验展示：
  - 通过CS上线跳板机：
  ![alt text](image-200.png)
  - 编辑frpc.toml文件：
  ```
  serverAddr = "59.110.53.241"
  serverPort = 7000

  [[proxies]]
  name = "socks5"
  type = "tcp"
  remotePort = 6028
  [proxies.plugin]
  type = "socks5"
  ```
  - 上传frpc.exe/frpc.toml：
  ![alt text](image-201.png)
  - 服务端开启frp服务端：
  ```
  ./frps -c frps.toml
  ```
  ![alt text](image-202.png)
  - cs开启frpc客户端：
  ![alt text](image-203.png)
  - 配置proxifier：
  ![alt text](image-204.png)
  - 访问内网机服务：
  ![alt text](image-205.png)

## 2、frp(tcp)
  - 改变frpc.toml文件内容如下：
  ```
  serverAddr = "59.110.53.241"
  serverPort = 7000

  [[proxies]]
  name = "test-tcp"
  type = "tcp"
  localIP = "192.168.1.132"
  localPort = 80
  remotePort = 6008
  ```
  - 先后分别运行frp服务端和客户端：
  ![alt text](image-206.png)
  ![alt text](image-207.png)
  - 浏览器访问http://59.110.53.241:6008/
  ![alt text](image-208.png)

## 3、nps(socks)+proxifier
  - 原理：
    - 服务端 (NPS) 与客户端 (NPC) 架构：NPS 由两部分组成：服务端 (nps) 和客户端 (npc)。服务端通常部署在一台具有公网 IP 地址的服务器上（例如云服务器 VPS），而客户端则部署在内网中需要将其服务暴露出去的机器上。
    - 建立连接与隧道：客户端 (npc) 主动向部署在公网的服务端 (nps) 发起连接请求，并在两者之间建立一个稳定、加密的通信隧道。这个隧道可以是基于 TCP 或 UDP 协议的。
    - 端口映射与流量转发：一旦隧道建立成功，用户可以在服务端配置端口映射规则。这些规则定义了公网服务器上的某个端口（外部访问端口）如何映射到内网客户端机器上的特定端口（内部服务端口）。当外部用户访问公网服务器的指定外部端口时，服务端 (nps) 会通过已建立的隧道，将接收到的网络请求流量转发给内网的客户端 (npc)。
    - 数据传输与响应：客户端 (npc) 接收到从服务端转发过来的请求后，会将请求交给内网中实际提供服务的应用程序。应用程序处理完请求后，将响应数据原路返回给客户端 (npc)，客户端再通过隧道将响应数据回传给服务端 (nps)，最终由服务端将响应数据发送给外部访问用户。
  - 常用命令：
    - sudo nps start：开启服务
    - sudo nps stop：停止服务
    - sudo nps restart：重启服务
    - sudo nps status：查看服务状态
    - sudo nps uninstall：卸载服务
  - 实验：
    - 服务端开始nps服务端：
    ![alt text](image-209.png)
    - 开启服务端web页面：
    ![alt text](image-210.png)
    - cs上线跳板机并上传npc.exe：
    ![alt text](image-211.png)
    - 服务端新建客户端：
    ![alt text](image-212.png)
    - 给此客户端添加socks代理：
    ![alt text](image-213.png)
    - 复制对应命令到跳板机：
    ![alt text](image-214.png)
    ![alt text](image-215.png)
    - 查看socks代理服务器对应端口：
    ![alt text](image-216.png)
    - 开启proxifier设置socks代理服务器及对应端口：
    ![alt text](image-217.png)
    - 建立对应代理规则进行对内网机web服务访问：
    ![alt text](image-218.png)

## 4、nps(tcp)
  - 实验：
    - 同理，在客户端中建立tcp隧道：
    ![alt text](image-219.png)
    ``注意：上述目标IP及端口写错了，应该是192.168.1.132:80``
    - 在客户端执行命令：
    ![alt text](image-220.png)
    - 访问59.110.53.241:8989
    ![alt text](image-221.png)