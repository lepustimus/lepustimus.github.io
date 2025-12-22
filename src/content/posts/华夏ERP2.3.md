---
title: 代码审计 | 华夏ERP2.3
published: 2025-12-22
description: ''
image: ''
tags: []
category: '代码审计'
draft: false 
lang: ''
---

# 1、搭建过程
- 调整Java版本：
![alt text](image-284.png)
- 修改数据库连接账号密码并创建项目相关数据库：
![alt text](image-285.png)
- 将项目相关sql文件导入到数据库中：
```bash
source xxx.sql
```
![alt text](image-286.png)

# 2、审计过程
## 1、未授权访问：
### 分析过程：
- 直接看filter层的LogCostFilter代码：

![alt text](image-287.png)

该过滤器通过 @WebFilter 注解声明，配置如下：
- filterName：过滤器名称（LogCostFilter）
- urlPatterns = {"/*"}：拦截所有请求
- initParams：初始化参数，包含两类关键配置：
  - ignoredUrl：需要忽略拦截的资源（静态资源：.css、.js、.jpg 等）
  - filterPath：无需登录即可访问的接口（/user/login、/user/registerUser 等）


![alt text](image-288.png)

服务器启动之后，过滤器开始初始化，该过程会解析 @WebFilter 中的初始化参数，并存储到对应集合中，过程如下：
1. 解析filterPath：
  - 读取 filterPath 参数值（如 /user/login#/user/registerUser#/v2/api-docs）
  - 以 # 为分隔符，将字符串分割为数组 allowUrls（存储无需登录即可访问的接口）
  - 若参数为空，则 allowUrls 为 null，不影响后续逻辑
2. 解析ignoredUrl：
  - 读取 ignoredUrl 参数值（如 .css#.js#.jpg#.png#.gif#.ico）
  - 以 # 为分隔符，将字符串分割为数组 ignoredUrls
  - 遍历 ignoredUrls，将每个忽略规则（如 .css）存入 ignoredList 集合（供后续匹配使用）

![alt text](image-289.png)
![alt text](image-290.png)

doFilter 是过滤器的核心方法，判断所有请求是否符合放行条件，过程如下：
1. 登录校验：
  - 从 HttpSession 中获取 user 对象（登录成功后会存入该对象），若 userInfo != null，说明用户已成功登录，直接通过 chain.doFilter(request,response) 放行，不拦截后续请求
2. 特定页面校验：
  - 若请求 URL 不为空并且访问 /doc.html、/register.html、/login.html任意一个时，直接放行。
3. 资源页面校验：
  - 调用 verify 方法，通过正则判断请求 URL 是否有 ignoredList 中的内容，比如 .css、.js 等静态资源，如果存在则放行请求，不做拦截
4. 特殊接口校验：
  - 遍历 allowUrls 数组（如 /user/login、/user/registerUser），若请求 URL 以数组中的某个接口开头，比如 /user/login 、/v2/api-docs，直接放行，不做拦截
5. 拦截跳转页面：
  - 若上述所有条件都不满足（未登录、访问非公开资源），则通过servletResponse.sendRedirect("/login.html") 强制跳转到登录页，实现登录拦截

### 利用过程：
- 通过上述分析可知，该 filter 没有对路径穿越（如/../）场景设计任何校验逻辑，仅通过startsWith前缀匹配或模糊正则匹配 URL 的表面字符串，完全未对 URL 解析后的真实访问路径进行验证。所以存在未授权访问风险，攻击者可以通过白名单路径拼接路径穿越字符和恶意请求绕过登录拦截。比如：/user/login/../../secret.html、hello.css/../secret.html、doc.html/../secret.html等
![alt text](image-291.png)
![alt text](image-292.png)
![alt text](image-293.png)

## 2、fastjson反序列化：
### 分析过程：
- 从pom.xml中得知引用的fastjson是存在漏洞的版本：
![alt text](image-294.png)
- 全局搜索关键字：parseObject、JSON.parseObject等，定位到如下代码位置：
![alt text](image-295.png)
前端传入pageSize、currentPage、search三个参数后，后端直接通过JSON.parseObject函数对search参数的值进行反序列化处理，由于反序列化前未对该参数的用户输入做任何安全校验，攻击者可构造恶意序列化数据传入，程序在后端完成反序列化后便会触发恶意逻辑。

