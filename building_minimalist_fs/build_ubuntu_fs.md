# 安装依赖工具
> 在Ubuntu或Debian宿主机上
```shell
rambo@ub2404:~$ sudo apt install debootstrap arch-install-scripts
```

# 构建
```shell
# 使用debootstrap引导安装
以生成Ubuntu 26.04.1(Resolute Raccoon)极简根文件系统为例：
rambo@ub2404:~$ mkdir test2 && cd test2
rambo@ub2404:~/test2$ sudo debootstrap --arch=amd64 --variant=minbase resolute /tmp/ubuntu-26.04 http://archive.ubuntu.com/ubuntu/

注意：
在使用 debootstrap 时，不需要诸如 26.04.1 这样带小数点的主版本号
debootstrap识别的是Ubuntu的发行版代号(Codename)Ubuntu26.04(及其后续的 26.04.1、26.04.2 等点发布版)的底层套件代号都是 resolute(对应其代号 Resolute Raccoon)

# 进入/tmp/ubuntu-base目录，清理残留的apt缓存和日志
rambo@ub2404:~/test2$ sudo rm -rf /tmp/ubuntu-26.04/var/cache/apt/* /tmp/ubuntu-26.04/var/lib/apt/lists/*

# 打包为tar.gz
rambo@ub2404:~/test2$ sudo tar -czvf ubuntu-base-resolute-amd64.tar.gz -C /tmp/ubuntu-26.04 .
rambo@ub2404:~/test2$ ls -lh ubuntu-base-resolute-amd64.tar.gz 
-rw-r--r-- 1 root root 41M Oct 30 08:39 ubuntu-base-resolute-amd64.tar.gz


# 在Docker中测试使用
rambo@ub2404:~/test2$ vim Dockerfile
FROM scratch
ADD ubuntu-base-resolute-amd64.tar.gz  /
CMD ["/bin/bash"]

# 构建
rambo@ub2404:~/test1$ sudo docker build -t ubuntu:26.04 .
rambo@ub2404:~/test1$ sudo docker run --rm -it ubuntu:26.04 cat /etc/lsb-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=26.04
DISTRIB_CODENAME=resolute
DISTRIB_DESCRIPTION="Ubuntu 26.04 LTS"

如果成功输出版本信息说明你已经成功自制并跑通了整个极简根文件系统

```

