# Syzkaller

> google开源的fuzz工具syzkaller的安装教程

# 前言

最近要做一个利用syzkaller来fuzz内核并挖洞测试、优化的项目，所以在自己的虚拟机上试着部署了一下syzkaller然后往服务器上装一下开始fuzz，但是无论是github上面的官方文档还是其他博客上给出的安装方法都没有成功，这里给出了我目前成功的安装方式：

> 系统：ubuntu 16.04(64 bits)
>
> 内存：2GB
>
> 磁盘空间：30GB

# 准备阶段

* gcc 7.0以上版本
* g++ 7.0以上版本
* vim
* git

gcc和g++的作用体现在编译内核中，因此先来讲一下gcc和g++如何安装7.0版本以上。

## gcc&g++

网上给出的源码安装gcc 7.0的教程我一直都没有成功，最后总会出现莫名其妙的错误，所以我没有用源码安装而是之间apt安装。

```
sudo add-apt-repository ppa:jonathonf/gcc-7.2
sudo apt-get update
sudo apt-get install gcc-7 g++-7
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 60 --slave /usr/bin/g++ g++ /usr/bin/g++-7
```

然后在终端使用如下命令查看gcc和g++版本：

```
gcc --version
g++ --version
```

如果回显的版本与自己安装的版本相符合，那么证明安装成功。

但是仅仅这么做在编译内核的时候还是会报错的，因为gcc-ar、gcc-nm等软连接还是旧版本的软连接，因此要删掉旧版本的软连接并给新版本的建立软连接。

以下需要建立新软连接:(g++、gcc已经建立)

```
c++  gcc-ar       gcov-tool                x86_64-linux-gnu-gcc-7.1.0

cpp  gcc-nm       x86_64-linux-gnu-c++		 x86_64-linux-gnu-gcc-ar

g++  gcc-ranlib   x86_64-linux-gnu-g++  	 x86_64-linux-gnu-gcc-nm

gcc  gcov         x86_64-linux-gnu-gcc  	 x86_64-linux-gnu-gcc-ranlib
```

拿gcc-ar来举例：

* 找到gcc-ar原来的位置
* rm旧连接
* ln建立新连接

```
whereis gcc-ar
```

我的系统上gcc-ar的位置是：/usr/bin/gcc-ar

```
sudo rm -r /usr/bin/gcc-ar
sudo ln /usr/bin/gcc-ar-7 /usr/bin/gcc-ar
```

其他同理直到上述所有软连接全部建立好，然后更新gcc:

````
sudo apt-get install flex bison libc6-dev libc6-dev-i386 linux-libc-dev linux-libc-dev:i386 libgmp3-dev libmpfr-dev libmpc-dev build-essential bc
sudo apt install libssl-dev
````

## vim&git

直接利用apt安装即可

# 编译内核

## 下载内核