### 利用过程：
- 前端构造请求，并通过yakit抓包构造测试payload：
![alt text](image-296.png)
![alt text](image-297.png)
![alt text](image-298.png)
成功接收到dnslog请求，漏洞存在

## 3、SQL注入：
### 分析过程：
- 全局搜索"${"并跳转到AccountMapperEx.xml：
![alt text](image-299.png)
接收了AccountExample类型的参数，分别是name、serialNo、remark、offset、rows，然后通过模糊匹配在jsh_account表中查询对应的数据，后两个参数经过了预编译处理所以不存在注入问题，所以主要看前三个参数

- 全局搜索selectByConditionAccount并跳转过去：
![alt text](image-300.png)
可以看到对应参数传入到了select函数进行数据库查询处理

- 跟进select函数：
![alt text](image-301.png)
getAccountList中对search、name等参数进行了赋值传递给了accountService.select，getAccountList函数又是在select函数中进行的调用

- 跟进select函数：
![alt text](image-302.png)
![alt text](image-303.png)
当apiName参数值不为空时，会从容器（container）中根据这个名称（apiName）找到对应的查询处理器，然后调用它的查询方法来处理传入的参数。

- 跟进select函数：
![alt text](image-304.png)
通过apiName匹配动态路径，传入pageSize、currentPage、search三个参数并存储进parameterMap中，经过对pageSize、currentPage参数的简单判断之后，将apiName、parameterMap的值传入configResourceManager.select进行处理，结果赋值给list。如果list结果不为空，就将其设置到queryInfo的rows中，并通过configResourceManager.counts获取总记录数设置到total中，最后将所有数据封装到objectMap中，以JSON格式返回给前端；如果list为空，则返回一个空列表和提示信息。

### 利用过程：
- 找到存在漏洞参数的前端请求并抓包：
![alt text](image-305.png)
![alt text](image-306.png)
- 构造payload传入并发包：
![alt text](image-307.png)
![alt text](image-308.png)
成功延迟，存在SQL注入

## 4、存储XSS
### 分析过程：
``存储型XSS的审计主要关注程序中将用户输入持久化存储并在后续页面展示的位置。可通过全局搜索“@PostMapping”、“@RequestBody”、“save(”等关键词，定位数据接收和存储的代码部分进行审计，或者通过web页面找到一些可修改数据状态的功能点，然后定位到代码部分进行审计``

- 前端找到可编辑数据的位置：
![alt text](image-309.png)
- 定位到对应的代码位置：
![alt text](image-311.png)
定义两个参数：info（JSON格式的用户数据）和id（用户ID），通过JSON.parseObject()将info字符串反序列化为UserEx对象并设置ID，然后调用userService.updateUserAndOrgUserRel()方法更新用户信息。将用户可控的JSON数据转为对象并持久化存储
- 跟进updateUserAndOrgUserRel：
![alt text](image-312.png)
该方法主要是通过用户的输入对原有内容进行更新，过程中没有对用户输入内容的转义、过滤操作，可导致攻击者写入xss payload

### 利用过程：
- 编辑用户信息，并写入xss payload：
![alt text](image-313.png)
![alt text](image-314.png)
类似的问题还有很多，这里不再一一列举

## 5、垂直越权
### 分析过程：
- 上述代码因为只是校验了用户的登录情况，但是没有校验用户的权限从而导致低权限用户可以输入正常用户的id再构造出对应的info进行越权修改
- 参考存储XSS分析过程中的代码

### 利用过程：
- 登录低权限账号，并构造如下数据包并发包：
![alt text](image-315.png)
![alt text](image-316.png)
- 登录管理员账号查看结果：
![alt text](image-317.png)
用户111数据已经被越权修改为hacker123，类似问题还有很多，这里不再一一列举