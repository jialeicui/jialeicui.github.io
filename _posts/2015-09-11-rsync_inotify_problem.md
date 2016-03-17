---
layout: post
title:  "rsync + inotifywait 遇到的小坑"
date:   2015-09-11
categories: rsync linux inotifywait
---

### 初衷

用`rsync`推送一些配置文件到指定目录, `inotifywait` 监控此目录进行配置加载, 监控脚本类似下面这样

```
#!/bin/sh
#推送的配置的路径
CONF_PATH=/usr/home/xxx
while true; do
    F=`inotifywait -q --format %f -e create $CONF_PATH`
    #do something
    echo $F
done
```

### 遇到的问题

通过程序日志发现

* 经常的会有类似 `.XX.conf.ztjZBc` 名字的文件被加载
* 没有正常的配置文件被加载  

### 分析

**异常文件名问题**

看到文件名是以`.`开始, 并且以这种后缀结束, 猜测是 rsync 的临时文件用[文件原子操作](/blog/file_atomic_operations.html),保证使用文件的程序直接得到完成的目标文件.  

man rsync, 在 `-T, --temp-dir=DIR`参数介绍中

>This option instructs rsync to use DIR as a scratch directory when creating temporary copies of the files transferred  on  the  receiving
side.  The default behavior is to create each temporary file in the same directory as the associated destination file.

可以看出猜测是正确的  

**正常配置不加载问题**

监控脚本中只监控了 `create` 事件, 而 rsync 推送的目标文件是通过 `mv` 完成的, 所以监控 `create` 事件起不了作用

### 解决

**1.解决加载临时文件的问题**

inotifywait 有一个 `--exclude` 选项, 支持正则表达式匹配

>Do not process any events whose filename matches the specified POSIX extended regular expression, case sensitive.

修改脚本如下

```
#!/bin/sh
#推送的配置的路径
CONF_PATH=/usr/home/xxx
while true; do
    F=`inotifywait -q --format %f -e create  --exclude "^$CONF_PATH/\..*$" $CONF_PATH`
    #do something
    echo $F
done
```
经测试, 不再加载临时文件了

**2.解决加载临时文件的问题**

inotifywait 中监控的事件有一个 `moved_to`

>A file or directory was moved into a watched directory.  This event occurs even if the file is simply moved from and to the  same  direc-tory.

修改脚本如下, 功能OK

```
#!/bin/sh
#推送的配置的路径
CONF_PATH=/usr/home/xxx
while true; do
    F=`inotifywait -q --format %f -e create,moved_to  --exclude "^$CONF_PATH/\..*$" $CONF_PATH`
    #do something
    echo $F
done
```