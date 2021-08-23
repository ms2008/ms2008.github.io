---
layout:     post
title:      "Gzip 的一个坑"
subtitle:   "fileDescriptor embed"
date:       2021-08-23
author:     "ms2008"
header-img: "img/post-bg-binary.jpg"
catalog:    true
tags:
    - Golang
typora-root-url: ..
---

我们的项目里为了方便部署，swagger 文档是通过 gzip 压缩后，被植入到程序里的。其实这个思路源自于 [gRPC ProtoBuf fileDescriptor][1]

```golang
var fileDescriptor_308767df5ffe18af = []byte{
	// 2522 bytes of a gzipped FileDescriptorProto
	0x1f, 0x8b, 0x08, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0xff, 0xc4, 0x59, 0xcd, 0x6f, 0xdb, 0xc8,
	0x15, 0x5f, 0x7d, 0x5a, 0x7a, 0x92, 0x65, 0x7a, 0xec, 0x75, 0x18, 0xef, 0x47, 0x1c, 0xed, 0x66,
	0xe3, 0x24, 0xbb, 0xca, 0xc2, 0x49, 0x9c, 0xac, 0x53, 0x6c, 0x2b, 0x4b, 0x8c, 0x57, 0xa9, 0xbe,
```

每次大家更新完 swagger 文档后，都需要手动生成下这个 fileDescriptor，大概这样：

```sh
$ gzip -c swagger.yaml | xxd -p -c 16 | sed -e 's/../0x&,/g'
```

但是在多人协作的时候，总有个奇怪的问题：swagger 并没有更新，但是这个 fileDescriptor 有时却生成的不一样。并且每次都是文件头的第5-6个字节的位置：

![](/img/in-post/gzip-filedescriptor.png)

根据 gzip 的 [RFC 1952][2] 文档来看：

```
+---+---+---+---+---+---+---+---+---+---+
|ID1|ID2|CM |FLG|     MTIME     |XFL|OS | (more-->)
+---+---+---+---+---+---+---+---+---+---+
```

第5-8个字节为文档的修改时间（timestamp），使用小端模式编码（这也解释了为啥总是前两个字节变化）。动手来验证下：

```sh
$ gzip -c swagger.yaml | xxd -p -c 4 | sed -n '2p'
86672361
```

先转换为大端模式：`61236786`，再换算成10进制：`1629710214`，继续转换为本地时间：`2021-08-23 17:16:54`，发现和文档修改时间一致：

```
$ stat swagger.yaml
  File: ‘swagger.yaml’
  Size: 16434     	Blocks: 40         IO Block: 4096   regular file
Device: fd00h/64768d	Inode: 205340662   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-08-23 17:16:54.217753946 +0800
Modify: 2021-08-23 17:16:54.148753750 +0800
Change: 2021-08-23 17:16:54.148753750 +0800
 Birth: -
```

问题原因梳理明白了，解决就好办了，就像 gRPC 那样直接把这个时间戳置0就好了，好在 gzip 已经提供了这样一个选项：

```sh
$ gzip -n -c swagger.yaml | xxd -p -c 4 | sed -n '2p'
00000000
```

[1]: https://github.com/gogo/protobuf/blob/226206f39bd7276e88ec684ea0028c18ec2c91ae/protoc-gen-gogo/descriptor/descriptor.pb.go#L2705-L2709
[2]: https://datatracker.ietf.org/doc/html/rfc1952#page-5