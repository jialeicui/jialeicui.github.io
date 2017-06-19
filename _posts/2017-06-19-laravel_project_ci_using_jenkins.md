---
layout: post
title:  "使用jenkins对Laravel项目进行持续集成"
date:   2017-06-19
categories: PHP Laravel Jenkins CI
---

## 搭建 Jenkins

参考[官方说明](https://jenkins.io/download/)

本次测试搭建在 CentOS 上, 所以用的是 yum 的方式

``` sh
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum install jenkins -y
yum install jre -y
service jenkins start
```

如果没有错误, jenkins 会在 8080 端口启动了 http 服务, 在浏览器输入 ip:8080 应该就能访问了.

按照说明, 填写初始管理员密码 -> 安装需要的插件 -> 添加管理员账户, 就 OK 了

## 为运行 Laravel 准备环境

[参考](https://webtatic.com/packages/php56/)

```shell
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
yum install php56w php56w-opcache php56w-pecl-xdebug php56w-mysqli phpunit -y
```

说明:

* xdebug 是为了 phpunit 出覆盖率



## 再配置 Jenkins

* 添加 PHP 覆盖率插件

  Manage Jenkins -> Manage Plugins, 添加 `Clover PHP plugin` 和 `Gitlab Hook Plugin` 插件, 必要时重启 Jenkins

* 添加 ssh 私钥

  Credentials -> System -> Global credentials -> Add Credentials

  SSH Username with private key, 选择合适的选项 ( 我使用的是粘贴私钥 )

## 为 Jenkins 添加 Job

选项目类型( 我选的 Freestyle project ), 添名字 ( 测试项目: 报警中心 )

* Source Code Management

  Git -> `git@git.letv.cn:inf/alarm_center.git` , 选好刚才添加的 Credential, 选择分支

* Build Triggers

  勾选 GitHub hook trigger for GITScm polling, 配合 Gitlab Hook Plugin

* Build

  选择 shell, 填写类似脚本

  ```sh
  /usr/local/bin/composer install --prefer-dist --optimize-autoloader

  # 生成 .env 文件, 如果需要数据库, 也在这里配置
  echo 'APP_ENV=local
  APP_KEY=base64:Qfj14MI0OZyc/BvaYcLBdXHWzcU2bqDz84Ro0+Tw7eU=
  APP_DEBUG=true
  APP_LOG_LEVEL=debug
  APP_URL=http://localhost' > .env

  vendor/bin/phpunit --coverage-clover ./build
  ```

* Post-build Actions

  Publish Clover PHP Coverage Report, 位置是上面脚本中 phpunit 的输出文件  ./build 

  还有很多其他 action 可选, 比如发邮件之类的

## 在 Gitlab 上添加 hook

项目 -> Setting -> Web Hooks -> 选好trigger, url 添 `http://your-jenkins-server/gitlab/build_now`  [参考](https://github.com/elvanja/jenkins-gitlab-hook-plugin#build-now-hook)



这样, 在出发 Gitlab trigger 时, jenkins 就会开始对项目进行测试了, 执行脚本中添加部署策略, 则可以进行自动部署