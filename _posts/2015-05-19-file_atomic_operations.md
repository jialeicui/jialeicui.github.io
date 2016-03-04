---
layout: post
title:  "文件原子操作"
date:   2015-05-19
categories: file atomic
---

假设一个场景:  

>进程 A 往文件中写入数据  
另一个进程 B 不停的读取文件的内容, 进行一些操作

这个场景类似于 [竞争条件](http://www.ibm.com/developerworks/cn/linux/l-sprace.html), 如果进程 A 写数据时执行到一半, 那么 B 进程读取到的数据就是不完整的, 导致 B 进程不能正常工作.

要解决这个问题, 就必须保证每个进程在操作文件时, 没有其他进程同时操作, 保证操作是原子操作即可.

### 哪些操作是原子操作?

*说明: 以下原子操作说明基于 linux 操作系统, Windows 和 OS X 有部分的操作和 linux 不一致, 不能保证原子操作.*

* mv -T \<oldsymlink\> \<newsymlink\>
* link(oldpath, newpath)
* symlink(oldpath, newpath)
* rename(oldpath, newpath)
* open(pathname, O\_CREAT \| O\_EXCL, 0644)
* mkdir(dirname, 0755)

这些命令具体的操作结果参考 linux 帮助文档

### 使用 rename 保证原子操作
这里使用 rename 的方式有很多种, 最终都会进行原子的系统调用 例如:

* 使用 C 标准库中的 rename (man 2 rename)
* 使用 Shell 中的 rename 或者 mv (man 1 mv)

由于 `rename` 是原子操作, 所以我们可以依靠这个特性来避免文章开始假设场景中出现异常.  
具体做法是:

1. 创建临时文件 tmp
2. 在临时文件 tmp 中写入内容
3. 将 tmp rename 为目标文件

这里是一个 shell 例子

```sh
# 目标文件
dst="/var/some/file"
# 生成一个临时文件, 保证文件名不会与其他文件冲突
tmp="${trg%/*}/tmp/${trg##*/}.`date +%s`.$$"
printf "foo\nbar\nbaz" > "$tmp" || exit 111
mv "$tmp" "$dst"
```
**注: rename 之前保证文件已经写入磁盘是更好实践**  
例如使用 `flush`, `fsync` 等系统调用.  
这里是一个 python 例子

```py
def test():
	f = open(tmpfile, 'w')
	f.write(content)
	f.flush()
	os.fsync(f.fileno()) 
	f.close()
	os.rename(tmpfile, dstfile)
```

### rename 几条说明

具体参考 `man 2 rename`, 使用 `mv` 命令测试, 下面使用 rename(oldpath, newpath) 进行说明

* 如果 newpath 已经存在, 会覆盖 newpath (有权限的话)
* 如果 newpath 存在, 但是 rename 失败, newpath 不会被删除
* 如果 oldpath 是一个 symbolic link, 那么 link 被重命名(指向的文件不变)
* 如果 newpath 是一个 symbolic link, 那么 link 会被覆盖(指向的文件不变)

### 例外情况

* rename 的的 src 和 dst 所在文件系统如果不一样, 可能不会是原子操作(特别的, 使用 C 语言标准库中的 rename 时, 文件系统不一致会导致失败)
* OS X中的 mv 不是原子操作, [参考](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/mv.1.html)
* Windows 中使用 C 标准库的 rename 时, 如果 dst 已经存在, 调用会失败 (使用 Windows API 可以保证原子操作)

参考:  
[atomic writing to file with Python](http://stackoverflow.com/questions/2333872/atomic-writing-to-file-with-python)  
[Things UNIX can do atomically](http://rcrowley.org/2010/01/06/things-unix-can-do-atomically.html)  
[Atomically Replacing Files and Directories](http://axialcorps.com/2013/07/03/atomically-replacing-files-and-directories/)  
[Safely Updating Critical Files on a Linux System](http://linuxvm.org/info/howtos/rename.html)  
[Rename \(computing\)](http://en.wikipedia.org/wiki/Rename_\(computing\))  
[Linux 同步方法剖析](http://www.ibm.com/developerworks/cn/linux/l-linux-synchronization.html)  