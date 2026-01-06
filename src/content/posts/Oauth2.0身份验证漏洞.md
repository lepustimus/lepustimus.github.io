---
title: Oauth2.0身份验证漏洞
published: 2026-01-06
description: ''
image: ''
tags: []
category: 'SRC漏洞挖掘'
draft: false 
lang: ''
---

# 1、介绍
- 背景：OAuth2.0作为目前最主流的开放授权协议，被广泛应用于各类跨平台、第三方登录场景，例如登录页面的微信登录、QQ登录、微博登录等，本质上都是基于OAuth2.0协议实现的身份验证与授权流程
- 要素：
  - 资源所有者（Resource Owner）：即用户本人，拥有对资源服务器上自身资源的控制权，能够授权第三方应用访问其资源。
  - 客户端（Client）：指需要获取用户资源的第三方应用，例如使用微信登录的某款游戏APP、使用GitHub登录的某款开发工具等。客户端需要提前在授权服务器注册，获取唯一的客户端ID和客户端密钥（Client Secret）用于身份标识。
  - 授权服务器（Authorization Server）：负责验证资源所有者的身份，处理授权请求，并向客户端发放授权凭证（如授权码、令牌）的服务器。例如微信的授权服务器、GitHub的OAuth授权服务器等。
  - 资源服务器（Resource Server）：存储用户资源的服务器，能够验证客户端携带的令牌有效性，并根据令牌权限向客户端返回对应的资源。例如微信的用户信息服务器、GitHub的代码仓库服务器等。
- 认证流程举例：
  - 角色分配：
    - 小明：资源所有者（用户）
    - 快递站：第三方应用（客户端）
    - 小区物业：授权服务器
    - 快递仓库：资源服务器
  - 过程：小明先去物业进行登记，登记之后告诉物业可以让快递站帮我取快递，物业验证了小明的身份之后，给了快递站一个取件码（授权码），快递站拿着这个取件码到物业换了一张提货单（访问令牌），然后快递站拿着提货单去快递仓库提去小明的快递，仓库验证提货单有效之后，将小明的快递交给了快递站，然后再次转交给小明

# 2、认证流程图
![alt text](image-351.png)
> 网图来自：https://portswigger.net/web-security/oauth/grant-types

# 3、漏洞类型
## 1、开放重定向OAuth令牌劫持：
- 成因：当OAuth客户端存在开放重定向漏洞且未对redirect_uri参数进行严格校验时，攻击者可以构造恶意回调地址指向该重定向端点，并在参数中嵌入外部攻击域名；用户完成授权后，授权服务器将访问令牌或授权码通过URL参数或片段传递给该回调地址，客户端服务会执行重定向跳转至攻击者控制的域名，导致令牌在HTTP重定向过程中被截获，攻击者即可从URL中提取令牌并实施身份劫持
- Burp靶场实验：
![alt text](image-352.png)
依然访问My account进行登录，登录之后注销账号再次点击My account，并抓取此过程数据包进行分析：
![alt text](image-353.png)
可以看到该数据包用于身份验证，认证成功之后返回包中已经返回token，并跳转到/oauth-callback接口：
![alt text](image-354.png)
/oauth-callback接口通过上述获得的token跳转到/me接口进行访问：
![alt text](image-355.png)
/me接口用于返回用户信息：
![alt text](image-356.png)
经过测试，该靶场并未对redirect_uri做完全白名单校验，导致攻击者可以通过/../进行路径穿越，实现恶意URL重定向，可利用网站首页任意文章最下面Next post功能点进行恶意URL重定向
在exploit-server进行payload构造：
![alt text](image-357.png)
Burp开启拦截，重新点击My account：
修改如下数据包的redirect_uri参数值并放包：
![alt text](image-358.png)
查看日志：
![alt text](image-359.png)
![alt text](image-360.png)
可以看到这里截获了一段token，Burp repeater重放/me接口并替换token为截获的这个：
![alt text](image-361.png)
成功获取到administrator账号的apikey
回到My Account页面提交即可：
![alt text](image-362.png)
![alt text](image-363.png)

## 2、隐式授权滥用：
- 成因：由于缺少授权码（code）这一中间环节，令牌通过URL片段或重定向直接返回给浏览器，使得攻击者能够通过CSRF、恶意脚本、Referer泄露等方式截获令牌，从而假冒用户身份访问受保护资源。该模式本质上不适合高权限场景，但常被误用，加上开发者容易因此省略必要的安全校验（如state参数验证、令牌与用户身份的绑定验证），进一步放大了风险
- Burp靶场实验：
![alt text](image-364.png)
ACCESS THE LAB开启靶场来到靶场页面并点击My account来到登录页面：
![alt text](image-365.png)
![alt text](image-366.png)
通过wiener:peter进行登录并分析登录认证流程数据包：
![alt text](image-367.png)
登录之后点击Continue并抓包进行分析：
该数据包进行认证后会重定位到/oauth-callback路径：
![alt text](image-368.png)
/oauth-callback的数据包中又会跳转到/me路径下：
![alt text](image-369.png)
/me路径数据包验证了用户邮箱并返回相关的身份信息：
![alt text](image-370.png)
因为存在漏洞，所以通过/me API接口验证邮箱之后，后续操作用到的token默认通过授权且没有再次和邮箱做校验，所以在/authenticate修改邮箱为受害者邮箱即可获取受害者信息：
![alt text](image-371.png)
修改/authenticate中的邮箱再次进行认证流程即可获取到受害者信息：
![alt text](image-372.png)
![alt text](image-373.png)

## 3、授权码劫持：
- 成因：当OAuth流程中缺少对state参数的验证或未将其与用户会话绑定时，攻击者可预先获取自己的授权码并构造恶意链接。受害者点击该链接后，由于缺失state参数校验，OAuth服务会将攻击者的授权码与受害者当前登录的本地账户错误关联，从而实现身份绑定劫持，使攻击者后续能通过自己的OAuth身份访问受害者的账户数据
- Burp靶场实验：
![alt text](image-374.png)
ACCESS THE LAB进入靶场之后，点击My account进行登录：
![alt text](image-375.png)
登录之后点击绑定Attach a social profile：
![alt text](image-376.png)
![alt text](image-377.png)
抓包分析上述过程的流程：
该数据包进行了身份认证并且回调到/oauth-linking接口，但是过程中并没有state或者其他校验csrf的参数：
![alt text](image-378.png)
该数据包用于绑定博客信息，code参数就是身份验证之后拿到的授权码，网站（客户端）通过授权码可以获取到token，之后通过token向资源服务器请求博客信息并绑定到攻击者账号：
![alt text](image-379.png)
所以可能导致攻击者利用csrf诱导受害者绑定自己的博客到攻击者账号
重新点击绑定博客链接并通过Burp进行拦截：
![alt text](image-380.png)
复制该URL并drop所有数据包，并通过Burp自带的exploit-server发送构造好的csrf payload：
![alt text](image-381.png)
发送之后，点击注销账号：
![alt text](image-382.png)
再次点击绑定博客：
![alt text](image-383.png)
可以看到绑定高权限账号的博客成功，删除carlos即可：
![alt text](image-384.png)