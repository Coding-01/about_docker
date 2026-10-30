# scratch的延伸
```shell
FROM scratch 在容器语境下是极限裁剪的代名词，但在底层系统技术中，它本质上是对"如何纯手工组装一个无状态的块级或文件级根文件系统(Rootfs)"的抽象。
除了容器和前面排除的场景外，它在现代计算中还有一个极具技术含量的延伸应用：
微虚拟机与Serverless沙箱内核根镜像(如AWS Firecracker、Cloud-Hypervisor)
在这些场景中，为了追求毫秒级的冷启动速度和极小的内存占用，底层完全抛弃了传统的虚拟机磁盘镜像(qcow2/vmdk)，直接使用内存盘(Initramfs)或者由纯静态根文件系统打包的压缩包来挂载运行

# 嵌入式Linux固件的核心原理
在嵌入式设备（如路由器、智能网关）中，系统启动不依赖复杂的硬盘分区表和桌面引导程序，其标准链路为：
Bootloader（引导加载程序）：如U-Boot或PC端的GRUB，负责初始化硬件、将内核镜像读入内存
Linux Kernel（内核）：解压后接管硬件，在内存中挂载由内存盘（Initramfs）打包而成的微型文件系统
Init进程（1号进程）：文件系统中的 /init 脚本或程序，负责挂载虚拟文件系统（proc/sys/dev）并拉起用户态服务
整个过程不需要任何外部包管理器，完全由静态编译的二进制命令和脚本构成


还是以Ubuntu24.04.1为基础，从零开始亲手构建一个可运行的嵌入式Linux固件，并通过QEMU仿真器将其跑起来

本次将使用宿主机上的工具链，基于BusyBox和静态根文件系统，生成一个嵌入式Initramfs镜像，并用QEMU直接启动它


# 安装构建工具、静态busybox以及QEMU仿真器
rambo@ub2404:~$ sudo apt update && sudo apt install -y build-essential libncurses-dev bc bison flex libssl-dev libelf-dev qemu-system-x86 busybox-static cpio

# 创建根文件系统目录结构
嵌入式系统的标准目录结构非常精简，只需包含基础的挂载点和命令目录：
# 创建工作目录
rambo@ub2404:~$ mkdir -p /tmp/embedded_rootfs && cd /tmp/embedded_rootfs

# 建立标准目录
rambo@ub2404:/tmp/embedded_rootfs$ mkdir -p bin sbin etc proc sys dev root tmp

# 用Ubuntu自带的静态编译版BusyBox，将其复制并展开为几十个基础Linux命令（如sh、ls、echo等）：
# 复制静态编译的busybox到bin目录
rambo@ub2404:/tmp/embedded_rootfs$ cp /bin/busybox  bin/

# 通过软链接形式安装所有内置命令
rambo@ub2404:/tmp/embedded_rootfs$ cd bin/
rambo@ub2404:/tmp/embedded_rootfs/bin$ ./busybox --install -s .
rambo@ub2404:/tmp/embedded_rootfs/bin$ cd ..


# 编写1号引导脚本 (/init)
嵌入式系统的灵魂是 /init 脚本。当内核加载完根文件系统后，会无条件执行 /init 作为系统的PID 1进程
在 /tmp/embedded_rootfs 目录下创建 init 文件：
rambo@ub2404:/tmp/embedded_rootfs$ vim init
#!/bin/sh

# 1. 挂载内核必需的虚拟文件系统
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

# 2. 打印欢迎信息与系统状态
clear
echo "=========================================="
echo "  Welcome to Custom Embedded Linux System!"
echo "=========================================="
echo "Kernel version: $(uname -r)"
echo "Available memory:"
free -m

# 3. 启动busybox init进程
exec /sbin/init
EOF




# 赋予可执行权限(极其关键，否则内核无法执行PID 1)
rambo@ub2404:/tmp/embedded_rootfs$ chmod +x init

# 确保 /init 脚本没有 Windows 的 CRLF 换行符
rambo@ub2404:/tmp/embedded_rootfs$ sed -i 's/\r$//' init


# 创建里面的 /etc/inittab 配置文件
BusyBox init启动时必须读取该文件。在根目录下创建 etc/inittab：
rambo@ub2404:/tmp/embedded_rootfs$ vim etc/inittab
# /etc/inittab: BusyBox init configuration
::sysinit:/bin/echo "System initializing..."

# 关键：当系统启动后，在控制台(tty1 或串口 ttyS0)自动拉起一个shell
::respawn:/bin/cttyhack /bin/sh

# 关机时执行的操作
::shutdown:/bin/umount -a -r





# 打包为 Initramfs 镜像
嵌入式固件通常需要将整个根目录打包为压缩的cpio归档文件，供内核直接加载到内存中：
# 在根目录打包并通过gzip压缩
rambo@ub2404:/tmp/embedded_rootfs$ find . -print0 | cpio --null -o --format=newc | gzip -9 > /tmp/embedded_initramfs.cpio.gz
rambo@ub2404:/tmp/embedded_rootfs$ ls -lh ../embedded_initramfs.cpio.gz 
-rw-rw-r-- 1 rambo rambo 1.1M Oct 30 23:03 ../embedded_initramfs.cpio.gz


为了在QEMU中运行，可以直接借用Ubuntu24.04.1当前正在运行的内核镜像：
rambo@ub2404:/tmp/embedded_rootfs$ sudo cp /boot/vmlinuz-$(uname -r) /tmp/embedded_vmlinuz

# 通过QEMU启动并验证固件
rambo@ub2404:/tmp/embedded_rootfs$ sudo qemu-system-x86_64 \
    -kernel /tmp/embedded_vmlinuz \
    -initrd /tmp/embedded_initramfs.cpio.gz \
    -nographic \
    -append "console=ttyS0 panic=1"

# 命令参数注释
-kernel：指定你做好的Linux内核
-initrd：指定刚才打包的精简根文件系统固件
-nographic：不弹窗，直接将系统的标准输入输出重定向到当前终端
-append "console=ttyS0 panic=1"：告诉内核将控制台输出定向到串口，若启动崩溃则直接重启
启动后将在终端内直接看到引导日志，并最终进入由你亲手制作的极简嵌入式系统Shell(可输入ls、ps、mount 等命令进行交互，输入poweroff或关闭终端即可退出)

# 以下是回显
==========================================
  Welcome to Custom Embedded Linux System!
==========================================
Kernel version: 7.0.0-38-generic
Available memory:
              total        used        free      shared  buff/cache   available
Mem:             76          28          45           2           3          44
Swap:             0           0           0
System initializing...


BusyBox v1.36.1 (Ubuntu 1:1.36.1-6ubuntu3.1) built-in shell (ash)
Enter 'help' for a list of built-in commands.

~ # fdisk -l            # 因为还没有真实的硬盘所以这里输出为空
~ # df -Th
Filesystem           Type            Size      Used Available Use% Mounted on
devtmpfs             devtmpfs       32.5M         0     32.5M   0% /dev
~ # mount
rootfs on / type rootfs (rw,size=33308k,nr_inodes=8327,inode64)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
devtmpfs on /dev type devtmpfs (rw,relatime,size=33308k,nr_inodes=8327,mode=755,inode64)


~ # poweroff


```




