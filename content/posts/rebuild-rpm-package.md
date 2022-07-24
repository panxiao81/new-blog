---
title: "在新版本系统上重新打包 RPM -- 又：在 RHEL 8 上安装 APT-Cacher-NG"
date: 2022-07-24T19:53:27+08:00
categories: [
    "Linux"
]
tags: [
    linux
]
draft: false
---

旧版 RHEL 能得到 RPM，但新版本系统不再提供了怎么办呢？

除非你使用的软件是专有软件，没有提供 SRPM，否则最好使用 SRPM 在新系统下重新打包。这样可以保证编译时使用新版本的编译器和动态链接库，以便避免一些各种依赖版本带来的问题。

甚至你可以在原包基础上升级软件版本。

<!--more-->

## TL:DR

我已经做好 YUM Repo 了，不需要你们自己打包了。

```sh
curl -OL https://s3.ddupan.top/repo/yum/panxiao81.repo
sudo mv panxiao81.repo /etc/yum.repos.d/
sudo dnf install apt-cacher-ng
```

## 前言

我在实验环境中经常反复装 Linux 系统，每次装完都要更新，安装各类软件包，显然我可以考虑搭建一个缓存服务。

Nexus 只做缓存使用有些过于臃肿，APT-Cacher-NG 是一个接近开箱即用的方案，当然也可以用 Nginx 做反向代理缓存，甚至使用 Squid 这种本来就是用来做缓存的方案。

APT-Cacher-NG 提供了 deb 包，Debian 源中也有源码包提供，EPEL 在 EL7 时代曾经维护过 APT-Cacher-NG 的 RPM，但或许是用的人比较少等原因，EPEL 8 以后不再维护这个包了，因此我打算把 EL7 的 SRPM 下载回来，在 EL8 上使用更新版本的上游源码重新编译。

## 下载和安装 SRPM

如果你有一个安装了 EPEL 7 源的 EL7 系系统，你大可以直接使用 yum 在系统上下载 SRPM，否则你需要去源里翻 SRPM 并下载回来。

