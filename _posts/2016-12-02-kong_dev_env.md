---
layout: post
title:  "API网关 Kong 的开发环境搭建"
date:   2016-12-02
categories: API网关
---

### kong src

[从源码安装说明](https://www.postgresql.org/download/linux/redhat/)

### openssl

启动报错: `Error: /usr/local/share/lua/5.1/kong/cmd/start.lua:23: nginx: [emerg] at least OpenSSL 1.0.2e required but found OpenSSL 1.0.1e-fips 11 Feb 2013`

更新openssl


```
cd /usr/src
wget https://www.openssl.org/source/openssl-1.0.2-latest.tar.gz
tar -zxf openssl-1.0.2-latest.tar.gz


cd openssl-1.0.2j  (这里可能不一样)
./config
make
make test
make install

```

### openresty
目前openresty的最新版本是 `1.11.2.1`, 但看kong的官网说是需要 `1.9.15.1`, 安完之后报错, 说是需要 `1.11.2.1`, WTF!
所以这些依赖还是不要太信官网, 可能他们更新的不够及时, 信源码: kong/meta.lua 里有版本依赖信息

另, openresty直接通过yum就可以安装到最新的版本, 但是这样安装的没有resty这个命令行, 所以还是老实从源码安装了(或者从https://github.com/openresty/resty-cli下一个resty)

```
yum install openssl-devel gcc
wget https://openresty.org/download/openresty-1.11.2.1.tar.gz

tar -xvf openresty-1.11.2.1.tar.gz
cd openresty-1.11.2.1
./configure \
  --with-pcre-jit \
  --with-ipv6 \
  --with-http_realip_module \
  --with-http_ssl_module \
  --with-http_stub_status_module \
  --with-openssl=/usr/src/openssl-1.0.2j
  
gmake && gmake install
export PATH=/usr/local/openresty/bin:$PATH

```

### luarocks
下载最新的源码: https://github.com/luarocks/luarocks/releases

```
./configure && make && make install
```
`https://luarocks.org` 托管在 linode, 因为GFW的问题, 可能需要梯子, 借用宿主机的proxy

```
export http_proxy=http://10.211.55.2:1087
export https_proxy=http://10.211.55.2:1087
export all_proxy=http://10.211.55.2:1087
export no_proxy=localhost,127.0.0.1

```


### postgresql
开发机上的CentOS 7直接yum安装的postgresql版本是9.2的, kong要求要9.4+, 所以依照postgresql官网的说明安装更新的版本, 本次安装9.5

```
yum install http://yum.postgresql.org/9.5/redhat/rhel-7-x86_64/pgdg-redhat95-9.5-2.noarch.rpm
yum install postgresql95-server postgresql95-contrib
service postgresql-9.5 initdb
chkconfig postgresql-9.5 on service postgresql-9.5 start
```

其中有一个步骤可能出错: `service postgresql-9.5 initdb`, 报错如下

```
The service command supports only basic LSB actions (start, stop, restart, try-restart, reload, force-reload, status). For other actions, please try to use systemctl.
```

换成执行`/usr/pgsql-9.5/bin/postgresql95-setup initdb` 即可

之后一路启动服务, 开机启动应该都没问题

启动kong的时候报错: `Error: /usr/local/share/lua/5.1/kong/cmd/start.lua:18: [postgres error] FATAL: Ident authentication failed for user "kong"`

添加数据库和用户

```
su - postgres
createdb kong
psql kong
create user kong with password 'kong';
grant ALL PRIVILEGES ON DATABASE kong to kong;
```
OK, 添加完成之后, 启动的报错没有变.(除了启动kong来验证, 也可以直接用命令行验证:`psql -d kong -U kong -w`

`su - postgres`之后的路径就是postgresql的目录, 它配置文件等等都在这里,

`tail -f pg_log/postgresql-Wed.log`, 看到报错信息如下:

```
< 2016-11-30 13:50:39.212 EST >LOG:  could not connect to Ident server at address "127.0.0.1", port 113: Connection refused
< 2016-11-30 13:50:39.213 EST >FATAL:  Ident authentication failed for user "kong"
< 2016-11-30 13:50:39.213 EST >DETAIL:  Connection matched pg_hba.conf line 82: "host    all             all             127.0.0.1/32            ident"

```

先是端口113 Connection refused, 然后是验证失败的信息, 显示的是 pg_hba.conf 的82行配置
kong使用的是密码验证, 把这行的 `ident` 改成 `md5`, 保存

重启服务: `service postgresql-9.5 restart` 即可


```
kong start
ulimit is currently set to "1024". For better performance set it to at least "4096" using "ulimit -n"
Kong started
```

按照提示, 修改一下ulimit -n即可