# oemplay
一种类似ansible的流水线，用于开放给客户定制系统。从定制到安装进入在到用户界面主要分为三个阶段。
## 构建阶段
构建阶段的工作是将原始iso解压，定制部分内容后重新打包。有如下几个步骤：
 - 解压iso
 - 解压filesystem.squashfs
 - 根文件系统定制
 - 解压oem.squashfs
 - oem目录定制
 - 打包filesystem.squashfs
 - 打包oem.squashfs
 - iso根目录定制
 - 镜像签名
 - 打包iso

### 根文件系统定制
镜像安装的主要工作就是将压缩的根文件系统(filesystem.squashfs)，解压到目标分区。因此，如果提前将配置和安装包预置到根文件系统，不就可以完成我们的定制了？我们的原始镜像就是这么做的。它直接有效，但缺点非常明显，可以参考`oem目录定制`。`oem目录定制`有那么多优点,为什么还要保留根文件系统定制？
因为安装阶段对系统的修改，影响也只能在修改之后生效，对第一次grub启动，加载内核与驱动(在initramfs内)，lightdm启动等安装器启动之前的流程毫无作用。因此，硬件驱动，备份还原等依赖启动阶段执行的功能需要在此阶段定制。

### oem目录定制
oem目录定制是最常见的操作，由于filesystem.squashfs是一个完整的根文件系统，一般压缩文件都有几个G，解压和打包都会占用大量的CPU和IO资源，当多个任务并发时，很容易造成定制超时，甚至宿主机卡死。
因此，将对系统的修改延迟到安装阶段，在有限的服务器资源上可以提高定制的用户体验。此外，在宿主机上依赖chroot操作根文件系统，需要同体系结构的CPU架构。考虑到定制的本质就是执行命令和脚本来完成对系统的修改，如果将定制执行延迟到安装阶段，那么构建阶段只需要复制配置文件和脚本到系统，而复制操作不受CPU架构影响 可以充分利用现有服务器资源。
所有，oem目录定制的本质就是将对系统的修改脚本、安装包、配置文件等打包到iso，等安装阶段再运行。缺点也比较明显，安装相对比较慢。

### iso根目录定制
根文件系统定制和oem目录定制，都是对squashfs压缩文件的定制，iso镜像内其实还有定制的场景。如，U盘启动的grub启动项文案，由镜像根目录下的boot内的grub.cfg控制。此外有的镜像会打包一个微型仓库到镜像根目录下。
## 安装阶段
### 第一次启动
安装阶段是我们大部分定制工作的实际执行阶段，也是我们经常出错的阶段。理解安装阶段的整个工作流程，对我们编写合适的定制脚本以及排除故障都大有裨益。在此也会穿插讲解一些bootloader的知识，下文的BIOS通指UEFI接口编译BIOS固件，而非传统的legacy BIOS固件。
- BIOS扫描设备，把USB设备默认为可启动设备。
- 挂载可启动设备的文件系统，遍历BIOS内置的bootloader路径，x86的一般为\EFI\Boot\bootX64.efi(PE32格式)。
- 如果启动设备是光盘，则BIOS读取ISO镜像内的`El Torito`引导数据,通过该数据找到efi.img,会被挂载为efi分区,从该分区读取bootloader。
- BIOS把bootloader(linux上就是grub)作为可执行二进制执行，CPU的控制权交给了bootloader。
- Grub加载内置文件(生成efi的时候把文件当模块)，从配置内获知外部的grub.cfg相对路径，加载其他模块，再解析执行外部grub.cfg，可能会执行加载其他mod，最后显示grub启动项。
- 用户进入交互逻辑，选择启动项后，可以`enter`继续执行；可以按`e`编辑启动项用于调试，安装F10继续执行；也可以按`c`进入Grub Shell模式，这是一个简单的交互shell，不是bash，可用于调试。
- 当用户`enter`执行时候，找到配置的linux内核路径并加载，指定使用配置里的initrd路径作为内核的为initramfs,还能指定其他内核启动参数,也就说U盘里的配置影响安装启动参数。

到这里，CPU的控制权交给了内核，初始化一些数据结构，如果内核为压缩文件，在内存内解压内核文件，再边解析内核参数边执行内核代码，遍历执行内核文件的section，完成主流程的初始化，文件系统初始化，驱动框架初始化，内置驱动模块初始化，挂载initrd为临时根文件系统(initramfs),执行initramfs内的init作为第一个进程，它是一个bash脚本，该进程解析/proc/cmdline的内核参数，如外部驱动加载，备份还原，挂载并安装PXE环境里的IOS镜像等。

initramfs内执行一个关键步骤就是加载或者屏蔽驱动，如果缺少某个驱动导致系统启动异常，那么需要`根文件系统定制`阶段安装新的内核或者驱动模块，然后`update-initramfs -u`重新生成内核和initrd到替换ISO镜像内live目录下载的内核initrd文件。

在这里init会解析知道是livecd，将filesystem.squashfs挂载到为临时目录，再切换这个新的目录作为根文件系统，启动init进程，也就是systemd，再拉起lightdm，ligtdm拉起安装器。接下来就是安装器的执行逻辑。
上面的流程总结为:
- 内核启动initramfs内的init进程。
- init进程解析/proc/cmdline后挂载filesystem.squashfs。
- init切换到到新的根文件系统并启动systemd。
- systemd启动lightdm，lightdm启动安装器。
### 安装器执行

未完待续...

## 第一次启动阶段

未完待续...

## 其他知识：
1.由于UEFI是Intel和Microsoft联合制定，所以booloader是PE格式。当我们提到Bootloader加载内核后，CPU控制权交给内核，由于Grub做了兼容处理，内核既可以是ELF格式的Linux，也可以是继续交给windows的PE格式的bootloader，叫做链式加载。

2.控制权，从BIOS交给bootloader，再交给内核，BIOS是否可以直接交给内核绕过bootloader呢？答案是可以的(https://zhuanlan.zhihu.com/p/28708585)。我们只需要把ELF的你和开启efistub选项，把内核编译成PE格式即可。stub即桩代码，就是承上启下的中间代码,把压缩格式的linux内核文件头转为PE文件头。BIOS加载PE的时候，相当于父进程打开子进程，子进程的main入口函数参数不是C的**argv，而是`EFI_SYSTEM_TABLE *SystemTable`,efi系统表指针是BIOS内置功能函数或者服务的对外同一接口，通过这个表内获取其他操作BIOS硬件功能，如时钟，nvram等。不管是grub.efi，还是linux.efi，只要使用efi的mian函数作为入口地址，编译的PE二进制格式，就能被BIOS加载并执行，这样的efi程序也能对BIOS开放的功能进行设置。当然，Grub再通过其他方法传给内核的系统表也是能用的，否则很多系统提供的efi接口就不能用了。
https://www.kernel.org/doc/Documentation/efi-stub.txt