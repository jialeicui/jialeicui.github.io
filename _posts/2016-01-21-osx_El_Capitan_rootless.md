---
layout: post
title:  "OSX El Capitan下的 Rootless"
date:   2016-01-21
categories: OSX PHP
---

### 引出

试用 `Laravel` 的时候需要给PHP安装 intl 扩展, 简单的几个命令却执行失败.

```bash
cd /usr/lib/php
sudo php install-pear-nozlib.phar
```
提示没有权限在 `/usr/lib/` 创建文件夹, sudo 也不行.

### 是什么

这个是 Apple 在 `OSX 10.11 El Capitan` 添加的新特性, 简单的讲, 它的功能就是让用户即使切换到了 root 也无法修改一些重要的系统文件以及进程, 进而保障系统的安全.

假设一个情景, 使用 AppCleaner 删除某个程序的时候, 由于是要删除关键位置的文件, 系统会弹出对话框请求权限, 

![](/assets/image/posts/ask_permission.png)

这时候我们其实别无选择, 乖乖输密码, 如果 AppCleaner 是个恶意程序, 那么他对我的系统想干嘛就干嘛. 这个新特性可以有效的防止恶意程序对系统的破坏, 它包含的限制如下(root也不行):

* 不能修改 `/System`, `/bin`, `/sbin`, 或 `/usr (除了 /usr/local)`, 只有 Apple认证的程序安装包或者更新才能修改.
* 不能 **attach** 系统进程
* 不能加载未认证的系统模块

### 如何临时禁用

这个特性初衷是好的, 但是会影响一些软件的使用, 比如开始我遇到的问题. 为了安装程序到 `/usr/lib` 要临时禁用这个特性.

#### 禁用

重启电脑进入到恢复模式 (开机时按住 Command+R), 之后打开终端, 输入 `csrutil disable` 命令, 重启即可. 禁用之后就可以随心所欲的安装自己的软件了.

#### 启用

类似的操作, 命令换成 `csrutil enable`.

参考:

[Quora: Can someone elaborate on the OS X 10.11 feature called 'Rootless'?](https://www.quora.com/Can-someone-elaborate-on-the-OS-X-10-11-feature-called-Rootless)  
[What is the “rootless” feature in El Capitan, really?](https://apple.stackexchange.com/questions/193368/what-is-the-rootless-feature-in-el-capitan-really)



