# 前奏

```shell
使用上述说到的方法二：下载静态编译版或通过容器提取(适合制作Rootfs,本次使用该方法)
如果你像前面探讨的那样，需要在 Ubuntu 宿主机上通过 apk 命令去 --root 初始化一个Alpine根文件系统，直接用Ubuntu自带的apk-tools可能会遇到路径或架构兼容问题
最稳妥的方式是直接获取Alpine官方编译好的静态二进制文件，或者通过 Docker容器来运行它
```


# 构建 & 使用
> 用于脚本自动化更合适
```shell
# 1. 创建并进入工具存放目录
rambo@ub2404:~$ mkdir -p /tmp/apk-tool && cd /tmp/apk-tool

# 2. 从Alpine官方镜像站下载最新静态编译的apk二进制文件（以当前架构x86_64为例）
rambo@ub2404:/tmp/apk-tool$ wget https://dl-cdn.alpinelinux.org/alpine/v3.24/main/x86_64/apk-tools-static-3.0.8-r0.apk
rambo@ub2404:/tmp/apk-tool$ tar -xzf apk-tools-static-3.0.8-r0.apk
rambo@ub2404:/tmp/apk-tool$ ./sbin/apk.static --version         # 必要时手动加x权限
apk-tools 3.0.8-r0, compiled for x86_64.

rambo@ub2404:/tmp/apk-tool$ sudo cp ./sbin/apk.static  /usr/local/bin/

# 可用这个 apk.static 作为工具，配合 --root 参数去从零构建v3.24极简根文件系统
rambo@ub2404:/tmp/apk-tool$ sudo ./sbin/apk.static \
  --root /tmp/alpine-v3.24-rootfs \
  --repository https://dl-cdn.alpinelinux.org/alpine/v3.24/main \
  --repository https://dl-cdn.alpinelinux.org/alpine/v3.24/community \
  --allow-untrusted \
  --initdb add alpine-base

# 参数释义
--repository：指定 v3.24 的 main 和 community 官方源，解决"no such package"的根本原因
--allow-untrusted：新根目录下还没有官方公钥（etc/apk/keys/为空），必须临时允许不验证签名，否则会因为没有密钥直接中断

# 以下是回显
( 1/27) Installing alpine-baselayout-data (3.7.2-r1)
( 2/27) Installing musl (1.2.6-r3)
( 3/27) Installing busybox (1.37.0-r31)
  Executing busybox-1.37.0-r31.post-install
( 4/27) Installing busybox-binsh (1.37.0-r31)
( 5/27) Installing alpine-baselayout (3.7.2-r1)
  Executing alpine-baselayout-3.7.2-r1.pre-install
  Executing alpine-baselayout-3.7.2-r1.post-install
( 6/27) Installing bridge (1.5-r5)
( 7/27) Installing ifupdown-ng (0.13.0-r0)
( 8/27) Installing openrc-user (0.63.2-r0)
( 9/27) Installing libcap2 (2.78-r0)
(10/27) Installing openrc (0.63.2-r0)
  Executing openrc-0.63.2-r0.post-install
(11/27) Installing mdev-conf (4.10-r0)
(12/27) Installing busybox-mdev-openrc (1.37.0-r31)
(13/27) Installing alpine-conf (3.22.0-r0)
(14/27) Installing alpine-keys (2.6-r0)
(15/27) Installing alpine-release (3.24.3-r0)
(16/27) Installing libcrypto3 (3.5.9-r0)
(17/27) Installing libssl3 (3.5.9-r0)
(18/27) Installing ssl_client (1.37.0-r31)
(19/27) Installing zlib (1.3.2-r1)
(20/27) Installing libapk (3.0.8-r0)
(21/27) Installing ca-certificates-bundle (20261020-r0)
(22/27) Installing apk-tools (3.0.8-r0)
(23/27) Installing busybox-openrc (1.37.0-r31)
(24/27) Installing busybox-suid (1.37.0-r31)
(25/27) Installing scanelf (1.3.9-r1)
(26/27) Installing musl-utils (1.2.6-r3)
(27/27) Installing alpine-base (3.24.3-r0)
Executing busybox-1.37.0-r31.trigger
OK: 9952 KiB in 27 packages



rambo@ub2404:/tmp/apk-tool$ ls -lhd ../alpine-v3.24-rootfs/
drwxr-xr-x 19 root root 4.0K Oct 30 08:07 ../alpine-v3.24-rootfs/

rambo@ub2404:/tmp/apk-tool$ sudo du -sh ../alpine-v3.24-rootfs/
14M	../alpine-v3.24-rootfs/


# 开始打包
rambo@ub2404:/tmp/apk-tool$ cd  ../alpine-v3.24-rootfs
rambo@ub2404:/tmp/alpine-v3.24-rootfs$ sudo tar -czf /tmp/alpine-minirootfs-20261030-x86_64.tar.gz .
rambo@ub2404:/tmp/alpine-v3.24-rootfs$ ls -lh ../alpine-minirootfs-20261030-x86_64.tar.gz 
-rw-r--r-- 1 root root 6.9M Oct 30 08:17 ../alpine-minirootfs-20261030-x86_64.tar.gz


# 在Docker中测试使用
将生成的 tar.gz 放到你的 Dockerfile 同级目录下，编写测试：
rambo@ub2404:/tmp/alpine-v3.24-rootfs$ cd 
rambo@ub2404:~$ mkdir test1 && cd test1
rambo@ub2404:~/test1$ cp /tmp/alpine-minirootfs-20261030-x86_64.tar.gz .
rambo@ub2404:~/test1$ vim Dockerfile
FROM scratch
ADD alpine-minirootfs-20261030-x86_64.tar.gz /
CMD ["/bin/sh"]


# 构建并运行
rambo@ub2404:~/test1$ sudo docker build -t my-alpine:v3.24 .
rambo@ub2404:~/test1$ sudo docker run --rm -it my-alpine:v3.24 cat /etc/alpine-release
如果成功输出了 3.24.x 版本号，说明你已经成功自制并跑通了整个极简根文件系统

```


