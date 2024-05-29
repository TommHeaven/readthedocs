



## 项目3 Rust for Linux & 跨内核驱动框架

### 1. 树莓派模拟环境搭建

1. 需要Linux系统，以便进行内核开发。可以在物理机上安装Linux，也可以使用虚拟机的形式。（如Windows下的WSL2，或者VirtualBox、VMWare），**推荐安装WSL2**，相关安装可参考网上资料。关于Linux发行版，我们建议使用Ubuntu LTS 20.04即可,其他如Debian,Arch, Fedora等,注意使用的包管理器和包名与Deb系的略有区别，需要自行解决。
2. Rust的安装不过多强调，应该都安装了。

#### -  环境配置
进入Linux环境，项目的开发将在r4l_drivers里面进行，该项目目录将会包含以下内容：
```
r4l_driver
    |- linux_raspberrypi                           (树莓派linux kernel源码)
    |- qemu                                        (qemu模拟环境)
    |- 2024-03-12-raspios-bookworm-arm64-lite.img  (官方树莓派镜像)
    |- firmware                                    (用于挂载镜像分区1)
    |- rootfs                                      (用于挂载镜像分区2)
    |- userconf                                    (初始用户文件)    
    |- boot.sh                                     (qemu启动脚本)
```

通过以下命令创建项目，并作用到环境变量 `.bashrc` or `.zshrc`

```sh
mkdir /path/to/your/r4l_drivers/
echo "export R4L_DRV=$(pwd)" >> ~/.bashrc
source ~/.bashrc
```

**获取树莓派官方xz镜像文件**

```sh
cd $R4L_DRV
wget https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-13/2024-03-12-raspios-bookworm-arm64-lite.img.xz
xz -dk 2024-03-12-raspios-bookworm-arm64-lite.img.xz
sync
sudo rm 2024-03-12-raspios-bookworm-arm64-lite.img.xz
```
**Qemu9.0 模拟器安装和使用**

```sh
cd $R4L_DRV
# 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev libsdl2-dev \
              git tmux python3 python3-pip ninja-build
# 下载源码包           
wget https://download.qemu.org/qemu-9.0.0.tar.xz
tar -xvf qemu-9.0.0.tar.xz
rm qemu-9.0.0.tar.xz
cd qemu-9.0.0 && mkdir build
cd build && ../configure --target-list=aarch64-softmmu,arm-softmmu
make -j$(nproc)

# 作用到环境变量 `.bashrc` or `.zshrc`
echo "export PATH=\$PATH:$R4L_DRV/qemu-9.0.0/build/" >> ~/.bashrc
source ~/.bashrc
#验证qemu是否安装成功
qemu-system-aarch64 --version
#出现以下信息表示安装成功:
# QEMU emulator version 9.0.0
# Copyright (c) 2003-2024 Fabrice Bellard and the QEMU Project developers
```
**树莓派 linux安装和编译**

```sh
cd $R4L_DRV
git clone --depth=20 https://github.com/happy-thw/linux_raspberrypi.git
cd linux_raspberrypi
# Requires dependent libraries
apt install libelf-dev libgmp-dev libmpc-dev bc flex bison u-boot-tools
apt install llvm-17 lld-17 clang-17  
# 这里选择的是clang-17的版本，rust for linux 最少也要12以上，并且下面的LLVM根据相应的版本选择

# Requires Rust dependent libs
rustup override set $(scripts/min-tool-version.sh rustc)
rustup component add rust-src
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen-cli
make ARCH=arm64  O=build_4b LLVM=-17 rustavailable 
# Rust is available!  即支持rust

make ARCH=arm64 O=build_4b LLVM=-17 bcm2711_rust_defconfig
make ARCH=arm64 O=build_4b LLVM=-17 -j$(nproc)

# (可选项) 修改.config可通过
make ARCH=arm64 O=build_4b LLVM=-17 menuconfig
# (可选项) 保存修改的.config为默认config
make ARCH=arm64 O=build_4b LLVM=-17 savedefconfig && mv build_4b/defconfig arch/arm64/configs/bcm2711_rust_defconfig
# (可选项) 删除之前编译所生成的文件和配置文件
make ARCH=arm64 O=build_4b LLVM=-17 mrproper
# (可选项) 使用rust-analyzer
make ARCH=arm64 O=build_4b LLVM=-17 rust-analyzer

# 没有出现error，则编译成功，相应的kernel镜像文件和设备树文件在以下路径
ls ./build_4b/arch/arm64/boot/Image
ls ./build_4b/arch/arm64/boot/dts/broadcom/*.dtb
```

