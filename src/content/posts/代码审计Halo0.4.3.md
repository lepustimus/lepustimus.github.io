---
title: 代码审计 | Halo0.4.3
published: 2026-01-02
description: ''
image: ''
tags: []
category: '代码审计'
draft: false 
lang: ''
---

# 1、环境搭建
```java
JDK 1.8
Maven 3.8.8
```
- 点一下就好了：
![alt text](image-318.png)

# 2、审计过程
## 1、SSRF：
- 通过黑盒方式找到前端安装主题位置存在远程拉取功能点：
![alt text](image-319.png)
这里可以输入主题的远程链接进行在线拉取，因为可输入URL，所以猜测可能存在SSRF，所以进行dnslog回显判断
- 输入Burp dnslog地址并点击安装：
![alt text](image-320.png)
![alt text](image-321.png)
至此证明此处存在SSRF
- 通过URL路径定位到对应代码位置进行源码分析：
![alt text](image-322.png)
![alt text](image-323.png)
获取两个参数：远程拉取地址、主题名称，并且经过一些参数校验、路径拼接等基础处理操作之后来到如下代码：
```java
final String cmdResult = RuntimeUtil.execForStr("git clone " + remoteAddr + " " + themePath.getAbsolutePath() + "/" + themeName);
```
- 该行代码通过直接拼接固定命令git clone、用户传入的远程仓库地址remoteAddr以及本地主题存放路径，形成完整的命令之后，通过RuntimeUtil.execForStr()方法执行该命令
但是因为该行代码通过直接拼接固定命令git clone、用户传入的远程仓库地址remoteAddr以及本地主题存放路径，形成完整的命令后执行，由于对用户的输入参数未进行任何过滤，攻击者可以构造恶意的参数导致命令执行，实现RCE，也可通过控制remoteAddr参数指向内网地址进行探测，实现SSRF
![alt text](image-324.png)

## 2、任意文件读取：
- 搜索关键字reader找到如下代码：
![alt text](image-325.png)
- 前端通过tplName参数获取模板文件名，然后先后获取项目根路径和主题路径之后通过themePath.append(tplName)将模板文件名拼接上去用于读取该文件内容，但是因为没有对用户输入做任何过滤，导致攻击者可以输入“/../”等路径穿越字符实现任意文件读取：
![alt text](image-326.png)

## 3、存储型XSS：
- 全局搜索关键字“save”定位到如下代码：
![alt text](image-327.png)
- 通过校验前端提交的用户资料是否合法之后，进入了userService.create(user)进行下一步判断：
![alt text](image-328.png)
- 进入repository.save(domain)：
![alt text](image-329.png)
将用户输入内容更新到数据库中
整个过程中没有对用户输入进行过滤，所以此处存在存储XSS
- 构造XSS payload：
![alt text](image-330.png)
- 访问博客主页即可看到弹窗：
![alt text](image-331.png)

## 4、CVE-2021-42392：
- 通过pom.xml得知该系统存在cve-2021-42392漏洞：
![alt text](image-332.png)
- 访问路径http://192.168.2.12:8090/h2-console/
![alt text](image-333.png)
- 设置恶意类javax.naming.InitialContext以及恶意JNDI URL：
![alt text](image-334.png)
![alt text](image-335.png)
可以看到已经命令已经被执行，但由于我本地Java版本高于8u191，所以未能弹出计算器

## 5、Freemarker模板注入：
- 在前端有主图编辑的功能点：
![alt text](image-336.png)
![alt text](image-337.png)
- 通过URL路径定位到对应代码位置进行分析：
![alt text](image-338.png)
前端接收两个参数分别是文件名tplName以及文件内容tplContent，经过简单的文件内容校验以及获取到根路径以及主题路径之后，通过拼接文件名找到对应的文件，并通过fileWriter.write(tplContent)对该文件内容进行编辑
- 跟进fileWriter.write()：
![alt text](image-339.png)
将文件内容添加到指定文件中，写完会关闭文件流，出错时抛自定义异常
通过上述分析可知，整个编辑文件内容流程没有对恶意字符的过滤，所以会导致攻击者进行freemarker模板注入以及任意文件写入漏洞
- 将freemarker模板注入payload编辑到文件中：
```bash
<#assign value="freemarker.template.utility.Execute"?new()>${value("calc.exe")}
```
- ![alt text](image-340.png)
- 编辑成功之后访问首页即可弹计算器：
![alt text](image-341.png)
freemarker模板注入漏洞验证成功！
- Burp抓取编辑文件数据包：
![alt text](image-342.png)
- 修改文件名以及文件内容进行放包：
![alt text](image-343.png)
- 利用此处任意文件读取漏洞查看上述文件已经写入成功：
![alt text](image-344.png)

## 6、加密文章机制绕过：
- 抓取前端加密文章数据包：
![alt text](image-345.png)
![alt text](image-346.png)
- 通过URL路径定位到对应代码：
![alt text](image-347.png)
可以看到在第一处判断文章是否有加密的地方，是通过判断cookie：halo-post-password-[postid]=md5(密码)是否为空来判断该文章是否存在加密，而检验加密文章密码是否正确的代码，只是将用户输入的密码进行md5之后与数据库中存储的正确的md5密码进行了比对，但是无论比对结果对与错，只要在校验密码的数据包的请求头位置添加cookie: halo-post-password-[postid]=md5(任意值)，最终依然会返回到加密文章页面
- 所以利用顺序：
  - 1、在检查密码是否正确的数据包中加入cookie: halo-post-password-[postid]=md5(任意值)放包
  - 2、在加载加密文章的数据包中再次加入cookie: halo-post-password-[postid]=md5(任意值)放包
- 构造cookie值并且Burp开启拦截：
```bash
cookie: halo-post-password-49=96E79218965EB72C92A549DD5A330112
49：加密文章id
96E79218965EB72C92A549DD5A330112：任意值md5，这里是111
```

![alt text](image-348.png)
![alt text](image-349.png)
- 结果如下：

![alt text](image-350.png)
成功绕过文章加密机制！