---
layout: post
title:  "安利 map/reduce/filter"
date:   2017-04-21
categories: 函数式
---


#### 场景

我们要做一份东北大拌菜, 原料有: 白菜,青椒,洋葱,糖,醋,盐(简单点儿做), 流程如下

* 洗白菜/青椒/洋葱
* 切白菜/青椒/洋葱
* 切好的菜放一起, 把佐料糖/醋/盐放进去

用 python版的 map/reduce 来实现就是

```python
# -*- coding: utf8 -*-
#!/usr/bin/env python

print reduce(
    lambda x, y: '{},{}'.format(x, y), 
    map(
        lambda x: '切好的' + x, 
        map(
            lambda x: '洗好的' + x, 
            ['白菜', '青椒', '洋葱']
        )
    ), 
    reduce(
        lambda x, y: x + y, 
        ['糖', '醋', '盐']
    )
)

# output: 糖醋盐,切好的洗好的白菜,切好的洗好的青椒,切好的洗好的洋葱
```



经常会听到 Hadoop 里边的 MapReduce, 和这个指的是同一种思路: [MapReduce: Simplified Data Processing on Large Clusters ](https://research.google.com/archive/mapreduce.html)



#### Map

map 需要一个处理函数和一个待处理数组,  将处理函数作用于数组的每个item, 并将处理结果组成的新数组返回.

最前面例子里面的

```python
map(lambda x: '洗好的' + x, ['白菜', '青椒', '洋葱'])
```

基本上等价于

```python
def wash(x):
    return '洗好的' + x

washed = []
for item in ['白菜', '青椒', '洋葱']:
    washed.append(wash(item))

# 或: ['洗好的' + x for x in ['白菜', '青椒', '洋葱']]
```

在 Python 里, for…in 循环做的已经很好, 没有什么额外的副作用, 所以用 map 和用 for 循环都能很好的解决问题, 但是在某些语言里, 就不是这么简单了, 比如 php 里

```php
<?php

$arr = [1, 2, 3];
foreach ($arr as $item) {
    break;
}
echo current($arr);
// 输出 2, 说明 foreach 影响了数组内部的指针, 如果后面一些代码依赖于这个指针的话(比如使用 current 或者 next), foreach 的不确定位置的跳出, 就会影响后面代码的执行, 当然这种情况很少见, 这个例子只是为了说明"副作用" 而已
```

结论: 

* map的优点: 
  * map 实现的更直观, 能够很清楚的表达操作
  * 没有副作用
* 缺点:
  * 不能 break

这里给出 php 和 javascript 的示例代码, 来演示一下 map 的使用

```php
<?php

array_map(function($item) {
    return '洗好的'.$item;
}, ['白菜', '青椒', '洋葱']);
```

```javascript
['白菜', '青椒', '洋葱'].map(i=>`洗好的${i}`)
```



#### Reduce

reduce 需要:

* 一个处理函数接收上一次调用的结果(如果是第一次, 接收的是 reduce 的第三个参数: 初始值) 和待处理数组的 item
* 待处理数组
* 初始值(可以为空)

返回最后一次调用处理好的值

Python 文档中给出了对 reduce 大概实现, 可以用来理解 reduce 函数做的工作方式

```python
def reduce(function, iterable, initializer=None):
    it = iter(iterable)
    if initializer is None:
        try:
            initializer = next(it)
        except StopIteration:
            raise TypeError('reduce() of empty sequence with no initial value')
    accum_value = initializer
    for x in it:
        accum_value = function(accum_value, x)
    return accum_value
```

最前面例子里面的

```python
reduce(lambda x, y: x + y, ['糖', '醋', '盐'])
```

基本上等价于

```python
mix = ''
for item in ['糖', '醋', '盐']:
    mix = mix + item
# 当然, 也等价于:  ''.join(['糖', '醋', '盐']), 但不重要, 我们说的不是这个, 比如处理函数复杂了, 就不等价于这个了不是.
```

同样给出 php 和 javascript 的示例代码, 来演示一下 reduce 的使用

```php
<?php

array_reduce(['糖', '醋', '盐'], function($carry, $item) {
    return $carry.$item;
});
```

```javascript
['糖', '醋', '盐'].reduce((carry, item)=>carry+item)
```



#### Filter

filter 需要一个处理函数和一个待处理数组, 函数用于确定数组中的 item 是否要保留, 返回一个数组

前面例子没有说到, 其实就是过滤功能, 比如删除数组中是空字符串的元素

```python
filter(lambda x: x != '', ['糖', '', '醋', '盐', ''])
# ['糖', 醋', '盐']
```

基本上等价于

```python
filtered = []
for item in ['糖', '', '醋', '盐', '']:
    if item != '':
        filtered.append(item)

# 或:  [x for x in ['糖', '', '醋', '盐', ''] if x != '']
# Python 是个牛逼的语言呐....
```

同样给出 php 和 javascript 的示例代码, 来演示一下 reduce 的使用

```php
<?php

array_filter(['糖', '', '醋', '盐', ''], function($item) {
    return $item !== '';
});
```

```javascript
['糖', '', '醋', '盐', ''].filter(i=>i !== '')
```



#### 总结一下 map/reduce/filter

这三个函数有一些共同的特性:

* 着重描述了去干什么, 而不是让人(特别是读代码的人)把注意力放在怎么干上
* 没有使用循环体
* 没有副作用, 也就是说对原数组没有影响

在代码直观这一点上, 最开始那个例子稍微有点儿过, 但是这个取决于具体语言的描述方式, 如果用 javascript, 就会变成下面这样, 可读性升了一大截:

```javascript
['白菜', '青椒', '洋葱']
  .map(i=>`洗好的${i}`)
  .map(i=>`切好的${i}`)
  .reduce(
    (c, i)=>`${c},${i}`,
    ['糖', '醋', '盐'].reduce((c, i)=>c+i)
  )
```



#### 函数式编程

前面说了一大堆, 很多地方都能看到一点: 没有副作用, 这里就自然而然的引出函数式编程, 就因为没有副作用, 它有以下优点:

* 易于测试, 因为函数不依赖于外部状态, 所以很容易的就可以构造测试
* 并发

另外函数式编程还有一个概念: 高阶函数, 就是接收函数作为参数或者返回一个函数的函数, 前面说的 map/reduce/filter 都是高阶函数

* 接收函数作为参数, 前面已经描述的很多了

* 返回一个函数

  举个例子:

  ```javascript
  function addTow() {
    return function(val) {
      return 2 + val
    }
  }

  var fn = addTow()
  console.log(fn(3))
  // 5
  ```

* currying

  根据函数的参数来返回定制化的函数, 接上面的例子, 可以改造为

  ```javascript
  function add(base) {
    return function(val) {
      return base + val
    }
  }

  var add2 = add(2)
  console.log(add2(3))
  // 5
  var add3 = add(3)
  console.log(add3(3))
  // 6
  ```

  ​