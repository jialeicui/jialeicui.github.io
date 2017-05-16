---
layout: post
title:  "Laravel 的一些总结"
date:   2017-04-05
categories: Laravel PHP
---

*参考 Laravel 版本为 5.4*

### 我理解的PHP框架

* 框架最核心的就是解决了路由 (Router)
* 附带有 `MVC/ORM/Helper/Template/Middleware` 等特性

### Laravel 是什么

Laravel 就是一个大的 Container, 里面装了基本上所有我们需要的东西, 如果不够, 还可以自己往里塞

Laravel 里面默认都塞了什么? [这里有介绍](https://laravel.com/docs/5.4/facades#facade-class-reference), 比如前面提到的我认为的最核心的`路由 `

#### 核心

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
<?php
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



#### Service Provider

Service Provider 就是配置使用 Container 的一个实现, 基本上相当于配置了在什么时机调用 bind (singleton/instance等) 函数来绑定, 更细节的 `boot/$defer/register/provides` 参考文档即可

#### Facade

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



#### 初始化流程

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
  - BootProviders  -调用 Service Provider 的 boot 函数
- 通过 `Pipeline` 将 request 用 middleware 虑一遍, 交给 `dispatchToRouter` 返回的 Closure 处理



### 一些常用模块

#### 路由

 有了路由, 其实我们就可以完成一个网站了, 不需要其他的东西, 比如

```php
Route::get('/', function() {
  return '<html>blah...</html>';
});
```

和我们了解的 Http 对应, 路由支持 `GET/POST/PUT/PATCH/DELETE` , 这些都对应小写的函数, 当然还有 `resource` 这个函数一站解决 `RESTful` 风格的接口



#### Middleware

* 编写

  * 最重要的函数就是 `handle`. 放行就 `return $next($request);`, 否则就做其他操作. 
  * 可以把 `$response = $next($request);` 暂存, 做完想做的操作再 `return $response;`

* 注册和使用

  * Router 中, ( `app/Http/Kernel.php` 的 `$routeMiddleware` 配置了可用的 middleware )

    ```php
    Route::get('baz', 'FooController@bar')->middleware('auth');
    ```

  * Controller 中

    ```php
    <?php

    class FooController extends Controller
    {
        public function __construct() {
            $this->middleware('auth');
            $this->middleware('log')->only('index');
            $this->middleware('subscribed')->except('store');
        }
    }
    ```

  * 整个路由规则中

    `app/Http/Kernel.php` 中的 `$middlewareGroups` 配置了对应路由文件使用的 middleware, 可见 `web` 和 `api` 使用的有区别, `api` 路由使用的很少.

    如果 `api` 路由需要识别 cookie, 使用 session, 添加 `\App\Http\Middleware\EncryptCookies::class,\Illuminate\Session\Middleware\StartSession::class` 即可

#### ORM

* 基本使用

  * $table/$timestamps/$connection 等参考文档
  * $fillable - 定义使用的字段, 在使用 `create` 插入数据的时候会进行过滤
  * $casts - 不局限于数据库类型, 可以对 object 的字段自定义类型, 平时用的比较多的就是将 json 字符串 cast 成 array
  * $hidden - get 的时候不包含的字段, 提供 api 时尤其有用
  * setXxxAttribute - 用于设置属性时的特殊处理, 同等的 getXxxAttribute 是获取处理, 'Xxx' 省略的时候是全局处理
  * scopeXxx - 用来比较优雅的添加限定函数, 用来修饰查询, 使查询更直观
  * 更多的定制可以重写成员函数进行, 平时重写比较多的有 `boot/create/destory` 等

* Relationship

  * hasOne/belongsTo/hasMany/belongsToMany/hasManyThrough 等参考文档

  * 查询时携带多重关系 `Foo::with('bar.baz.tmp')->where(...)`

  * 查询时关联条件

    ```php
    <?php
    $posts = Post::whereHas('comments', function ($query) {
        $query->where('content', 'like', 'foo%');
    })->get();
    ```

* Collections

  * 基类提供的函数可参考[文档](https://laravel.com/docs/5.4/collections#method-first)

  * first - 可以传入一个 callable 和一个 default ([源码](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Support/Collection.php#L471)), 所以为了避免一串链式的查询之后结果为空再直接取属性的时候有 `Trying to get property of non-object` 这个异常, 可以这样使用

    ```php
    <?php

    function getFoobar($default) {
        return Foo::where('id', $id)->get()->first(null, new Foo(['bar'=>$default]))->bar;
    }
    ```

  * 如果有特殊需求, 可以在 Model 中使用自定义的 Collection, 参考[文档](https://laravel.com/docs/5.4/eloquent-collections#custom-collections)