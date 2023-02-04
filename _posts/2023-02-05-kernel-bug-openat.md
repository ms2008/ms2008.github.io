---
layout:     post
title:      "ln 强制覆盖 symlink 失败问题研究"
subtitle:   "浅踩 kernel bug"
date:       2023-02-05
author:     "ms2008"
header-img: "img/post-bg-alitrip.jpg"
catalog:    true
tags:
    - C
typora-root-url: ..
---

最近公司 CI 升级，将 docker 基镜像由原先的 debian 切换到了 ubuntu，导致应用一旦成功启动之后，再次执行重启将会持续失败。查看日志，发现打印 `ln: failed to access '/tmp/access.log/stdout': Not a directory`

看来是 `ln` 执行失败，导致 docker entrypoint 无法执行成功，所以一直 restarting，查看其 `entrypoint.sh` 检查 `ln` 相关逻辑：`ln -sf /dev/stdout /tmp/access.log`

似乎并没有问题，那为啥后面几次执行会把 `access.log` 识别为一个目录呢？奇怪的是，debian 镜像就没有这个问题：

```sh
$ docker run -it --rm debian:10 bash
> ln -s /dev/stdout /tmp/access.log
> ln -s /dev/stdout /tmp/access.log
> exit
$ docker run -it --rm ubuntu:22.04 bash
> ln -s /dev/stdout /tmp/access.log
> ln -s /dev/stdout /tmp/access.log
ln: failed to access '/tmp/access.log/stdout': Not a directory
```

更为魔幻的是，同事们纷纷表示本地无法复现，只有我的开发环境有这个问题。无奈，只能寄希望于 `strace` 找到一些蛛丝马迹：

```sh
$ docker run --privileged -it --rm ubuntu:22.04 bash
> apt-get update && apt-get install strace
> ln -sf /dev/stdout /tmp/access.log
> strace ln -sf /dev/stdout /tmp/access.log
...
symlinkat("/dev/stdout", AT_FDCWD, "/tmp/access.log") = -1 EEXIST (File exists)
openat(AT_FDCWD, "/tmp/access.log", O_RDONLY|O_PATH|O_DIRECTORY) = 3
symlinkat("/dev/stdout", 3, "stdout")   = -1 ENOTDIR (Not a directory)
newfstatat(3, "stdout", 0x7fffcaf03a90, AT_SYMLINK_NOFOLLOW) = -1 ENOTDIR (Not a directory)
write(2, "ln: ", 4ln: )
...
```

有意思的地方出现了，`openat(AT_FDCWD, "/tmp/access.log", O_RDONLY|O_PATH|O_DIRECTORY) = 3` 表示将 `/tmp/access.log` 按照目录打开。理论上应该返回 `-1` 才对，但我这里却返回了 `3` 表示可以成功打开，也就是当真把 `/tmp/access.log` 识别成了一个目录。但是在同事们的环境中，却真真实实的返回了 `openat(AT_FDCWD, "/tmp/access.log", O_RDONLY|O_PATH|O_DIRECTORY) = -1 ENOTDIR (Not a directory)`

经询问，大家使用的内核都是 5.x，而只有我的环境用的是 3.10 😅。考虑到 `ln` 版本之间可能也会存在差异，所以准备用一段程序再次进行验证：

```sh
$ cat openat.c
#define _GNU_SOURCE
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(void) {

    int fd = openat(AT_FDCWD, "/dev/stdout", O_RDONLY|O_PATH|O_DIRECTORY);
    if (fd == -1) {
        perror("openat");
        return 1;
    }

    return 0;
}
$ cc openat.c
$ ltrace ./a.out
__libc_start_main([ "./a.out" ] <unfinished ...>
openat(0xffffff9c, 0x402010, 0x210000, 0x401190)      = 3
+++ exited (status 0) +++
```

可以看到，在我的宿主机还是返回了 `3`。但是到这里，还不能确定是 libc 的问题; 还是内核的问题：

```
Command-line utility -> glibc -> system call
```

接下来，有两个思路：

1. 静态链接 libc 之后让其在高内核机器上执行
2. 直接裸调 syscall，不调 libc 的包装函数

	```c
	#define _GNU_SOURCE
	#include <fcntl.h>
	#include <unistd.h>
	#include <sys/syscall.h>

	int main(void) {

		syscall(SYS_openat, AT_FDCWD, "/dev/stdout", O_RDONLY|O_PATH|O_DIRECTORY);
		return 0;
	}
	```

测试后，确认根因就是内核问题。同时又在 [stackoverflow][1] 上询问了下大家 ，一位老哥给贴出了个 [commit][2]。

看来是 4.2 内核以下，应该都有这个问题，手上有环境的同学可以试试。

等下，别走！还有一个疑问，那为啥 debian 下就没有问题？

答：debian 和 ubuntu 的 `ln` 版本不同，实现不一样，不依赖 `openat()`:

```
stat("/tmp/access.log", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
lstat("/tmp/access.log", {st_mode=S_IFLNK|0777, st_size=11, ...}) = 0
stat("/dev/stdout", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
symlink("/dev/stdout", "/tmp/access.log") = -1 EEXIST (File exists)
unlink("/tmp/access.log")               = 0
symlink("/dev/stdout", "/tmp/access.log") = 0
lseek(0, 0, SEEK_CUR)                   = -1 ESPIPE (Illegal seek)
```

[1]: https://stackoverflow.com/questions/75305383/openat-recognized-dev-stdout-as-a-directory
[2]: https://github.com/torvalds/linux/commit/97242f99a013950af63effa0732f8ef7db4e31ec
