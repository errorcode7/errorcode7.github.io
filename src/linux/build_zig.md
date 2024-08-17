## 源码编译zig

### 添加llvm18的源

zig要求这个版本

```shell
# uos v20
deb http://apt.llvm.org/buster/ llvm-toolchain-buster main
deb-src http://apt.llvm.org/buster/ llvm-toolchain-buster main
# 18 
deb http://apt.llvm.org/buster/ llvm-toolchain-buster-18 main
deb-src http://apt.llvm.org/buster/ llvm-toolchain-buster-18 main
# 19 
deb http://apt.llvm.org/buster/ llvm-toolchain-buster-19 main
deb-src http://apt.llvm.org/buster/ llvm-toolchain-buster-19 main

# deepin v23
deb http://apt.llvm.org/bookworm/ llvm-toolchain-bookworm main
deb-src http://apt.llvm.org/bookworm/ llvm-toolchain-bookworm main
# 18 
deb http://apt.llvm.org/bookworm/ llvm-toolchain-bookworm-18 main
deb-src http://apt.llvm.org/bookworm/ llvm-toolchain-bookworm-18 main
# 19 
deb http://apt.llvm.org/bookworm/ llvm-toolchain-bookworm-19 main
deb-src http://apt.llvm.org/bookworm/ llvm-toolchain-bookworm-19 main
```

```shell
https://apt.llvm.org/
wget -qO- https://apt.llvm.org/llvm-snapshot.gpg.key | sudo tee /etc/apt/trusted.gpg.d/apt.llvm.org.asc
```

> https://apt.llvm.org/

### 配置编译环境

```shell
sudo apt install -y build-essential cmake clang-18 libclang-18-dev libclang-cpp18-dev llvm-18 llvm-18-dev lld-18 liblld-18-dev libpolly-18-dev libllvm18
```

```shell
alias llvm-config=/lib/llvm-18/bin/llvm-config
llvm-config --cxxflags --ldflags --system-libs --libs core
```

> https://github.com/ziglang/zig/issues/419

### 下载zig源码编译

在build目录构建，方便删除

```shell
git clone https://mirror.ghproxy.com/https://github.com/ziglang/zig.git
cd zig
mkdir build
cd build
cmake ..  -DCMAKE_PREFIX_PATH=/usr/lib/llvm-18  -DZIG_STATIC_LLVM=ON
make
```

>  https://github.com/ziglang/zig/wiki/Building-Zig-From-Source

以上步骤为了验证是否能正常打包。



## 构建debian包

###  准备环境

```shell
sudo apt install dh-make
```



### 制作debian文件

生成debian目录

```shell
#移除编译build目录，否则--createorig参数创建源码包会包含这些
rm -rf build

#获取最新tag信息,用于作为debian版本
ver=$(git describe --tags --abbrev=0)
git checkout $ver
dh_make  -p zig_$ver  -s -y --createorig
rm -f debian/*.ex
```
编辑debian内的文件,关键是添加Build-Depends，方便`apt build-dep .`自动下载打包依赖。
如果有修改，需要`dch -i`命令添加changelog

debian/control文件内容。
```shell
Source: zig
Section: devel
Priority: optional
Maintainer: errorcode7 <errorcode7@qq.com>
Build-Depends: debhelper (>= 11),dh-make,dpkg-dev,build-essential,cmake,clang-18,libclang-18-dev,libclang-cpp18-dev,llvm-18,llvm-18-dev,lld-18,liblld-18-dev,libpolly-18-dev,libllvm18
Standards-Version: 4.1.3
Homepage: https://ziglang.org/
Vcs-Git: https://github.com/ziglang/zig

Package: zig
Architecture: any
Depends: ${shlibs:Depends}, ${misc:Depends}
Description: Zig is a general-purpose programming language and toolchain for maintaining robust, optimal and reusable software.
```
> https://github.com/errorcode7/zig/tree/master/debian

dpkg-buildpackage打包

```shell
# 因为有Cmake文件，直接构建即可
rm -rf  .zig-cache/
dpkg-buildpackage -uc -us -tc 
```

可能会提示gpg签名失败，不要紧，已经打包好了。

```shell
dpkg -c ../zig_*.deb
```

