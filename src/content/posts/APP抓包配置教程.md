---
title: APP抓包配置教程
published: 2025-11-05
description: '记录个人APP抓包的配置过程'
image: ''
tags: [漏洞挖掘, APP抓包, 安卓逆向]
category: '安卓逆向'
draft: false 
lang: ''
---

# 1、MuMu模拟器抓包配置

## 1、基础设置

![alt text](image-22.png)
![alt text](image-23.png)

## 2、配置MuMu模拟器、adb基本使用：
* 注意事项：在使用模拟器进行抓包时使用模拟器自带的adb
![alt text](image-24.png)

* 通过adb连接模拟器
![alt text](image-25.png)

* 申请root权限：
![alt text](image-26.png)

* 通过adb下载apk文件：
![alt text](image-27.png)

## 3、数据包抓取相关配置：

* 导出Burp证书：
![alt text](image-28.png)

* 通过openssl将cer格式证书转换成pem格式证书文件
    * openssl介绍：OpenSSL是一个功能丰富且开源的安全工具箱，她提供给的主要功能有：SSL\TLS协议实现、大量软算法（对称/非对称/摘要）、大数运算、非对称算法密钥生成、ASN.1编解码库、证书请求（PKCS10）编解码、数字证书编解码、OCSP协议、数字证书验证、PKCS7标准实现和PKCS12个人数字证书格式实现等功能
    * OpenSSL采用C语言作为开发语言，这使得她具有优秀的跨平台性能，OpenSSL夸平台，支持的系统：Linux\Windows\Mac\Unix等平台
    - 源码下载地址：[传送](https://slproweb.com/products/Win32OpenSSL.html)
    - OpenSSL工具箱主要包括下面三个组件：
        - openssl：多用途的命令行工具
        - libcrypto：加密算法库
        - libssl：加密模块应用库，实现了ssl及TLS协议
    * [openssl安装文章](https://blog.csdn.net/zyhse/article/details/108186278)
* openssl转换证书格式过程：
```
openssl x509 -inform der -in your_certificate.cer -out your_certificate.pem
openssl x509 -inform -subject_hash_old -in your_certificate.pem
对应参数解析：
x509：处理X.509证书的命令。
-inform der：输入的证书格式为DER。
-in：指定输入文件，默认是标准输入。
-out：指定输出文件，默认是标准输出。
-subject_hash_old：用md5方式显示项目的hash值
```
![alt text](image-29.png)

* 取到上述hash值并将Mumu.pem文件名修改为9a5ba575.0：
![alt text](image-30.png)

* 通过adb上传刚刚的证书到模拟器：
    * 注意事项：一定要将证书导入到模拟器存在系统证书目录下(/system/etc/security/cacerts)
    ![alt text](image-31.png)

## 4、设置代理

* 模拟器网络设置：
![alt text](image-32.png)

* 之后在Burp设置对应的IP及端口即可成功抓包

# 2、Pixel3真机抓包配置

## 1、安装scrcpy和adb

* scrcpy用于将真机页面投屏到电脑屏幕上：
![alt text](image-33.png)

* adb用于控制真机：
![alt text](image-34.png)

## 2、导出Burp证书

* Burp导出：
![alt text](image-35.png)

* 移动端需要cer后缀格式的证书，所以对导出的der后缀证书修改为cer后缀名即可：
![alt text](image-36.png)

## 3、将证书上传到移动端并通过adb查看对应目录上传结果

* 命令：
```bash
adb.exe push E:\Reverse\Certificate\mobile_burp.cer /sdcard/Dowdload
```
![alt text](image-37.png)
``注意：这里上传cer文件之后不知道为什么后缀变成ce了，把他重命名为cer后缀即可``

## 4、移动端安装CA证书：

* 打开手机设置-安全-加密与凭据-安装证书：
![alt text](image-38.png)

* 安装CA证书：
![alt text](image-39.png)

* 点击下载：
![alt text](image-40.png)

* 找到刚刚上传的cer后缀文件并安装即可

* 安装之后可以在信任与凭据-用户这里看到刚刚安装的mobile_burp.cer文件：
![alt text](image-41.png)

* 接下来需要将用户部分的证书移动到系统部分才行

* 安装movecert：
[链接](https://gitcode.com/open-source-toolkit/b1db8)
![alt text](image-42.png)

* 通过adb上传movecert到移动端：
![alt text](image-43.png)

* 通过Magisk面具进行安装（本地安装找到刚才上传好的movecert）并重启：
![alt text](image-44.png)

* 可以看到用户证书这里已经没有刚刚安装的burp证书了：
![alt text](image-45.png)

* 而是转移到了系统证书下：
![alt text](image-46.png)

## 5、设置代理

* 真机网络设置：
![alt text](image-47.png)

* Burp设置对应代理：
![alt text](image-48.png)

* 打开任意APP测试抓包结果：
![alt text](image-49.png)

* 成功抓到数据包

> 声明：
>
>本文章中所有内容仅供学习交流使用，不用于其他任何目的，不提供完整代码，抓包内容、敏感网址、数据接口等均已做脱敏处理，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无关，如有侵权，请联系作者（3892454318@qq.com）进行删除