---
layout: post
title:  在 CodeIgniter 中使用 Laravel 模板引擎 Blade
date:   2016-03-03
categories: PHP CodeIgniter Laravel
---

CodeIgniter 对视图模板的支持比较欠缺, 在使用 Laravel 的过程中, 觉着 Blade 比较简单好用, 上网搜了下, 找到了[这个项目](https://github.com/PhiloNL/Laravel-Blade), 是单独的 Blade 模板引擎, 理论上可以用在任何 PHP 项目中, 所以尝试结合到 CodeIgniter 中.  

下面是移植过程

#### 目标

* 支持 Blade 方式的解析渲染
* 类似于 Laravel 中的使用方式: `return view('view', $data);`

#### 环境准备

这里使用的是 [CodeIgniter 3.0.4](https://github.com/bcit-ci/CodeIgniter/archive/3.0.4.zip) 和 [Laravel-Blade](https://github.com/PhiloNL/Laravel-Blade)  

在 composer.json 里添加 (这里我们使用 Blade Laravel 5.1)

```
"philo/laravel-blade": "3.*"
```
然后 `composer install`

#### 使用 Blade

* 按照 Laravel-Blade 的 Usage 中的介绍, 在使用的 Controller 里相应的位置添加下面代码

```
require 'vendor/autoload.php';

use Philo\Blade\Blade;

$views = __DIR__ . '/../views';
$cache = __DIR__ . '/../cache';

$blade = new Blade($views, $cache);
echo $blade->view()->make('welcome_message')->render();
```

* 在 view 文件名种添加 `.blade`, 类似于 `welcome_message.blade.php`
* 将 `application/cache` 设置为 webserver 可读写

这样就可以使用 Blade 引擎了

#### Laravel 方式使用

按照上面的方式使用较为繁琐, 我们可以将上面的使用方式简化为类似于 Laravel 种的使用方式.

我使用的方案是添加一个自定义的 [helper](http://www.codeigniter.com/user_guide/general/helpers.html), 在 `application/helpers` 里添加一个 helper, 名为 `view_helper.php`, 内容如下:

```php
<?php

require_once 'vendor/autoload.php';
use Philo\Blade\Blade;

if (!function_exists('view')) {
    function view($name = NULL, $data = [], $mergeData = []) {
        $CI =& get_instance();
        if (!isset($CI->blade)) {
            $views = __DIR__ . '/../views';
            $cache = __DIR__ . '/../cache';
            $CI->blade = new Blade($views, $cache);
        }
        echo $CI->blade->view()->make($name, $data, $mergeData)->render();
    }
}
```

在 `autoload.php` 中的 `$autoload['helper']` 中添加 `view`. 后面就可以直接在 Controller 里按照 Laravel 的方式使用了.

[demo源码](https://github.com/jialeicui/CodeIgniter-Blade-demo)  


