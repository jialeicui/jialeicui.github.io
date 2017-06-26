---
layout: post
title:  "在 Go 语言项目中支持插件式开发"
date:   2017-06-26
categories: Go
---

需求:

* 项目需要支持多种协议, 每种协议地位平等
* 每种协议的实现比较独立, 想增加随时新增个目录就可以
* 增删插件时动态加载不是刚需, 就是说重新编译也是可以的

### 使用动态链接库模式

核心依赖  https://golang.org/pkg/plugin/

#### 开发方式

* 在指定目录放置插件, 比如 plugin
* 每个插件编译成动态链接库的方式
* 插件实现用 interface 约束好
* 在主程序中遍历插件目录加载动态链接库, 加载插件
* 在需要的地方调用插件提供的方法

#### 目录结构

```sh
├── Makefile
├── main.go
└── src
    ├── base
    │   └── base.go
    └── plugins
        └── foo
            ├── Makefile
            └── foo.go
```

#### Makefile

* 主 Makefile 需调用每个插件的 make 生成 *.so
* 再 build 主程序

主 Makefile 示例

```Makefile
TOPTARGETS := all clean

# 遍历插件目录
PLUGIN_DIR=src/plugins/*
SUBDIRS := $(wildcard ${PLUGIN_DIR})

$(TOPTARGETS): $(SUBDIRS)
$(SUBDIRS):
	$(MAKE) -C $@ $(MAKECMDGOALS)

.PHONY: $(TOPTARGETS) $(SUBDIRS)
```

插件 Makefile 示例

```Makefile
all:
	go build -buildmode=plugin **.go
```

遗憾的是目前 macOS 下还不支持, 所以放弃这个方式

```sh
/Applications/Xcode.app/Contents/Developer/usr/bin/make -C src/plugins/foo all
go build -buildmode=shared
-buildmode=plugin not supported on darwin/amd64
make[1]: *** [all] Error 1
make: *** [src/plugins/foo] Error 2
```



### The Hacky Way ( Makefile里动入口文件 )

核心思路: 用模块的 init 入口调用主程序的统一注册函数

需要解决的痛点: 保证模块被 import (否则模块的 init 函数就没有被调用机会)

#### 基本结构 (目录结构同上)

base.go

```go
package base

import (
	"fmt"
)

type Plugin interface {
	Run()
}

var AllPlugins []Plugin

func Regist(p Plugin) {
	AllPlugins = append(AllPlugins, p)
}

func Start() {
	fmt.Println("Start")
	for _, p := range AllPlugins {
		p.Run()
	}
}
```

foo.go

```go
package foo

import (
	"../../base"
	"fmt"
)

type plugin struct {
}

func init() {
	p := plugin{}
	base.Regist(p)
}

func (p plugin) Run() {
	fmt.Println("foo run")
}
```

main.go

```go
package main

import (
	"./src/base"
)

func main() {
	base.Start()

}
```

#### 动入口文件的工具

 ```go
package main

import (
	"go/token"
	_ "../src/plugins/foo"
	"go/parser"
	"fmt"
	"flag"
	"bytes"
	"go/printer"
	"io/ioutil"
	"golang.org/x/tools/go/ast/astutil"
)

var (
	entrance   *string
	pluginPath *string
	to         *string
)

func main() {
	entrance = flag.String("entrance", "", "entance file")
	pluginPath = flag.String("pluginPath", "", "plugin dir")
	to = flag.String("to", "", "out put file")
	flag.Parse()

	if *entrance == "" || *pluginPath == "" || *to == "" {
		return
	}

	fset := token.NewFileSet()
	f, err := parser.ParseFile(fset, *entrance, nil, parser.Mode(0))
	if err != nil {
		fmt.Println(err)
	}

	files, _ := ioutil.ReadDir(*pluginPath)
	for _, file := range files {
		astutil.AddNamedImport(fset, f, "_", fmt.Sprintf("%s/%s", *pluginPath, file.Name()))
	}

	printerMode := printer.UseSpaces
	printConfig := &printer.Config{Mode: printerMode, Tabwidth: 4}

	var buf bytes.Buffer
	err = printConfig.Fprint(&buf, fset, f)
	if err != nil {
		return
	}
	out := buf.Bytes()

	ioutil.WriteFile(*to, out, 0644)
}
 ```

```Makefile
APPNAME=plugin_demo
ENTTMP=tmpmain.go

all:
	./tool/tool -entrance=main.go -pluginPath=./src/plugins -to=${ENTTMP}
	go build -o ${APPNAME} ${ENTTMP}
	rm ${ENTTMP}

.PHONY: all
```

