由 [此](https://github.com/alpinelinux/docker-alpine/blob/410e6fcf56b6bfd6e6b3acf0ab59a5ad1381ca54/x86_64/Dockerfile) 产生的想深挖一下背后的东西

# 前奏
```shell

FROM scratch
ADD alpine-minirootfs-20260805-x86_64.tar.gz /
CMD ["/bin/sh"]

# 释义
在Docker中，FROM scratch代表一个绝对空白的镜像。直接基于它无法运行任何程序或执行shell

alpine-minirootfs-20260805-x86_64.tar.gz是Alpine Linux官方发布的极简根文件系统(Mini Root Filesystem)
核心内容：包含Linux系统的标准目录结构（如/bin、/etc、/lib、/proc、/sys等）、由BusyBox提供的基础命令行工具集，以及Alpine独有的轻量级包管理器apk
这个tar.gz包就是向空白镜像中"注入"的最基础Linux用户态环境。包名中的2026-08-05（无空格）代表构建日期，x86_64代表硬件架构


如何通过其他方式获取类似的 tar.gz 包
除了Alpine，主流Linux发行版都官方提供类似的“Bootstrap”或“Base”根文件系统压缩包，用于容器构建或系统引导：
Alpine官方渠道：直接访问Alpine Linux官网的下载页面获取不同版本和架构的Mini Root Filesystem
Arch Linux：Arch Linux官网提供的archlinux-bootstrap压缩包
```
Ubuntu Base：Ubuntu官方提供的 [Ubuntu Base tarball](https://cdimage.ubuntu.com/ubuntu-base/)，适合需要apt生态的场景

Debian：Debian官方维护的 [Debuerreotype](https://github.com/debuerreotype/debuerreotype) 工具生成的精简tarball



# 自己制作一个这样的 tar.gz 包
> 该阶段仅限于说明个大概，详情需要看本仓库中具体的文件系统构建文档
## 构建Alpine根文件系统
```shell
完全可自己从零构建一个。以在Linux宿主机上制作定制版的Alpine根文件系统为例，操作步骤如下：

步骤1：准备工具与目录
在宿主机上准备一个工作目录，并获取Alpine的包管理工具apk。如果你当前使用的是支持apk的系统(或通过下载静态编译的apk二进制文件)，可直接进行引导安装

步骤2：初始化根目录并安装核心组件
在指定目录下初始化一个带有alpine-base(包含核心工具和apk)的独立根文件系统：
# 创建目标根目录
mkdir -p /tmp/my-rootfs

# 初始化并安装基础包
apk --root /tmp/my-rootfs --initdb add alpine-base


步骤3：按需裁剪与配置
进入 /tmp/my-rootfs 目录，你可以对这个环境进行任意定制：
清理无用的缓存文件：rm -rf /tmp/my-rootfs/var/cache/apk/*
预置自定义的系统配置文件（如 /etc/resolv.conf、/etc/hosts、本地用户等）

步骤4：打包为 tar.gz
将定制好的根目录打包，这就得到了你自己的minirootfs：
cd /tmp/my-rootfs
tar -czf /tmp/my-custom-minirootfs-2026-08-05-x86_64.tar.gz .

打包完成后，即可将其放入Dockerfile中直接使用：
FROM scratch
ADD my-custom-minirootfs-2026-08-05-x86_64.tar.gz /
CMD ["/bin/sh"]

```


## 构建Ubuntu根文件系统
> Ubuntu和Debian均基于apt生态，其官方根文件系统（Rootfs）的核心构建工具是 debootstrap。Debian官方在此基础上进一步开发了自动化确定性构建工具 debuerreotype
```shell
1. Ubuntu Base 的构建方法
Ubuntu官方使用内部流水线脚本，但其核心原理和外部复现方式完全依赖 debootstrap。

构建步骤：
安装依赖工具（在Ubuntu或Debian宿主机上）：
sudo apt install debootstrap arch-install-scripts

执行 debootstrap 引导安装：
以生成Ubuntu 22.04（Jammy）极简根文件系统为例：
sudo debootstrap --arch=amd64 --variant=minbase jammy /tmp/ubuntu-base http://archive.ubuntu.com/ubuntu/

--variant=minbase：仅安装构建基础系统所必须的最小包集合（如apt、dpkg等），剔除文档和非必要组件

清理与裁剪：
进入 /tmp/ubuntu-base 目录，清理残留的apt缓存和日志：
sudo rm -rf /tmp/ubuntu-base/var/cache/apt/* /tmp/ubuntu-base/var/lib/apt/lists/*

打包为 tar.gz：
sudo tar -czf ubuntu-base-jammy-amd64.tar.gz -C /tmp/ubuntu-base .

```



## 构建Debian根文件系统
> Debian官方镜像以及Docker官方基础镜像不再直接手跑 debootstrap，而是使用由Docker和Debian团队共同维护的专有工具 debuerreotype。该工具能够实现完全可复现（Reproducible）的构建
```shell
方案A：直接使用 debootstrap（快速轻量）
与Ubuntu类似，Debian也可以直接通过 debootstrap 生成基础包：
sudo debootstrap --arch=amd64 --variant=minbase bookworm /tmp/debian-base http://deb.debian.org/debian/
sudo rm -rf /tmp/debian-base/var/cache/apt/* /tmp/debian-base/var/lib/apt/lists/*
sudo tar -czf debian-base-bookworm-amd64.tar.gz -C /tmp/debian-base .



方案B：使用官方 debuerreotype（追求极致瘦身与确定性）
如果你需要制作完全对标Debian官方品质的镜像，需使用 debuerreotype：
获取工具：
git clone https://github.com/debuerreotype/debuerreotype.git
cd debuerreotype
构建过程：
debuerreotype 通常在Docker容器中运行以保证环境纯净：
# 生成基础构建环境镜像
docker build -t debuerreotype .

# 执行初始化引导
docker run --rm -v /tmp/debian-output:/output debuerreotype debuerreotype-init /output/rootfs bookworm 2026-08-05

# 执行精简与优化（移除man手册、locale、配置apt源等）
docker run --rm -v /tmp/debian-output:/output debuerreotype debuerreotype-common /output/rootfs
docker run --rm -v /tmp/debian-output:/output debuerreotype debuerreotype-tarball /output/rootfs /output/debian-tarball.tar.gz



```

