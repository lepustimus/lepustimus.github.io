---
title: 内网渗透 | ATT&CK2打靶记录
published: 2025-12-15
description: ''
image: ''
tags: []
category: '内网渗透'
draft: false 
lang: ''
---

# 1、环境介绍
```
外网机：192.168.111.80 / 10.10.10.80
PC：10.10.10.201
DC：10.10.10.10
```
![alt text](image-251.png)

# 2、打靶过程：
## 1、拿下webshell
- 访问外网机：
![alt text](image-252.png)
- nmap探测目标端口：
![alt text](image-253.png)
- 发现存在7001端口，通过weblogic扫描此端口：
![alt text](image-254.png)
- 注入内存马：
![alt text](image-255.png)
- 蚁剑连接：
![alt text](image-256.png)
- 连接成功：
![alt text](image-257.png)

## 2、内网信息收集
- Ipconfig查看当前机器信息：
![alt text](image-258.png)
- 目标存在两个网段，蚂剑上传CS马，获取目标CS终端：
![alt text](image-259.png)
![alt text](image-260.png)
- 上传fscan进行扫描：
![alt text](image-261.png)
- 导出扫描结果并查看：
![alt text](image-262.png)
![alt text](image-263.png)
```
WEB：192.168.111.80 / 10.10.10.80
DC：10.10.10.10  administrator:1qaz@WSX
PC：10.10.10.201 administrator:1qaz@WSX
doamin：DE1AY
```

## 3、socks+proxifier全局代理
- CS搭建socks5：
![alt text](image-264.png)
- proxifier搭建全局代理：
![alt text](image-265.png)

## 4、内网横向
- 通过impacket工具箱中的psexec横向到PC：
![alt text](image-266.png)
- 通过impacket工具箱中的psexec横向到DC：
![alt text](image-267.png)
![alt text](image-268.png)