我们可以从 EPEL 7 中得到 [apt-cacher-ng-3.1-4.el7.src.rpm](https://download-ib01.fedoraproject.org/pub/epel/7/SRPMS/Packages/a/apt-cacher-ng-3.1-4.el7.src.rpm)，实际是这就是一个普通的 RPM 包。

这个 RPM 包可以被安装，他将会被安装在用户家目录中。

```sh
rpm -i apt-cacher-ng-3.1-4.el7.src.rpm
```

现在你应当能在家目录中发现一个 `rpmbuild` 目录。

## 重新构建 RPM

进入到 `rpmbuild` 目录后，你可以发现这就是一个标准的 RPM 工作目录。

但为了不弄脏我自己的系统，我决定在容器中构建。

由于我的宿主系统是 RHEL，所以我也打算使用 Red Hat 的 UBI 镜像，也可以使用 Alma Linux 等 EL 系的发行版，步骤是一致的。

```sh
podman run -it --rm -v `pwd`:/root registry.access.redhat.com/ubi8/ubi:8.6
```

RPM 工作目录内部结构是这个样子

```sh
[panxiao81@nexus rpmbuild]$ ls
BUILD  BUILDROOT  RPMS  SOURCES  SPECS  SRPMS
```

其中，SOURCES 保存了所有用到的源码；SPECS 保存了构建 RPM 的描述 SPEC 文件；BUILD 目录保存源码的中间过程，因为存在对上游源码打 Patch 的可能；BUILDROOT 目录顾名思义，在构建过程完成后 BUILDROOT 保存了整个 RPM 的目录结构；RPMS 和 SRPMS 保存构建成品。

因此，进入 SPECS 目录，可以看到 `apt-cacher-ng.spec` 文件，这就是构建 RPM 所需的文件。

在构建 RPM 之前，首先安装 RPM 工具包

```sh
sudo dnf install rpm-build
```

如果不打算升级软件包直接开始构建的话，此时就可以直接构建了。

首先安装构建依赖，dnf 存在一个子命令可以直接从 SPEC 中安装所需依赖包。

```sh
dnf builddep apt-cacher-ng.spec
```

如果使用 YUM（如果使用 EL7 及更早的版本）则使用 `yum-builddep`

这样一来就可以构建 RPM 了。

```sh
cd SPECS/
rpmbuild -ba apt-cacher-ng.spec
```

构建完成如果没有错误的话，则会在 RPM 的工作目录中的 RPMS 子目录下生成 RPM 包，在 SRPMS 子目录中生成 SRPM 包。

但如果你想同时升级软件的话，你就需要修改 SPEC 文件，下载新的源码，甚至可能需要新增依赖包。

打开 SPEC 文件，你可以看到源码的上游地址，即 `http://ftp.debian.org/debian/pool/main/a/apt-cacher-ng/%{name}_%{version}.orig.tar.xz`，顺着这里，可以看到两个更新的版本，一个是随 Debian 10 分发的 3.2.1，另一个是随 Debian 11 分发的 3.6.4

3.6.4 更新的内容较多，引入了许多新的库，我选择编译 3.2.1 版本。

把新的源码包下载回来，放进 SOURCES 目录中，修改 SPEC 中的版本字段，构建 RPM 即可。

## 给 RPM 签名与 YUM 源

如果打算使用 YUM 源，则在建立源之前最好先给 RPM 签名，保证包在构建后不被第三者篡改。

首先保证 GPG 已安装，并且存在一个签名密钥，若还没有密钥，可先生成一个。

```sh
$ gpg --list-keys
gpg --list-keys
/home/panxiao81/.gnupg/pubring.kbx
-----------------------------------
pub   rsa4096 2022-06-18 [SC]
      E187C04BCB99AB1459DF82584191215A82DA49B3
uid             [ 绝对 ] Xiao Pan (panxiao81) <pan-xiao@live.cn>
sub   rsa4096 2022-06-18 [E]
```

接下来设置 RPM 全局宏，让 RPM 使用该 Key 对 RPM 签名

```sh
echo "%_signature gpg" >> ~/.rpmmacros
echo "%_gpg_name E187C04BCB99AB1459DF82584191215A82DA49B3" > /root/.rpmmacros
```

接下来对所有 RPM 附加签名信息。

安装 rpm-sign 工具

```sh
sudo dnf install rpm-sign
```

对所有 RPM 签名

```sh
rpm --addsign *.rpm
```

接下来可以建立 YUM 源了。

安装工具

```sh
sudo dnf install createrepo
```

建立目录结构：

```sh
tree repo
repo
├── 8
│   ├── source
│   │   ├── package
│   │   │   ├── apt-cacher-ng-3.1-4.el8.src.rpm
│   │   │   └── apt-cacher-ng-3.2.1-4.el8.src.rpm
│   │   └── repodata
│   │       ├── 274d597b12ab09dbacbf9399112ac0e7e979db30ad0ca6267c4b79b401b9d9d4-filelists.sqlite.bz2
│   │       ├── 4dfffd3378cd3bef128f9585cd6c2cbd516a13d04de9ce3a331712cb09b3510e-primary.xml.gz
│   │       ├── 5c82e87053b2bc80935e1b431e339b62091a237e351e24d7bae4304b682f850f-other.xml.gz
│   │       ├── 622fe6bdbafc84e6797aa0e6fe2d5919570287be7ebe00a09dcc9737755365c1-filelists.xml.gz
│   │       ├── 7e3a9659559cf0d479be53981f11a2c479ec18c7f497603101992a6867b7f00e-other.sqlite.bz2
│   │       ├── a7ccaf83e1745a09f9a99b3cb833a648537bec9b8bdebd13338fb7e4f57a999c-primary.sqlite.bz2
│   │       ├── repomd.xml
│   │       └── repomd.xml.asc
│   └── x86_64
│       ├── debug
│       │   ├── package
│       │   │   ├── apt-cacher-ng-debuginfo-3.1-4.el8.x86_64.rpm
│       │   │   ├── apt-cacher-ng-debuginfo-3.2.1-4.el8.x86_64.rpm
│       │   │   ├── apt-cacher-ng-debugsource-3.1-4.el8.x86_64.rpm
│       │   │   └── apt-cacher-ng-debugsource-3.2.1-4.el8.x86_64.rpm
│       │   └── repodata
│       │       ├── 09554566f83fd9349f5eea17b24caf195973f81a7e23e4db45765ab14125d250-filelists.sqlite.bz2
│       │       ├── 3c40475c856494cc17df8efce82dc117e7e71c0c7e1e42c8ae0e2c95864aaa1a-other.sqlite.bz2
│       │       ├── 57790698cbe2c008f5db27dd14f15f2d7a7ea2d906a77301be869990b136e093-filelists.xml.gz
│       │       ├── 8f2aefd4b3f823b2f157592737f7b9bb193c1dc3420e3d2eb0a25d8b5149ca53-other.xml.gz
│       │       ├── e329820d70a8c6984029f54c21ef65b622ee7e0c0436eb0a1891e827fbf9d54f-primary.sqlite.bz2
│       │       ├── eed10c75e12bbff9233b0effad7be8f4ff4651d99c5dcdc972b0cca1349b004d-primary.xml.gz
│       │       ├── repomd.xml
│       │       └── repomd.xml.asc
│       └── package
│           ├── apt-cacher-ng-3.1-4.el8.x86_64.rpm
│           ├── apt-cacher-ng-3.2.1-4.el8.x86_64.rpm
│           └── repodata
│               ├── 62fb19e348a50b864a4f4b6c7c4d4acc1516550dee71b8ca6137a0c3f35d34a6-filelists.xml.gz
│               ├── 914e6e4ee80ea374a17142378599047795a3d15fb9ab8bf6449eb172cf713d3c-primary.sqlite.bz2
│               ├── a073711eea3811831060a02033fc321b8923114304138e59d95417a7feffe83f-filelists.sqlite.bz2
│               ├── c16dc70163bb851cd2fde4b8154b8d81d38fb0f871ae5010e0a6ed8130fbe9fa-other.sqlite.bz2
│               ├── e9e4db2f26890d356b0a86aa4183fe63fccf7eb0da0f453b0fadbe707687cacf-primary.xml.gz
│               ├── ee8e730e72d142b48106aa1f3c8b63b11ad5bcfe0f0b650b3a2029aef91002f4-other.xml.gz
│               ├── repomd.xml
│               └── repomd.xml.asc
├── panxiao81.repo
└── RPM-GPG-KEY-PANXIAO81
```

以上供参考。

导出 GPG Public Key 供源使用者使用

```sh
gpg --export E187C04BCB99AB1459DF82584191215A82DA49B3 > RPM-GPG-KEY-PANXIAO81
```

最好将 Debuginfo 包，SRPM 包和二进制 RPM 包分到不同的源存放。

对所有的源进行 createrepo 操作

```sh
createrepo .
```

对 repodata 的 metadata 进行签名

```sh
gpg --detach-sign --armor repodata/repomd.xml
```

我选择将这个目录结构整体上传 S3 存储，也可以使用其他方法发布，为方便使用，可以提供一个预先写好的 repo 文件。

```ini
[panxiao81]
name=Xiao Pan Rebuild Package
baseurl=https://s3.ddupan.top/repo/yum/8/$basearch/package
enabled=1
gpgcheck=1
gpgkey=https://s3.ddupan.top/repo/RPM-GPG-KEY-PANXIAO81

[panxiao81-debug]
name=Xiao Pan Rebuild Package - Debug
baseurl=https://s3.ddupan.top/repo/yum/8/$basearch/debug
enabled=0
gpgcheck=1
gpgkey=https://s3.ddupan.top/repo/RPM-GPG-KEY-PANXIAO81

[panxiao81-source]
name=Xiao Pan Rebuild Package - Source
baseurl=https://s3.ddupan.top/repo/yum/8/source
enabled=0
gpgckeck=1
gpgkey=https://s3.ddupan.top/repo/RPM-GPG-KEY-PANXIAO81
```

以上完成后，可以换一台机器做测试。
