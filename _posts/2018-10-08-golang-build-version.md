---
layout:     post
title:      "Golang -ldflags 的一个技巧"
subtitle:   "go version 信息注入"
date:       2018-10-08
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
---

我在开发 go 的项目时，习惯上会在编译后的二进制文件中注入 git 等版本信息后再发布。一直以来都是这么做的：

```go
package main

import (
	"fmt"
	"os"
	"runtime"
)

var buildstamp = ""
var githash = ""

func main() {
	args := os.Args
	if len(args) == 2 && (args[1] == "--version" || args[1] == "-v") {
		fmt.Printf("Git Commit Hash: %s\n", githash)
		fmt.Printf("UTC Build Time : %s\n", buildstamp)
		return
	}
}
```

然后编译的时候，通过链接选项 `-X` 来动态传入版本信息：

```sh
flags="-X main.buildstamp=`date -u '+%Y-%m-%d_%I:%M:%S%p'` -X main.githash=`git describe --long --dirty --abbrev=14`"
go build -ldflags "$flags" -x -o build-version main.go
```

这样编译后的程序就可以通过 `-v` 参数查看版本信息了

```
[root@KONG:bin (master)]# ./build-version -v
Git Commit Hash: 1.3-5-ge9edea3e8f986d
UTC Build Time : 2018-09-30_03:40:36AM
[root@KONG:bin (master)]#
```

然而在多人协作的项目时，大家的 go 版本可能会不同，所以我又尝试将 `go version` 信息植入到程序中。

```go
package main

import (
	"fmt"
	"os"
	"runtime"
)

var buildstamp = ""
var githash = ""
var goversion = ""

func main() {
	args := os.Args
	if len(args) == 2 && (args[1] == "--version" || args[1] == "-v") {
		fmt.Printf("Git Commit Hash: %s\n", githash)
		fmt.Printf("UTC Build Time : %s\n", buildstamp)
		fmt.Printf("Golang Version : %s\n", goversion)
		return
	}
}
```

通过添加编译参数：`-X main.goversion=$(go version)`，然而却失败了。通过错误输出可以判断是因为 `go version` 的输出有空格，导致编译被打断。之后我又改了下编译参数，添加了单引号：`-X main.goversion='$(go version)'`，遗憾的是，编译依然无法通过。

记得之前，我看到过 [codis][1] 的源码是通过一个 shell 脚本生成一个额外的 go package 来处理的：

```sh
version=`git log --date=iso --pretty=format:"%cd @%H" -1`
if [ $? -ne 0 ]; then
    version="unknown version"
fi

compile=`date +"%F %T %z"`" by "`go version`
if [ $? -ne 0 ]; then
    compile="unknown datetime"
fi

describe=`git describe --tags 2>/dev/null`
if [ $? -eq 0 ]; then
    version="${version} @${describe}"
fi

cat << EOF | gofmt > pkg/utils/version.go
package utils
const (
    Version = "$version"
    Compile = "$compile"
)
EOF

cat << EOF > bin/version
version = $version
compile = $compile
EOF
```

但是这样，每开启一个新项目都需要额外的 shell 脚本，也够麻烦的。所以，很长时间我都是这么来偷懒的：


```go
package main

import (
	"fmt"
	"os"
	"runtime"
)

var buildstamp = ""
var githash = ""
var goversion = fmt.Sprintf("%s %s/%s", runtime.Version(), runtime.GOOS, runtime.GOARCH)

func main() {
	args := os.Args
	if len(args) == 2 && (args[1] == "--version" || args[1] == "-v") {
		fmt.Printf("Git Commit Hash: %s\n", githash)
		fmt.Printf("UTC Build Time : %s\n", buildstamp)
		fmt.Printf("Golang Version : %s\n", goversion)
		return
	}
}
```

但是这样的话，也有一个问题：**就是在交叉编译的时候无法正确反应出 go 的版本**。比如，你是在 OSX 下编译 linux 的可执行程序，这时候你通过 `-v` 参数查看显示的也是 linux 平台，而不是期待的 darwin 平台。

不过，近日我在闲逛 go nuts 时，看到一个贴子：[v1.5 -ldflags -X change breaks when string has a space][2]，谈到了这个技巧：**如果要赋值的变量包含空格，需要用引号将 -X 后面的变量和值都扩起来**。原来如此，再次 build 一下，还真好用：

```sh
flags="-X 'main.goversion=$(go version)'"
go build -ldflags "$flags" -x -o build-version main.go
```

子曰：三人行，必有我师焉。看来，社区还得逛起来呀~~

[1]: https://github.com/CodisLabs/codis
[2]: https://groups.google.com/forum/#!topic/golang-nuts/aNDB4FrmEiA
