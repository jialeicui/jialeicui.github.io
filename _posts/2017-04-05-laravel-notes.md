---
layout: post
title:  "Laravel 的一些总结"
date:   2017-04-05
categories: Laravel PHP
---

*参考 Laravel 版本为 5.4*

#### 我理解的PHP框架

* 框架最核心的就是解决了路由 (Router)
* 附带有 `MVC/ORM/Helper/Template/Middleware` 等特性

#### Laravel 是什么

Laravel 就是一个大的 Container, 里面装了基本上所有我们需要的东西, 如果不够, 还可以自己往里塞

Laravel 里面默认都塞了什么? [这里有介绍](https://laravel.com/docs/5.4/facades#facade-class-reference), 比如前面提到的我认为的最核心的`路由 `

##### 核心

前面写 Laravel 是一个大的 Container, 下面是一个最简陋的山寨



```php
<?php

class Container {
    protected $instances = [];
    protected $bindings = [];

    public function bind($abstract, Closure $concrete) {
        $this->bindings[$abstract] = $concrete;
    }

    public function make($abstract) {
        if (isset($this->instances[$abstract])) {
            return $this->instances[$abstract];
        }

        $object = $this->build($this->bindings[$abstract]);
        $this->instances[$abstract] = $object;

        return $object;
    }

    public function build(Closure $concrete) {
        return $concrete($this);
    }
}
```



这个 class 只有三个成员函数:

* bind 负责将实例化方法注册到 Container
* build 负责实例化
* make 负责返回对象

下面是使用的例子



```php
class Foo {
}

$c = new Container();
$c->bind('foo', function($container) {
    return new Foo();
});

$f = $c->make('foo');

print_r($f);

/* 将会打印
Foo Object
(
)
*/
```

完全版本的 Container 参考 [Illuminate\Container\Container](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Container/Container.php)



##### Service Provider

Service Provider 就是配置使用 Container 的一个实现, 基本上相当于配置了在什么时机调用 bind (singleton/instance等) 函数来绑定, 更细节的 `boot/$defer/register/provides` 参考文档即可

##### Facade

Facade 是便捷访问 Container 中资源的一种方式, 使用起来像是在使用一个类的静态方法, 但其实是通到了 Laravel 内部, 有 Facade 之后的使用方式就是简单的调用

```php
<?php
use Foo\Bar;

function test() {
  Bar::baz();
}
```

自定义 Facade 的方法

```php
<?php

namespace Foo;

use Illuminate\Support\Facades\Facade;

class Bar extends Facade
{
    protected static function getFacadeAccessor() {
        return 'router';
    }
}
```

这样我们使用 `Foo\Bar` 这个类的时候就相当于调用 `app('router')` ( 抛开 mock 特性 )

整个 Facade 的核心就是 [Illuminate\Support\Facades\Facade](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Support/Facades/Facade.php#L213) 中的 `__callStatic` 函数, 此函数通过 `getFacadeAccessor` 获取到我们定制的 alias, 再从 Laravel 这个 Container 中通过  ArrayAccess 的  `offsetGet` 这个特性来间接的调用到了 make 函数, 整条路就打通了.

所以这里我们用 `Foo\Bar` 就和用 `Router` 是一个效果, Laravel 的路由 Facade 就是这么来的, 如果想看具体实现的类, 只需要查找 bind 或者 alias 的地方就能找到.



##### 初始化流程

- 入口 `public/index.php` 
- 实例化一个 `Illuminate\Foundation\Application` 为 $app 变量 (这里包含额外的一些初始化)
- 将 `App` 命名空间下的 `Http\Kernel`, `Console\Kernel` , `Exceptions\Handler` 注册为单例
- 实例化  `Http\Kernel`, 调用 `handle` 函数
- 调用 `sendRequestThroughRouter` , 进行关键初始化
  - DetectEnvironment
  - LoadConfiguration
  - ConfigureLogging
  - HandleExceptions
  - RegisterFacades  -初始化内置的 Facade ([AliasLoader](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Foundation/AliasLoader.php#L150) 具体实现, 依靠 spl_autoload_register 和 class_alias)
  - RegisterProviders  -初始化内置 Service Provider ([Application](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Foundation/Application.php#L546) 此处区分实例化时机)
  - BootProviders  -调用 Service Provider 的 boot 函数, 对
- 通过 `Pipeline` 将 request 用 middleware 虑一遍, 交给 `dispatchToRouter` 返回的 Closure 处理



#### 常见模块

##### 路由

 有了路由, 其实我们就可以完成一个网站了, 不需要其他的东西, 比如

```php
Route::get('/', function() {
  return '<html>blah...</html>';
});
```

和我们了解的 Http 对应, 路由支持 `GET/POST/PUT/PATCH/DELETE` , 这些都对应小写的函数, 当然还有 `resource` 这个函数一站解决 `RESTful` 风格的接口