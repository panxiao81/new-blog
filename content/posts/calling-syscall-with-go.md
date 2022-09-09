---
title:  在 Go 中调用 Syscall 的正确打开方式 - 以创建 Tun 接口为例
date: 2022-09-10T00:56:15+08:00
categories: [Go，Programming，Linux]
tags: [golang，go，programming，linux]
---


最近在做一个工具，估计很快就能和大家见面了。

这是我第一次用 Go 写 CLI 工具，功能意外的还蛮多蛮复杂，其中涉及到一些 Linux 运维的工作，包括对网络接口进行操作，对 IPtables 的维护等。

不可避免的，这个过程中需要调用 Syscall，Go 的 Syscall 不像 C 那样直接，虽然 Go 提供了 golang.org/x/sys/unix 包，其中打包了一些常用的 Syscall，但就我们的案例而言需要去调用裸 Syscall，因此研究了一下 Go 调用 Syscall 的正确打开方式。

<!--more-->

## 案例分析

在这个需求中，我需要创建一个 Tun 接口，尽管有一些第三方库包装了 Tun/Tap 接口，但就当前的早期版本而言，我需要让程序创建一个持久化的 Tun 接口并退出，只能自己实现 Syscall 包装。

首先，先来看对于 C 语言实现的 Tun 接口创建过程：

```c
#include <sys/types.h>
#include <linux/if.h>
#include <linux/if_tun.h>
#include <fcntl.h>
#include <errno.h>
#define TUNDEV "/dev/net/tun"

static int tap_add_ioctl(struct ifreq *ifr, uid_t uid, gid_t gid)
{
    int fd;
    int ret = -1;

#ifdef IFF_TUN_EXCL
    ifr->ifr_flags |= IFF_TUN_EXCL;
#endif

    fd = open(TUNDEV, O_RDWR);
    if (fd < 0) {
        perror("open");
        return -1;
    }
    if (ioctl(fd, TUNSETIFF, ifr)) {
        perror("ioctl(TUNSETIFF)");
        goto out;
    }
    if (uid != -1 && ioctl(fd, TUNSETOWNER, uid)) {
        perror("ioctl(TUNSETOWNER)");
        goto out;
    }
    if (gid != -1 && ioctl(fd, TUNSETGROUP, gid)) {
        perror("ioctl(TUNSETGROUP)");
        goto out;
    }
    if (ioctl(fd, TUNSETPERSIST, 1)) {
        perror("ioctl(TUNSETPERSIST)");
        goto out;
    }
    ret = 0;
out:
    close(fd);
    return ret;
}
```

这一段代码截取自 [iproute2/ip/iptuntap.c](https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/tree/ip/iptuntap.c)，他做了如下几件事情：

- 打开 `/dev/net/tun` 设备
- 对 Tun 设备调用 ioctl() Syscall 设置参数，这个部分可以参考内核主线文档中的 [TUN/TAP 设备](https://www.kernel.org/doc/html/latest/networking/tuntap.html)，而对于 ifreq 这个结构体，则可以查看 netdevice(7) 文档
- 对创建的设备设置参数，如设备所有者，设备组，设备持久化等
- 关闭文件

删除一个持久化的设备只需要将持久化位重新置为 0 即可。

## Go 调用 Syscall

Go 标准库中有 syscall 库，但文档有这样一句话：

> The syscall package is locked down. Callers should use the corresponding package in the golang.org/x/sys repository instead. That is also where updates required by new systems or versions should be applied. See https://golang.org/s/go1.4-syscall for more information.

即，syscall 标准库不被鼓励使用。

这样一来，按照他的说法，可以使用新的 golang.org/x/sys 库，这个库包装了不同系统的 Syscall，我们需要使用的是 Unix 的版本。

这个库包装了许多 Syscall 可以不需要使用 `unix.Syscall()` 直接调用，虽然没有提供 Tun/Tap 接口相关的包装，但我们可以最大化利用他已有的方法。

这样一来我们得到了如下的代码：

```go
package main

import (
    "fmt"
    "os"
    "syscall"
    "unsafe"

    "golang.org/x/sys/unix"
)

func NewTunDevice(n string) error {
	f, err := os.OpenFile("/dev/net/tun", os.O_RDWR, 0)
	if err != nil {
		return err
	}
	defer f.Close()

	// Create a new TUN device
	ifr, err := unix.NewIfreq(n)
	if err != nil {
		return err
	}

	// ifr.flags = IFF_TUN
	ifr.SetUint16(unix.IFF_TUN)

	_, _, errno := unix.Syscall(unix.SYS_IOCTL, f.Fd(), uintptr(unix.TUNSETIFF), uintptr(unsafe.Pointer(ifr)))
	if errno != 0 {
		return fmt.Errorf(errno.Error())
	}
	// Set Tun Interface Persistent
	_, _, errno = unix.Syscall(unix.SYS_IOCTL, f.Fd(), uintptr(unix.TUNSETPERSIST), uintptr(1))
	if errno != 0 {
		return fmt.Errorf(errno.Error())
	}
	return nil
}
```

这段代码的功能是显而易见的。

ifreq 结构体的第二个元素是一个 union，ifr_flags 元素是一个 Short 的变量，因此调用 ifr.SetUint16() 方法设置 ifr_flags 标志位。

对于这个 Syscall 调用，ioctl 没有有意义的返回值，因此直接忽略，只取 errno 即可。

unix.Syscall() 的 errno 返回一个 syscall.Errno 类型的值，他实际上是一个 uintptr，但我们可以直接用它和 0 比较，此处和 C 风格错误处理一致。

这个重新定义的 syscall.Errno 提供了 Error() 方法，他返回一个字符串，可用于 fmt.Errorf() 用来返回 Go 风格的 error。

对 Tun/Tap 的 Syscall 需要 CAP_NET_ADMIN 权限，因此需要增加权限或以 root 身份运行程序。

## 参考

https://pkg.go.dev/golang.org/x/sys@v0.0.0-20220908164124-27713097b956/unix

https://www.kernel.org/doc/html/v5.8/networking/tuntap.html

https://git.kernel.org/pub/scm/network/iproute2/iproute2.git/tree/ip/iptuntap.c

https://github.com/getlantern/tuntap/blob/1cde8300bb3754e63386e8c79b544b9e237a79d8/tun_linux.go#L9
