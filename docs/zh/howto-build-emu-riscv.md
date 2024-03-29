**如何编译 android emulator**

注：提供此文档为方便国内参与小伙伴。

<!-- TOC -->

- [1. 硬件环境](#1-硬件环境)
- [2. 安装依赖软件](#2-安装依赖软件)
- [3. 安装 repo](#3-安装-repo)
- [4. 下载源码](#4-下载源码)
- [5. 构建](#5-构建)
    - [5.1. 增量构建](#51-增量构建)
    - [5.2. 使用 “ccache” 加速构建](#52-使用-ccache-加速构建)
- [6. 使用生成的 AOSP 系统映像进行测试](#6-使用生成的-aosp-系统映像进行测试)

<!-- /TOC -->

# 1. 硬件环境

本文所有操作在以下系统环境下验证通过：

```
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.6 LTS
Release:        18.04
Codename:       bionic
```

# 2. 安装依赖软件

```
$ sudo apt-get install -y git build-essential python qemu-kvm ninja-build \
                          python-pip ccache
```

# 3. 安装 repo

下载（参考 [清华大学开源软件镜像站 Git Repo 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/git-repo/)）

```
$ curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
$ chmod +x repo
```
为了方便可以将其拷贝到你的 PATH 里。

repo 运行过程中会尝试访问官方的 git 源更新自己，如果想使用 tuna 的镜像源进行更新，可以
将如下内容复制到你的 `~/.bashrc` 里并重启终端生效。

```
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'
```

安装好后可以运行 `repo version` 检查效果，出现类似以下输出说明安装成功。repo 版本需要 2.15 以上。

```
$ repo version
<repo not installed>
repo launcher version 2.17
       (from /home/wangchen/bin/repo)
git 2.17.1
Python 3.6.9 (default, Jan 26 2021, 15:33:00)
[GCC 8.4.0]
OS Linux 4.15.0-144-generic (#148-Ubuntu SMP Sat May 8 02:33:43 UTC 2021)
CPU x86_64 (x86_64)
Bug reports: https://bugs.chromium.org/p/gerrit/issues/entry?template=Repo+tool+issue
```

# 4. 下载源码

创建一个模拟器源码构建目录，这里假设是`/home/u/emu-dev`，然后进入该目录。

```
$ mkdir -p /home/u/emu-dev && cd /home/u/emu-dev
```

进入构建目录后，执行以下命令下载源代码。

```
$ repo init -u git@gitee.com:aosp-riscv/platform_manifest -b riscv64-emu-31.2.1.0-dev-cn
$ repo sync -j8
```

# 5. 构建

```
$ cd external/qemu && android/rebuild.sh
```

如果一切顺利，你应该在 objs 目录中看到生成的模拟器。

您可以在运行 `rebuild.sh` 脚本时带上 `--help` 查看更多帮助。

## 5.1. 增量构建

`rebuild.sh` 脚本会进行完全干净的构建。 您可以使用 ninja 进行部分构建：

```
$ ninja -C objs
```

## 5.2. 使用 “ccache” 加速构建 

强烈建议在您的开发中安装 “ccache”, Android 模拟器构建脚本将探测它并使用它, 这可以大大
加快增量构建的速度。

```
sudo apt-get install ccache
```

# 6. 使用生成的 AOSP 系统映像进行测试

可以尝试使用基于 AOSP 源码做出来的 Android 系统映像来启动我们自己构建的模拟器。基于 AOSP
源码制作 Android 系统镜像首先要选择一个支持模拟器的产品，例如：

```
$ cd $AOSP/
$ . build/envsetup.sh 
$ lunch sdk_phone_arm64-eng
$ make -j8
```

`$AOSP` 是你 aosp 源码树的路径。

推荐选择的产品类型有 'sdk_phone_arm64-eng' 或者 'sdk_phone_x86_64-eng'。

利用我们自己构建的模拟器启动 AOSP 系统镜像的方法如下：

```
$ cd /home/u/emu-dev/external/qemu
$ ./android/rebuild.sh 
$ export ANDROID_BUILD_TOP=/path/to/aosp
$ objs/emulator
```

注意：
- 以上启动 emulator 的命令行必须和 lunch 命令在一个终端会话中，否则会报错。
- 如果您想在没有图形界面的的模式（headless mode）下启动模拟器，您可以添加 `-no-window` 选项。
- 如果您看到错误：“pulseaudio: Failed to initialize PA contextaudio: Could not
  init 'pa' audio driver”，您可以添加 `-no-audio` 选项。
- 如果您看到错误：“PCI bus not available for hda”，您可以添加 `-qemu -machine virt`。
- 如果您想在 headless mode 下查看内核日志，可以添加 `-show-kernel` 选项。

综上所述，假设我们已经编译生成了 `sdk_phone_arm64-eng` 的 aosp image 和 emulator，现
在我们想用自己编译的 emulator 运行测试一下这个 aosp image，，并且采用文本模式，不启动
图形界面，可以输入如下命令：
```
$ cd $AOSP/
$ . build/envsetup.sh
$ lunch sdk_phone_arm64-eng
$ cd /home/u/emu-dev/external/qemu
$ export ANDROID_BUILD_TOP=$AOSP
$ objs/emulator -no-window -show-kernel -no-audio -qemu -machine virt
```

**补充**：如果嫌麻烦想避免在 AOSP 和 Emulator 两个项目之间切换以及不想执行避免 lunch，我们也可以自己定义 `ANDROID_PRODUCT_OUT`，这个环境变量用于指定 `target-specific out directory where disk images will be looked for`。具体操作，以上面的例子为例，可以用以下命令替换上面的命令输入：
```
$ cd /home/u/emu-dev/external/qemu
$ export ANDROID_BUILD_TOP=$AOSP
$ export ANDROID_PRODUCT_OUT=$AOSP/out/target/product/emulator_arm64
$ objs/emulator -no-window -show-kernel -no-audio -qemu -machine virt
```
