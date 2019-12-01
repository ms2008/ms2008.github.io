---
layout:     post
title:      "VS Code 快速查看 Golang 接口"
subtitle:   "工欲善其事，必先利其器"
date:       2019-12-02
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
    - VS Code
typora-root-url: ..
---

### 背景

使用 vscode 阅读 Go 项目源码时，有个不太方便的地方，就是跟踪 `interface` 的实现。vscode 只能追到 `interface` 定义的地方，而无法定位到其具体的实现。比如，我在追 etcd 关于 revision 的读取的时候只能追到这里：

![](/img/in-post/gopls-1.png)

如果项目比较小，还比较容易对付，因为按照习惯来讲，其实现往往都在对应接口的下方。但是遇到这种像 etcd 的项目就抓瞎了，因为其实现可能会跨越多个文件。好在 vscode 有个非常好用的功能：**Go to Implementation**

![](/img/in-post/gopls-2.png)

`Ctrl+F12` 就能找到实现了该 `interface` 的所有方法，然后再结合上下文，这样就很容易把调用关系都串下来。

vscode 之所以能够找到这些调用关系，依赖的是 Go 官方提供的代码导航工具：`guru`，它有几个缺点：

- 查找速度慢
- 不支持 Go Module
- 官方不再维护

### gopls

微软在开发 VS Code 过程中, 定义一种协议, 语言服务器协议：[Language Server Protocol][2]，用来统一不同语言的静态检测、自动补全问题。

`gopls` 就是 Go Team 目前正在积极维护的 lsp，有望成为 vscode Go 插件的默认补全工具。它最大的优点就是非常快，和 `guru` 相比有质的提升，同时还支持 Go Module。当然也少不了缺点：不支持 **Go to Implementation**（其实已经实现了，只是还没有发布）

如果你想现在就用上这个特性，可以有两个选择：

1. 自己编译 `master` 分支的 `gopls`
2. 使用 `bingo` 的 lsp（`bingo` 的作者参考了 `guru` 的实现单独 fork 了一个版本）

当然也可以用我目前的方案：

我的 Go 项目基本都会拷贝 vendor，所以并不希望开启 mod 支持。另外禁用 `gopls` 的 `goToTypeDefinition`、`goToImplementation` 选项，这样 vscode 就会继续用 `guru` 的实现。

> 此外，linter 工具我选择的是 `golangci-lint`，并没有使用官方的 `golint`，主要是因为后者烦人的「exported method should have comment or be unexported」建议，而前者还支持检测内存对齐，非常有用。

最后贴下我的完整配置：

```json
// For Golang
// "go.goroot": "C:\\go",
// "go.gopath": "${workspaceRoot}",
"go.useLanguageServer": true,
"go.inferGopath": true,
"go.buildOnSave": "off",
"go.lintTool": "golangci-lint",
"go.lintFlags": ["--disable-all"],
"go.vetFlags": [],
"go.autocompleteUnimportedPackages": true,
"go.gotoSymbol.includeImports": false,
"go.gotoSymbol.includeGoroot": false,
"go.useCodeSnippetsOnFunctionSuggest": true,
"go.formatTool": "goreturns",
"go.docsTool": "gogetdoc",
"[go]": {
    "editor.snippetSuggestions": "top",
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
        "source.organizeImports": true
    }
},
"go.toolsEnvVars": {
    "GO111MODULE": "off",
    "GOPROXY": "https://goproxy.cn,direct",
    "GOSUMDB": "gosum.io+ce6e7565+AY5qEHUk/qmHc5btzW45JVoENfazw8LielDsaI+lEbq6",
},
"go.languageServerExperimentalFeatures": {
    "format": true,
    "autoComplete": true,
    "rename": true,
    "goToDefinition": true,
    "hover": true,
    "signatureHelp": true,
    "goToTypeDefinition": false,
    "goToImplementation": false,
    "documentSymbols": true,
    "workspaceSymbols": true,
    "findReferences": true,
    "diagnostics": false,
    "completeUnimported": true,
    "watchFileChanges": true,
    "deepCompletion": true,
},
"go.languageServerFlags": [
    "-rpc.trace",
    "serve",
    "--debug=localhost:6060",
],
```

### 参考文献

- [Search for implementations doesn't work][1]
- [x/tools/gopls: support module-local implementation request][3]
- [Use gogetdoc instead of godef and godoc][4]

[1]: https://github.com/microsoft/vscode-go/issues/2823
[2]: https://microsoft.github.io/language-server-protocol/
[3]: https://github.com/golang/go/issues/32973
[4]: https://github.com/Microsoft/vscode-go/pull/622