**启动树莓派模拟环境**

```sh
cd $R4L_DRV
fdisk -l 2024-03-12-raspios-bookworm-arm64-lite.img
#得到以下信息：
# Disk 2024-03-12-raspios-bookworm-arm64-lite.img: 4 GiB, 4294967296 bytes, 8388608 sectors
# Units: sectors of 1 * 512 = 512 bytes
# Sector size (logical/physical): 512 bytes / 512 bytes
# I/O size (minimum/optimal): 512 bytes / 512 bytes
# Disklabel type: dos
# Disk identifier: 0xd08d1401

# Device                                      Boot   Start     End Sectors  Size Id Type
# 2024-03-12-raspios-bookworm-arm64-lite.img1         8192 1056767 1048576  512M  c W95 FAT32 (LBA)
# 2024-03-12-raspios-bookworm-arm64-lite.img2      1056768 5406719 4349952  2.1G 83 Linux

#根据img1 Start的信息，将分区1挂载到firmware下，该分区里面存放了启动引导和配置的相关文件
mkdir firmware
sudo mount -v -o offset=$((512*8192)) 2024-03-12-raspios-bookworm-arm64-lite.img firmware

# 里面可以看到相应的kernel镜像，配置文件和设备树文件
# 卸载
sudo umount 2024-03-12-raspios-bookworm-arm64-lite.img firmware

#另外需要给镜像扩容
qemu-img resize 2024-03-12-raspios-bookworm-arm64-lite.img 4G
qemu-img info 2024-03-12-raspios-bookworm-arm64-lite.img

#创建boot.sh脚本，用于启动qemu，虽然qemu9.0版本开始支持树莓派4B，但是尝试后发现键盘没有作用-.-，因此启动树莓派3B+也能满足项目需求
#以下是boot.sh的内容
#!/bin/sh
kernel_image="linux_raspberrypi/build_4b/arch/arm64/boot/Image"
kenrnel_dtb="linux_raspberrypi/build_4b/arch/arm64/boot/dts/broadcom/bcm2710-rpi-3-b-plus.dtb"
IMG="2024-03-12-raspios-bookworm-arm64-lite.img"
qemu-system-aarch64 \
        -machine type=raspi3b \
        -m 1024 \
        -k en-us \
        -dtb $kenrnel_dtb \
        -kernel $kernel_image \
        -drive id=hd-root,format=raw,file=$IMG \
        -append "rw earlycon=pl011,0x3f201000 console=ttyAMA0 loglevel=8 root=/dev/mmcblk0p2 \
        fsck.repair=yes net.ifnames=0 rootwait memtest=1 dwc_otg.fiq_fsm_enable=1" \
        -serial stdio \
        -usb -device usb-kbd \
        -device usb-tablet -device usb-net
        

#启动boot.sh
chmod 777 boot.sh
./boot.sh
# 可能启动会比较慢一点...
```

结果如下：

![result](../source/picture/pic1.png)

附：感兴趣的可以尝试下启动树莓派4B，执行命令如下：

```sh
cd $R4L_DRV
qemu-system-aarch64 \
	-machine raspi4b \
	-cpu cortex-a72 \
	-smp 4 \
	-m 2G \
	-kernel linux_raspberrypi/build_4b/arch/arm64/boot/Image \
	-dtb linux_raspberrypi/build_4b/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb \
	-drive id=hd-root,format=raw,file=2024-03-12-raspios-bookworm-arm64-lite.img \
	-append "rw earlycon=pl011,0x3f201000 console=ttyAMA0 loglevel=8 \
	root=/dev/mmcblk1p2 rootfstype=ext4 rootwait memtest=1 dwc_otg.fiq_fsm_enable=0" \
	-serial stdio -device usb-kbd -device usb-tablet
```

**加载linux驱动模块**

在sample里面已有R4L社区提交的一些驱动模块，在路径`./linux_raspberrypi/build_4b/samples/rust`下,把`rust_print.ko`移动到qemu环境里去

