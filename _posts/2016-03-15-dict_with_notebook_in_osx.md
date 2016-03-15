---
layout: post
title:  OSX 下的字典添加生词本功能
date:   2016-03-15
categories: OSX
---

效果图

![](/assets/image/posts/dict-in-osx.png)

使用的软件(插件)有:

* [BetterDictionary](http://pooriaazimi.github.io/BetterDictionary/)
* [EasySIMBL](https://github.com/norio-nomura/EasySIMBL)
* [SIMBL](http://www.culater.net/software/SIMBL/SIMBL.php)

系统环境为: OS X 10.11.3 (El Captian)

实现功能的软件是 `BetterDictionary`, 在它的官网有完整的安装使用说明.  
这里记录主要是因为它没有针对 `OS X El Captian` 说明, 因为 SIMBL 在这个版本的 OS X 失效了, 但是可以通过 [这里](https://github.com/norio-nomura/EasySIMBL/issues/26#issuecomment-117028426) 的说明使其生效

下面是安装各个软件的步骤:

1. 下载 `SIMBL`, v0.9.9
2. 关闭 SIP, 方法参考 [OSX El Capitan下的 Rootless](/blog/osx_El_Capitan_rootless.html)
3. 安装 `SIMBL`, 安装过程如下:

   ```sh
   sudo installer -verbose -pkg Downloads/SIMBL-0.9.9/SIMBL-0.9.9.pkg -target /
sudo rm -rf /System/Library/ScriptingAdditions/SIMBL.osax
sudo mv /Library/ScriptingAdditions/SIMBL.osax /System/Library/ScriptingAdditions/
sudo cp -p /System/Library/ScriptingAdditions/SIMBL.osax/Contents/Resources/SIMBL\ Agent.app/Contents/Resources/net.culater.SIMBL.Agent.plist /System/Library/LaunchAgents/
sudo sed -e "s/Library/System\/Library/" -i "" /System/Library/LaunchAgents/net.culater.SIMBL.Agent.plist
```

4. 下载 `EasySIMBL`, v1.7.1, 解压放至 `/Applications`
5. 下载 `BetterDictionary`, v0.992, 解压
6. 运行 EasySIMBL, 把 `BetterDictionary.bundle` 拖进 `EasySIMBL`, 勾选 `Use SIMBL`
7. 重启打开字典即可看到效果