安装好了那些准备阶段需要的工具，我们就可以编译内核了,可以使用[syzkaller官方推荐的linux kernel](https://github.com/torvalds/linux)来fuzz也可以到[linux kernel官方下载网址](https://www.kernel.org/)下载其他版本的linux kernel，下载好之后保存到文件夹，记住这个文件夹的名字以及路径，这个后面会用到。

## make

进入kernel文件夹后执行以下指令：

````
make defconfig

make kvmconfig
````

然后利用vim打开.config更改下面几个选项：

````
CONFIG_KCOV=y
CONFIG_DEBUG_INFO=y
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_CONFIGFS_FS=y
CONFIG_SECURITYFS=y
````

在vim中搜索并删除.config中原有的关于这些选项的结果，例：

````
删除：
#CONFIG_KCOV=N
其他的同理
````

添加完这些选项后执行

```
make oldconfig
```

最后执行make：

````
make -j4
````

在kernel文件夹下执行```ls /vmlinux```和```ls /arch/x86/boot/bzImage```，若能看到相应文件则说明安装成功。

## 安装Image

安装debootstrap:

```
sudo apt-get install debootstrap
```

创建一个放置image的文件夹，进入该文件夹后执行

```
wget https://raw.githubusercontent.com/google/syzkaller/master/tools/create-image.sh -O create-image.sh
chmod +x create-image.sh
./create-image.sh
```

执行完成之后出现image文件

## QEMU

我们搞定内核之后还需要将它启动，因此需要安装QEMU

```
sudo apt-get install qemu-system-x86
```

注意：需要开启cpu虚拟化才能使用qemu启动内核，对于不同系统根据需求google即可。

在终端执行以下语句启动内核

```
qemu-system-x86_64 \
  -kernel $KERNEL/arch/x86/boot/bzImage \
  -append "console=ttyS0 root=/dev/sda debug earlyprintk=serial slub_debug=QUZ"\
  -hda $IMAGE/stretch.img \
  -net user,hostfwd=tcp::10021-:22 -net nic \
  -enable-kvm \
  -nographic \
  -m 2G \
  -smp 2 \
  -pidfile vm.pid \
  2>&1 | tee vm.log
```

注意：

$KERNEL是安装的linux kernel文件夹的路径，

$IMAGE是安装的image文件夹的路径

如果运行过程中出现报错可能是内存不足或者磁盘空间不足

启动成功之后需要在另一个终端ssh连接qemu

```
ssh -i $IMAGE/ssh/id_rsa -p 10021 -o "StrictHostKeyChecking no" root@localhost
```

运行结束后kill掉qemu就好

## GO语言

由于syzkaller是GO写的，因此配置好GO环境也是理所应当的，安装过程中要正确设置goroot和gopath

```
wget https://storage.googleapis.com/golang/go1.8.1.linux-amd64.tar.gz
tar -xf go1.8.1.linux-amd64.tar.gz
mv go goroot
export GOROOT=`pwd`/goroot
export PATH=$PATH:$GOROOT/bin
mkdir gopath
export GOPATH=`pwd`/gopath
```

## syzkaller

最后一步就是安装syzkaller了

```
go get -u -d github.com/google/syzkaller/...
cd gopath/src/github.com/google/syzkaller/
mkdir workdir
make
```

安装完之后在文件夹下创建一个config文件，里面包含以下内容：

```
{
	"target": "linux/amd64",
	"http": "127.0.0.1:56741",
	"workdir": "$GOPATH/src/github.com/google/syzkaller/workdir",
	"kernel_obj": "$KERNEL",
	"image": "$IMAGE/stretch.img",
	"sshkey": "$IMAGE/stretch.id_rsa",
	"syzkaller": "$GOPATH/src/github.com/google/syzkaller",
	"procs": 8,
	"type": "qemu",
	"vm": {
		"count": 4,
		"kernel": "$KERNEL/arch/x86/boot/bzImage",
		"cpu": 2,
		"mem": 2048
	}
}
```

$GOPATH记录系统中GO的安装路径

$IMAGE是image的文件夹路径

$KERNEL是目标内核的文件夹路径

这三个要素分别是为了启动syzkall、寻找syzkaller进行fuzz的目标

有关配置字段的具体含义可以参考[官方文档给出的解释](https://github.com/google/syzkaller/blob/master/docs/configuration.md)

运行syzkaller：

```
sudo ./bin/syz-manager -config=my.cfg
```

运行成功之后访问:127.0.0.1:56741可以查看就能看到一个包括系统调用的覆盖量，syzkaller生成的程序数量,内核被crash的日志报告等的调试信息。

## 后续

关于syzkaller去fuzz的结果以及syzkaller官方文档中的一些其他有用的信息会在后续文章真难过进行整理。

## 参考

[syzkaller官方文档](https://github.com/google/syzkaller)

[Syzkaller：Linux内核模糊测试工具分享](https://www.freebuf.com/sectool/142969.html)
