---
layout: post
title:  "Go中的defer"
date:   2015-03-02
categories: Go
---

* defer用于注册延迟调用, 在外围函数return之前调用  
* 使用多个defer的话,这些defer按照注册时的FILO的顺序执行(可以想象成栈)  
* 多个defer依次执行,即使中间有panic. panic会在外围函数结束时捕获,向外扩散
* defer的参数在注册时求值和复制
  ```golang
  package main

  import "fmt"

  func main() {
      i := 10;
      defer fmt.Println(i)
      i = i + 1
      fmt.Println(i)
  }
  ```
  输出为
  ```
  11
  10
  ```
  另一个例子
  ```
  package main
  
  import "fmt"
  
  func end(line string) {
      fmt.Printf("%s\n", line)
  }
  
  func main() {
      line := "first"
      defer end(line)
      line = "second"
      fmt.Printf("%s\n", line)
  }
  ```
  输出为
  ``` js
  second
  first
  ```
* 用指针和闭包可以避免上面的情况,达到延迟读取的效果
  ```
  package main
  
  import "fmt"
  
  func main() {
      line := "first"
      defer func() {
          fmt.Printf("%s\n", line)
      }()
      line = "second"
      fmt.Printf("%s\n", line)
  }
  ```
  输出为
  ``` js
  second
  second
  ```
