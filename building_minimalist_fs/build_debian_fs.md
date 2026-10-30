> 本次使用方案B：使用官方debuerreotype (追求极致瘦身与确定性)
> 该方式通常在Docker容器中运行以保证环境纯净
```shell
rambo@ub2404:~$ mkdir test3 && cd test3
rambo@ub2404:~/test3$ git clone https://github.com/debuerreotype/debuerreotype.git
rambo@ub2404:~/test3$ cd debuerreotype

# 修改Dockerfile的第32和67行，即将这两行旧版的ENV语法改成带等号的新版语法即可
rambo@ub2404:~/test3/debuerreotype$ vim Dockerfile
# 旧写法
32 ENV WGETRC /.wgetrc
67 ENV DEBUERREOTYPE_DIRECTORY /opt/debuerreotype
# 新写法
32 ENV WGETRC=/.wgetrc
67 ENV DEBUERREOTYPE_DIRECTORY=/opt/debuerreotype

```


# 生成基础构建环境镜像
```shell
rambo@ub2404:~/test3/debuerreotype$ sudo docker build -t debuerreotype .

# 执行初始化引导
rambo@ub2404:~/test3/debuerreotype$ sudo docker run --rm -v /tmp/debian-output:/output debuerreotype debuerreotype-init /output/rootfs bookworm 2026-08-05

各项参数具体拆解如下：
sudo docker run：创建并启动一个新的容器实例
--rm：容器运行结束后自动销毁，避免在宿主机上残留无用的停止状态容器
-v /tmp/debian-output:/output：将宿主机的/tmp/debian-output目录挂载到容器内的/output目录
debuerreotype：指定要运行的Docker镜像名称
debuerreotype-init：容器内要执行的主命令(脚本)，底层封装了debootstrap，专门用于初始化Debian根文件系统
/output/rootfs：传给 debuerreotype-init 的第一个参数，指定根文件系统在容器内(即挂载点内)的生成路径(对应宿主机即/tmp/debian-output/rootfs)
bookworm：传给脚本的Debian版本代号(Debian 12)
2026-08-05：传给脚本的时间戳参数。debuerreotype的核心特性之一是支持"时间旅行"，它会利用Debian官方的快照源(Snapshot Archive)，拉取指定日期当天锁定的软件源状态，从而实现完全确定性、可复现的构建
Debian官方的快照服务器大约从2011年左右开始记录全球软件源的每天状态。因此有效日期可从2011年之后的任意一天都可以
无效日期：
未来的日期（比如明天、或者 2030 年）：快照源里没有记录，构建会直接报错退出
太古老的日期（比如 2000 年）：当时 Debian 还没做每日快照归档，也会失败

这个通常你不需要特意去查，因为这个日期的核心意义在于"锁定一个精确的历史版本以确保可复现"：
如果你想用最新鲜的软件包，直接写今天的日期即可
如果你想复现某个历史稳定状态(比如测试某个特定旧版本软件)，可直接访问Debian Snapshot官网(https://snapshot.debian.org/)，在页面上查看你想回溯的年份和日期
```

# 执行精简与优化
> (移除man手册、locale、配置apt源等)
```shell
# 1. 初始化
sudo docker run --rm --privileged -v /tmp/debian-output:/output debuerreotype debuerreotype-init /output/rootfs bookworm 2026-08-05
作用：基于debootstrap底层机制，拉取Debian12(bookworm)在指定快照日期 2026-08-05 的官方软件源，在容器内的/output/rootfs目录下构建出一个基础的Debian用户>态文件系统
大白话：
去Debian官方2026年8月5号的档案库里，把bookworm系统的底子拿出来，给我装进宿主机的 /tmp/debian-output/rootfs 目录里
前面那些 sudo docker run --rm --privileged -v... 纯粹是Docker的"后勤保障和安保措施"(权限放开、目录映射、完事删容器)，和你要构建什么系统、哪个版本毫无>关系

# 2. 精简配置
sudo docker run --rm --privileged -v /tmp/debian-output:/output debuerreotype debuerreotype-minimizing-config /output/rootfs
作用：对刚刚生成的/output/rootfs进行瘦身和容器化加固，就是给系统里的包管理器(APT)立个规矩：以后安装软件时，不准顺带安装任何"推荐但非必须"的附属软件包>，能省一点体积是一点
大白话：把这个毛坯房里所有占地方但没用的杂物全清出去，并立下规矩以后不准乱买杂物，让它瘦身到极致，变成一个精简轻量的小房子

# 3. 打包
sudo docker run --rm --privileged -v /tmp/debian-output:/output debuerreotype debuerreotype-tar /output/rootfs /output/debian-tarball.tar.gz
作用：把刚才处理好的那个根目录(/output/rootfs)打包压缩，做成一个标准的 tar.gz 压缩包，放到外面去(即宿主机的/tmp/debian-output/debian-tarball.tar.gz)
这个.tar.gz的包即可直接用于 FROM scratch 镜像构建

rambo@ub2404:~/test3/debuerreotype$ ls -lh /tmp/debian-output/debian-tarball.tar.gz
-rw-rw-r-- 1 root root 47M Aug  5 08:00 /tmp/debian-output/debian-tarball.tar.gz

```


# 在Docker中测试使用
```shell
rambo@ub2404:~/test3/debuerreotype$ cd ..
rambo@ub2404:~/test3$ cp /tmp/debian-output/debian-tarball.tar.gz .
rambo@ub2404:~/test3$ vim Dockerfile
FROM scratch
ADD debian-tarball.tar.gz  /
CMD ["/bin/bash"]

# 构建 & 使用
rambo@ub2404:~/test1$ sudo docker build -t debian:12 .
rambo@ub2404:~/test3$ sudo docker run --rm -it debian:12 cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"
VERSION_ID="12"
VERSION="12 (bookworm)"
VERSION_CODENAME=bookworm
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```
