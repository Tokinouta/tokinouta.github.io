---
title: 编译内核
date: '2024-01-12 15:29:47'
updated: '2024-11-30 21:04:01'
tags:
  - Linux
  - 内核
  - 编译
permalink: /post/compile-kernel-zegy7a.html
comments: true
toc: true
---



要学习 Linux 内核，最好还是能够自己编译一个内核。这样能够实际体会到编译过程中的一些细节。

编译内核并不难，如果只是想要体验一下编译流程并且运行一下最终的内核的话，可以利用一份 Linux 内核源码和 qemu 虚拟机。qemu 提供了一个[Direct Linux Boot ](https://www.qemu.org/docs/master/system/linuxboot.html)机制，可以用一个 Linux 内核镜像和一个根文件系统启动一个虚拟机，然后在里面就可以体验了。

编译内核最简单的方式如下：

1. 从 [The Linux Kernel Archives](https://www.kernel.org/) 获取一份源码，解压到任意文件夹。
2. 使用 `make defconfig`​ 生成一份编译配置。如果想用 llvm 工具链，可以在 `make`​ 后面加上一个 `LLVM=-17`​ 的参数，这说的是用 17 版本的工具链。详细信息可以参考[Building Linux with Clang/LLVM](https://www.kernel.org/doc/html/latest/kbuild/llvm.html)。
3. 使用 `make -j$(nproc) bzImage`​ 编译内核镜像。
4. 为 Linux 制作一个最简单的文件系统。这套文件系统使用 busybox，需要首先编译一份静态链接的 busybox，然后制作一个文件系统镜像。

    1. 生成一个空镜像文件 `dd if=/dev/zero of=rootfs.img bs=1M count=20; mkfs.ext4 rootfs.img`​，并挂载到新建的 `fs`​ 文件夹下 `sudo mount -t ext4 -o loop rootfs.img ./fs`​。
    2. 获取 busybox，解压。
    3. 使用 `make menuconfig`​ 生成编译配置，这里需要调整的是 Settings->Build static binary，把这个选项勾选上。
    4. 编译 busybox，注意最后一个参数需要指向第一步的 `fs`​ 文件夹 `sudo make install -j $(nproc) CONFIG_PREFIX=../fs`​。
    5. 给文件系统里创建必要的文件夹

        ```bash
        cd ../fs
        sudo mkdir proc dev etc home mnt
        sudo cp -r ../busybox-1.35.0/examples/bootfloppy/etc/* etc/
        ```
    6. 卸载临时文件系统 `sudo umount fs`​
5. 内核，启动！`qemu-system-x86_64 -kernel linux-6.7/arch/x86_64/boot/bzImage -hda rootfs.img -append "root=/dev/sda console=ttyS0" -nographic`​

然后就可以看到激动人心的命令行了。

## 参考链接

* [教你在十分钟内编译一个Linux内核，并在虚拟机里运行 – 龙进的技术笔记 (longjin666.cn)](https://longjin666.cn/1599/)
* [Linux内核编译与安装（完整过程）【Ubuntu18.04下】_linux 内核编译-CSDN博客](https://blog.csdn.net/shiftrain/article/details/118575854)
* [技术|Linux 内核动手编译实用指南](https://linux.cn/article-16252-1.html)
* [手把手教你Linux内核编译(三天吐血经历) - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/456536119)
* [Qemu 启动 Linux Kernel(Arm64)](https://wangloo.github.io/posts/os/arm64-linux-qemu/)
