# Linux Qemu环境搭建

## 源码下载

&emsp;&emsp;使用git clone下载linux、qemu和busybox源码，具体方法如下：

```C
git clone git@github.com:taokong1017/linux.git --depth=1
git clone git@github.com:taokong1017/busybox.git --depth=1
git clone git@github.com:taokong1017/qemu.git --depth=1
```

## Qemu编译

&emsp;&emsp;Ubuntu系统下，Qemu编译方法如下：

```C
sudo apt-get install ninja-build libglib2.0-dev libpixman-1-dev flex bison make
./configure –target-list=aarch64-softmmu
make -j && sudo make install
```

&emsp;&emsp;编译安装后，在/usr/local/bin下会生成qemu-system-aarch64文件。

## Busybox编译

&emsp;&emsp;构建busybox的目的，是构建Linux运行依赖的文件系统，具体配置方法如下：

```C
make ARCH=arm64 defconfig 
make ARCH=arm64 menuconfig
```

&emsp;&emsp;使能静态库，并指定文件系统安装目录，具体配置项如下：

```C
Settings  ---> 
    [*] Build static binary (no shared libs)
    --- Installation Options ("make install" behavior) 
        What kind of applet links to install (as soft-links)  ---> 
    (/home/ws/Src/rootfs) Destination path for 'make install'
```

&emsp;&emsp;编译，并安装文件系统，具体方法如下：

```c
make ARCH=arm64 CROSS_COMPILE=/home/ws/aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu- -j install
```

### 文件系统制作

&emsp;&emsp;文件系统的制作流程分为六个步骤，具体如下：

&emsp;&emsp;**步骤1，创建根文件目录**

```C
mkdir -p /home/ws/Src/rootfs; cd /home/ws/Src/rootfs; mkdir -p dev etc home lib mnt proc root sys tmp var
```

&emsp;&emsp;**步骤2，创建inittab**

```C
touch etc/inittab; chmod 755 etc/inittab; chmod +x etc/inittab
```

&emsp;&emsp;etc/inittab文件内容如下：

```C
#this is run first except when booting in single-user mode.
::sysinit:/etc/init.d/rcS
#/bin/sh invocations on selected ttys
::respawn:-/bin/sh
#start an "askfirst" shell on the console (whatever that may be)
::askfirst:-/bin/sh
#stuff to do when restarting the init process
::restart:/sbin/init
#stuff to do before rebooting
::ctrlaltdel:/sbin/reboot
::shutdown:/sbin/swapoff -a
```

&emsp;&emsp;**步骤3，创建rcS**

```C
mkdir -p etc/init.d/; touch etc/init.d/rcS; chmod +x etc/init.d/rcS
```

&emsp;&emsp;etc/init.d/rcS文件内容如下：

```C
#!/bin/sh
#This is the first script called by init process
/bin/mount -a
```

&emsp;&emsp;**步骤4，创建fstab**

```C
touch etc/fstab
```

&emsp;&emsp;etc/fstab文件内容如下：

```C
#device  mount-point   type    options    dump   fsck order
proc     /proc         proc    defaults   0      0
```

&emsp;&emsp;**步骤5，创建profile**

```C
touch etc/profile
```

&emsp;&emsp;etc/profile文件内容如下：

```C
#!/bin/sh
export HOSTNAME=farsight
export USER=root
export HOME=root
export PS1="[$USER@$HOSTNAME \W]\# "
#
#export PS1="[\[\033[01;32m\]$USER@\[\033[00m\]\[\033[01;34m\]$HOSTNAME\[\033[00m\ \W]\$ "
PATH=/bin:/sbin:/usr/bin:/usr/sbin
LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
export PATH LD_LIBRARY_PATH
```

&emsp;&emsp;**步骤5，打包文件系统**

```C
cd /home/ws/Src/rootfs;  find . | cpio -o -H newc |gzip > ../rootfs.cpio.gz
```

## Linux编译

&emsp;&emsp;Linux编译流程比较简单，配置方法如下：

```C
sudo apt-get install libncurses-dev libssl-dev
make ARCH=arm64 defconfig 
make ARCH=arm64 menuconfig
```

&emsp;&emsp;开启内核调试的配置如下：

```C
File systems 
    -> Pseudo filesystems 
        -> /proc file system support (PROC_FS [=y])
            -> /proc/kcore support (PROC_KCORE [=n])
```

&emsp;&emsp;关闭内核地址空间布局随机化（kaslr），具体方法如下：

```C
Kernel Features  --->
    [] Randomize the address of the kernel image
```

&emsp;&emsp;最后，编译构建Linux命令如下：

```C
make ARCH=arm64 CROSS_COMPILE=/home/ws/aarch64-none-elf/bin/aarch64-none-elf- Image -j
```

## Qemu启动Linux

&emsp;&emsp;Qemu启动Linux的命令如下:

```C
qemu-system-aarch64 -machine virt -nographic -m size=1024M -cpu cortex-a53 -smp 4 -kernel  /home/ws/Src/linux/arch/arm64/boot/Image -initrd /home/ws/Src/rootfs.cpio.gz -append "root=/dev/ram console=ttyAMA0 rdinit=/linuxrc"
```

启动后，输入日志如下：

```C
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 6.9.0-rc4-g977b1ef51866 (ws@ubuntu) (aarch64-none-elf-gcc (Arm GNU Toolchain 12.3.Rel1 (Build arm-12.35)) 12.3.1 20230626, GNU ld (Arm GNU Toolchain 12.3.Rel1 (Build arm-12.35)) 2.40.0.20230627) #8 SMP PREEMPT Mon Apr 22 23:15:40 CST 2024
[    0.000000] random: crng init done
[    0.000000] Machine model: linux,dummy-virt
[    0.000000] efi: UEFI not found.
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000040000000-0x000000007fffffff]
...
[    1.326364] PM: genpd: Disabling unused power domains
[    1.327635] ALSA device list:
[    1.327951]   No soundcards found.
[    1.385873] Freeing unused kernel memory: 10048K
[    1.389190] Run /linuxrc as init process

Please press Enter to activate this console. 
/ # pwd
/
/ # ls
bin      etc      lib      mnt      root     sys      usr
dev      home     linuxrc  proc     sbin     tmp      var
```