# 尝试给固件"扩容"(挂载真实的虚拟硬盘)
```shell
当前的内存盘是只读且易失的（重启数据就没了）。如果你想模拟一个真正的嵌入式设备：
可在宿主机上用dd制作一个空白的磁盘镜像(比如 qemu-img create -f raw disk.img 100M)
在 QEMU 启动参数中挂载这块硬盘，然后在固件里用 fdisk 对它进行分区、用 mkfs.ext4 格式化，并把它挂载到根目录下的某个文件夹中


# 在宿主机上创建虚拟磁盘镜像

rambo@ub2404:/tmp/embedded_rootfs$ cd
rambo@ub2404:~$ qemu-img create -f raw /tmp/embedded_disk.img 100M

修改QEMU启动参数，通过-drive参数将刚创建的虚拟磁盘挂载为系统的块设备：
rambo@ub2404:~$ sudo qemu-system-x86_64 \
    -kernel /tmp/embedded_vmlinuz \
    -initrd /tmp/embedded_initramfs.cpio.gz \
    -drive file=/tmp/embedded_disk.img,format=raw,if=ide \
    -nographic \
    -append "console=ttyS0 panic=1"

# 参数解析：
1、if=ide 会让 Linux 内核将该虚拟盘识别为标准的 IDE 硬盘设备(通常在系统中表现为/dev/sda)
2、console=ttyS0：
含义：将 Linux 内核的控制台（Console）重定向到第一个串口（ttyS0）。
效果：所有的内核启动日志、驱动加载信息、以及最终拉起的 BusyBox Shell，都会通过这个虚拟串口输出。当配合 QEMU 的 -nographic 时，串口的数据就会直接流向你当前的宿主机终端窗口
panic=1：含义：当内核遇到无法恢复的严重错误（Kernel Panic）时，等待 1 秒然后自动重启，而不是无限期地死锁或挂起。这在自动化测试或嵌入式设备防死机中非常实用


# 默认的窗口会太小(80列 × 24行)所以需要更正下：
# 方法1
在QEMU启动命令中指定显式串口终端大小，即在 -append 参数里加入 conswitch 或调整串口波特率与控制台尺寸：
-append "console=ttyS0,115200n8 vt.default_utf8=1"
作用：指定串口的波特率为 115200、8 位数据位、无校验、1 位停止位（115200n8），并开启 UTF-8 编码支持
对窗口大小的影响：它本身不能自动放大终端的列数和行数。在 -nographic 模式下，终端的大小是由你当前在宿主机上打开的这个终端窗口（Terminal）本身的物理尺寸决定的。如果你的宿主机终端窗口本身很窄，这行参数无法改变它

# 方法2
如果你希望彻底摆脱这种纯文字的串口限制，可去掉 -nographic 参数，让 QEMU 弹出一个图形化的 SDL/GTK 窗口。这样不仅能自由拉伸窗口大小，还能直接享受标准的Linux图形化控制台：
sudo qemu-system-x86_64 \
    -kernel /tmp/embedded_vmlinuz \
    -initrd /tmp/embedded_initramfs.cpio.gz \
    -drive file=/tmp/embedded_disk.img,format=raw,if=ide \
    -append "console=tty0 console=ttyS0"
(提示：使用图形窗口时，内核参数指定了 console=tty0，系统会直接在弹出的QEMU虚拟机图形窗口里输出，窗口支持鼠标任意拉伸放大)
作用：这是解决窗口太小最彻底的方法。
效果：去掉-nographic后，QEMU会在宿主机桌面弹出一个独立的虚拟机图形窗口。这个窗口默认自带一个较为宽敞的虚拟屏幕分辨率。更棒的是，你可直接用鼠标任意拉伸、放大这个图形窗口，里面的文字和控制台也会随之获得更好的视觉体验

console=tty0：
含义：将控制台绑定到系统的虚拟前端屏幕（即由 QEMU 弹出的图形化显示窗口，对应 Linux 的 tty0 抽象)
效果：系统启动时，内核日志和 Shell 交互会直接输出到 QEMU 的图形桌面窗口里

console=tty0 console=ttyS0（双控制台叠加）：
含义：Linux 内核支持同时向多个控制台输出。这里先后声明了 tty0 和 ttyS0
效果：内核会把所有的启动日志同时复制并发送到这两个地方 --- 你既可以在弹出的 QEMU 图形窗口里看到它，也可以在宿主机的后台终端(串口)里看到它

# 以下是回显
....
  ....
==========================================
  Welcome to Custom Embedded Linux System!
==========================================
Kernel version: 7.0.0-38-generic
Available memory:
              total        used        free      shared  buff/cache   available
Mem:             76          28          45           2           3          44
Swap:             0           0           0
System initializing...


BusyBox v1.36.1 (Ubuntu 1:1.36.1-6ubuntu3.1) built-in shell (ash)
Enter 'help' for a list of built-in commands.

~ # fdisk -l
Disk /dev/sda: 100 MB, 104857600 bytes, 204800 sectors
12 cylinders, 255 heads, 63 sectors/track
Units: sectors of 1 * 512 = 512 bytes

Disk /dev/sda doesn't contain a valid partition table


# 对硬盘进行分区
~ # fdisk /dev/sda
Command (m for help): n        # 创建一个新分区
Partition type
   p   primary partition (1-4)
   e   extended
p                              # 选择主分区（Primary）
Partition number (1-4): 1      # 分区编号为 1
First sector (63-204799, default 63):    # 接受默认的起始扇区
Using default value 63
Last sector or +size{,K,M,G,T} (63-204799, default 204799):      # 接受默认的结束扇区(占满整个100M)
Using default value 204799
Command (m for help): w        # 保存分区表并退出



# 格式化分区
在精简的BusyBox环境中，没有直接提供mkfs.ext4这个软链接或独立二进制文件，但它内置了mke2fs
Ubuntu宿主机上自带的这个静态 BusyBox 内联版 mke2fs 裁剪得非常极致，不支持-t参数(它默认创建的是标准ext2格式)
虽然它不支持直接通过 -t ext4 指定，但在 Linux 内核中，ext4 是向下兼容 ext2 的。你可以直接用默认的 mke2fs 格式化，然后当作 ext4（或者直接以 ext2/ext3）挂载使用

~ # mke2fs  /dev/sda1
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
25688 inodes, 102368 blocks
5118 blocks (5%) reserved for the super user
First data block=1
Maximum filesystem blocks=262144
13 block groups
8192 blocks per group, 8192 fragments per group
1976 inodes per group
Superblock backups stored on blocks:
	8193, 24577, 40961, 57345, 73729
[  664.824466] random: crng init done


# 创建挂载点并挂载(可直接指定 -t ext4 挂载，内核会自动识别)
~ # mkdir -p /mnt/data
~ # mount -t ext4 /dev/sda1 /mnt/data
[  788.828319] EXT4-fs (sda1): mounted filesystem 2c4f3aea-b1bd-4452-bf7a-9e549768a59c r/w without journal. Quota mode: none.
[  788.840321] ext4 filesystem being mounted at /mnt/data supports timestamps until 2038-01-19 (0x7fffffff)

# 在挂载点下创建一个测试文件
~ # echo "what are you doing?" > /mnt/data/hello.txt

# 验证读取
~ # cat /mnt/data/hello.txt
what are you doing?


```
