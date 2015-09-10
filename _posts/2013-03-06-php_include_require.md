---
layout: post
title:  "php下的include和require"
date:   2013-03-06
categories: php
---

摘自[When should I use require_once vs include?](http://stackoverflow.com/questions/2418473/when-should-i-use-require-once-vs-include)  

1. When should I use require vs. include? [here](http://www.w3schools.com/php/php_includes.asp)

>The require() function is identical to include(), except that it handles errors differently. If an error occurs, the include() function generates a warning, but the script will continue execution. The require() generates a fatal error, and the script will stop.

2. When should I use require_once vs. require? [here](http://php.net/manual/en/function.require-once.php)

>The require_once() statement is identical to require() except PHP will check if the file has already been included, and if so, not include (require) it again.