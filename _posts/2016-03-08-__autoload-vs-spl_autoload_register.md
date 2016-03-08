---
layout: post
title:  __autoload vs spl_autoload_register
date:   2016-03-08
categories: PHP
---

`__autoload()` 用于自动加载资源来取代一大堆的 `require`  
在 PHP 的文档 [Autoloading Classes](http://php.net/manual/en/language.oop5.autoload.php) 中 首先建议的是使用 `spl_autoload_register()`, 并且描述到:

> Whilst the __autoload() function can also be used for autoloading classes and interfaces, its preferred to use the spl_autoload_register() function. This is because it is a more flexible alternative (enabling for any number of autoloaders to be specified in the application, such as in third party libraries). For this reason, using __autoload() is discouraged and it may be deprecated in the future.

`spl_autoload_register()` 的[文档](http://php.net/manual/en/function.spl-autoload-register.php)中提到:

>Register a function with the spl provided __autoload queue. If the queue is not yet activated it will be activated.

从这些说明可以看出, `spl_autoload_register()` 和 `__autoload()` 比较, 有如下好处:

* 可以灵活的指定某个函数为 "autoloaders"
* 可以指定多个 "loader", 使这些 "loader" 按顺序执行

从 [Autoloading Classes](http://php.net/manual/en/language.oop5.autoload.php) 的另外的 `Note`:

>Prior to PHP 5.3, exceptions thrown in the __autoload() function could not be caught in the catch block and would result in a fatal error. From PHP 5.3 and upwards, this is possible provided that if a custom exception is thrown, then the custom exception class is available. The __autoload() function may be used recursively to autoload the custom exception class.

和 它的 `Example #3`

```php
<?php
spl_autoload_register(function ($name) {
    echo "Want to load $name.\n";
    throw new Exception("Unable to load $name.");
});

try {
    $obj = new NonLoadableClass();
} catch (Exception $e) {
    echo $e->getMessage(), "\n";
}
// 输出如下:
// Want to load NonLoadableClass.
// Unable to load NonLoadableClass.
?>
```

可以看出 `spl_autoload_register()` 有更友好的错误处理能力

再配合上 `spl_autoload_unregister()` 就更加灵活了

参考:  

[Stackoverflow: someone explain spl_autoload, __autoload and spl_autoload_register?](http://stackoverflow.com/questions/7651509/someone-explain-spl-autoload-autoload-and-spl-autoload-register)  
[PHP SPL autoload vs. __autoload](https://dissectionbydavid.wordpress.com/2010/11/03/php-spl-autoload-vs-__autoload/)  
[SPL Autoload](http://www.phpro.org/tutorials/SPL-Autoload.html)