```sh
cd $R4L_DRV
mkdir rootfs
#根据前面img2 Start的信息，将分区2挂载到rootfs目录，该分区是根文件系统
sudo mount -v -o offset=$((512*1056768)) 2024-03-12-raspios-bookworm-arm64-lite.img rootfs
sudo cp ./linux_raspberrypi/build_4b/samples/rust/*.ko ./rootfs/home/pi/
#卸载
sudo umount 2024-03-12-raspios-bookworm-arm64-lite.img rootfs
#重新启动qemu，此时在qemu环境下就能看到放进来的驱动模块了（想通过网络传输的，但是wsl2好像没这么简单...）
```

驱动加载演示如下所示

![result2](../source/picture/pic2.png)

以上就是通过QEMU的树莓派模拟环境去进行驱动开发的过程。

相关问题解决：

1.修改初始密码，在qemu树莓派环境中通过`sudo raspi-config`进行配置，参考链接:[系统配置 | Raspberry Pi 树莓派 (pidoc.cn)](https://pidoc.cn/docs/computers/configuration)

2.键盘输入乱码，是键盘布局不对，树莓派(raspberry pi)是英国产品，默认键盘布局是英国(GB)，参考[树莓派(raspberry pi)学习4: 更改键盘布局_树莓派配置键盘布局-CSDN博客](https://blog.csdn.net/c80486/article/details/8460271)



### 2. 树莓派开发板硬件驱动开发

相关硬件：1.树莓派4B开发板  

​					2.HDMI线和显示屏  

​					3.串口TTL转USB模块 

​                    4.网线  

​            		5.面包板、电阻和LED小灯（不一定需要）

根据官方文档进行镜像烧录，同样选择https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-13/2024-03-12-raspios-bookworm-arm64-lite.img.xz，通过HDMI和显示屏查看树莓派硬件启动阶段的相关信息，尤其是无线连接的IP地址，然后通过SSH方式就可以进入树莓派终端了。

linux开源下载&编译方式和前面qemu里面步骤相同。

考虑到无线wifi的ip是动态分配的，可配置有线网络的静态IP，命令如下：

```sh
sudo nmcli -p connection show

# Sample static IP configuration:
sudo nmcli c mod 'Wired connection 1' ipv4.addresses 20.6.2.199/22 ipv4.method manual
sudo nmcli con mod 'Wired connection 1' ipv4.gateway 20.6.2.1
sudo nmcli con mod 'Wired connection 1' ipv4.dns "20.6.2.1"
sudo nmcli c down 'Wired connection 1' && sudo nmcli c up 'Wired connection 1'
# 其中IP,掩码,网关，DNS根据自己的情况进行配置
```

驱动模块通过scp就可以进行传输。

另外还有相关的系统配置问题，可参考树莓派文档进行查看：[系统配置 | Raspberry Pi 树莓派 (pidoc.cn)](https://pidoc.cn/docs/computers/configuration)



### 练习一： 根据上面的方法配置环境，编写和加载HelloWorld的驱动模块。完成后截图。

### 练习二： 实现`Miscdev`驱动基本读写
- 远端仓库 [代码仓库](https://github.com/happy-thw/linux_raspberrypi/commit/ca5198f6449b4076f3a48df9c9a7e71460977cab) 更新了练习二的内容，添加了一个samples/rust/rust_miscdev.rs文件，但是没有绑定驱动;
- 按要求添加配置项，补充内容，使得杂项字符设备可以实现基本读写操作。
- 提示：里面缺少到miscdev抽象文件可以从rust for linux社区rust分支获取。
- 测试样例：
```
//! How to build only modules:
//! make LLVM=-17 O=build_4b ARCH=arm64 M=samples/rust
//!
//! How to use in qemu:
//! / # sudo insmod rust_miscdev.ko
//! / # sudo cat /proc/misc  -> c 10 122
//! / # sudo chmod 777 /dev/rust_misc
//! / # sudo echo "hello" > /dev/rust_misc
//! / # sudo cat /dev/rust_misc  -> Hello
//!
``` 
- 测试结果如图：  
![result](../source/picture/pic3.png)

### 练习三： 跨内核驱动框架下的树莓派4B GPIO点灯实现
- 远端仓库 [代码仓库-练习三](https://github.com/happy-thw/linux_raspberrypi/commit/61785d5672d7006391cf589fd75774eb22381627) 更新了练习三的内容，添加了一个drivers/gpio/exercise-3-gpio目录,包括C语言的树莓派4B点灯的参考代码，以及通过Cargo new的一个跨内核驱动框架Adapter driver的目录，里面提供了练习作业的相关说明;
- 参考代码链接: [DesignWare_APB_GPIO Adapter Driver模块](https://github.com/happy-thw/gpio-dw-linux-adapter) 以及 [DesignWare_APB_GPIO Pure Driver模块](https://github.com/happy-thw/gpio-dw-linux-adapter),参考drivers/gpio/gpio-dwapb.c的C代码实现
- 注意事项：  
    1. 只支持驱动模块build-in，无法基于Cargo生成.ko文件
    2. 编译跨内核的驱动时，无法使用多线程并行编译（不能make -jx）,由于Cargo的依赖重定位问题导致(建议先编译内核，然后修改跨内核的驱动配置项，单线程编译）
    3. 需要在rust/cargo_driver/Cargo.toml中添加adapter层的dependencies,例如：
        ```
        [dependencies]
        gpio_rpbled_linux_adapt  = { path = "../../drivers/gpio/exercise-3-gpio/gpio-rpbled-adapter"}
        ```
        以及Rust/makefile 476行中增加binaries，adapter层Crate的name（可在改配置后面追加）
        ```
        # When adding a cargo project, you must add the binaries
        # that need to be excluded here. 
        # They will be removed when the cargo_driver is actually generated.
        CARGO_EXCLUDE_OBJ := gpio_rpbled_linux_adapt
        ```
- 在树莓派中运行Arceos:
    1. 克隆这个仓库,生成ArceOS代码仓库：https://github.com/chenlongos/arceos
    2. 在ArceOS目录下，输入：

        ```shell
        make A=apps/boards/raspi4 ARCH=aarch64 PLATFORM=aarch64-raspi4 LOG=debug
        ```
        编译出ArceOS在raspi4上的.bin镜像。
    3. 生成kernel8.img文件
        ```shell
        git clone https://github.com/chenlongos/rust-raspberrypi-OS-tutorials.git
        # 在在06_uart_chainloader目录下，执行：
        BSP=rpi4 make
        ```
        便可以看到生成了一个kernel8.img文件
    4. 把 kernel8.img 和 raspi4_aarch64-raspi4.bin 通过 cat 命令拼接到一个 bin 文件中，仍然取名为 kernel8.img:
        ```shell
        # 在ArceOS目录下执行
        cat ../rust-raspberrypi-OS-tutorials/06_uart_chainloader/kernel8.img apps/boards/raspi4/raspi4_aarch64-raspi4.bin > kernel8.img
        ``` 
    5. 已有树莓派开发板:把新生成的 kernel8.img 拷贝到 sd 卡上,替换原来的kernel8.img，上电启动。
    6. 在qemu模拟器中运行:
        ```shell
        # 在qemu中运行kernel8.img
        qemu-system-aarch64 -m 2G -smp 4 -cpu cortex-a72 -machine raspi4b -nographic -kernel arceos/kernel8.img
        ```
    GPIO在Arceos的点灯驱动可以参考: [https://github.com/happy-thw/arceos/commit/2f5aae5b1d74ac55cd3cee4f0e8240acb9c99618](https://github.com/happy-thw/arceos/commit/2f5aae5b1d74ac55cd3cee4f0e8240acb9c99618) ;  
    更详细的信息参考: [ttps://chenlongos.com/raspi4-with-arceos-doc/chapter_1.1.html](https://chenlongos.com/raspi4-with-arceos-doc/chapter_1.1.html)
    - 测试样例：  
        1. 在R4L里面还是通过msicdev设备驱动操作gpio点灯，选取的Gpio Pin引脚随意，但是需要说明寄存器和引脚的关系，相关数据手册查看官网和网上资料
        ``` sh
        sudo echo 1 > /dev/rust_misc # 灯亮
        sudo echo 0 > /dev/rust_misc # 灯灭
        ```
        2. 在Arceos里面通过shell里面新增cmd操作gpio点灯，达到循环亮灭的效果
        3. Arceos和R4L中使用同一份Pure driver代码，只完成R4L部分获得该练习作业50%的分值。
