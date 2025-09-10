# LVGL

在linux下使用lvgl编写gui程序，使用lv_port_linux编译出动态库，简化在linux的开发环境流程。

## 制作Debian包
### 1.安装环境

```shell
sudo apt update
sudo apt install pkg-config
sudo apt install libwayland-dev libxkbcommon-dev libwayland-bin wayland-protocols libdrm-dev
```

### 2.修改lv_port_linux的一些默认配置

CMakeLists.txt,，方便编译动态库。

```shell
 # Link LVGL with external dependencies - Modern CMake/CMP0079 allows this
-target_link_libraries(lvgl PUBLIC ${PKG_CONFIG_LIB} m pthread)
+target_link_libraries(lvgl PUBLIC ${PKG_CONFIG_LIB} m pthread rt)
```
编译开关

LV_USE_PRIVATE_API，打包dev开发包的时候，私有头文件也要包含，否则会找不到头文件。

LV_USE_WAYLAND，启用wayland后端

LV_WAYLAND_WINDOW_DECORATIONS，开启这个，可以使得wayland窗口管理器管理app，否则app无法拖动。

LV_USE_LINUX_DRM，使用DRM后端

```shell
 /** Include `lvgl_private.h` in `lvgl.h` to access internal data and functions by default */
 #ifndef LV_USE_PRIVATE_API
-    #define LV_USE_PRIVATE_API  0
+    #define LV_USE_PRIVATE_API  1
 #endif

 
 /** Use Wayland to open a window and handle input on Linux or BSD desktops */
-#define LV_USE_WAYLAND          0
+#define LV_USE_WAYLAND          1
 #if LV_USE_WAYLAND

/**< When LV_WAYLAND_USE_DMABUF is disabled, only LV_DISPLAY_RENDER_MODE_PARTIAL is supported*/
-    #define LV_WAYLAND_WINDOW_DECORATIONS   0    /**< Draw client side window decorations only necessary on Mutter/GNOME. Not supported using DMABUF*/
+    #define LV_WAYLAND_WINDOW_DECORATIONS   1    /**< Draw client side window decorations only necessary on Mutter/GNOME. Not supported using DMABUF*/
     #define LV_WAYLAND_WL_SHELL             0    /**< Use the legacy wl_shell protocol instead of the default XDG shell*/
 #endif

 /** Driver for /dev/dri/card */
-#define LV_USE_LINUX_DRM        0
+#define LV_USE_LINUX_DRM        1
 
 #if LV_USE_LINUX_DRM
```

3.创建Debian构建文件

```shell
cd lv_port_linux && mkdir -p debian && cd debian
```

添加control文件

```shell
Source: lvgl
Section: libs
Priority: optional
Maintainer: LVGL Maintainer <maintainer@example.com>
Build-Depends: debhelper-compat (= 13), cmake, pkg-config, libwayland-dev, wayland-protocols, libxkbcommon-dev, libdrm-dev
Standards-Version: 4.6.2
Homepage: https://lvgl.io

Package: liblvgl9
Architecture: any
Depends: ${shlibs:Depends}, ${misc:Depends}
Description: LittlevGL runtime library
 LVGL graphics library runtime shared objects.

Package: liblvgl-dev
Architecture: any
Depends: liblvgl9 (= ${binary:Version}), ${misc:Depends}
Description: LittlevGL development files (headers, pkg-config)
 Development headers, shared libs and pkg-config for LVGL.
```



添加rules

```shell
#!/usr/bin/make -f
export DH_VERBOSE=1

%:
	dh $@ --buildsystem=cmake

#编译动态库
override_dh_auto_configure:
	# 可选：若需从 lv_conf.defaults 生成 lv_conf.h
	# python3 lvgl/scripts/generate_lv_conf.py .
	bash ./scripts/gen_wl_protocols.sh ./protocols
	# 解决 Wayland 协议生成代码中符号可见性为 hidden 导致 DSO 链接失败
	sed -i 's/^#define WL_PRIVATE __attribute__ ((visibility("hidden")))/#define WL_PRIVATE/' ./protocols/wayland_xdg_shell.c || true
	dh_auto_configure --builddirectory=build -- \
	  -DCMAKE_INSTALL_PREFIX=/usr \
	  -DBUILD_SHARED_LIBS=ON \
	  -DLIB_INSTALL_DIR=lib/${DEB_HOST_MULTIARCH} \
	  -DCMAKE_C_FLAGS='-DWL_PRIVATE=' \
	  -DCMAKE_EXE_LINKER_FLAGS='-Wl,--export-dynamic' \
	  -DLV_BUILD_EXAMPLES=ON \
	  -DLV_BUILD_DEMOS=ON

override_dh_auto_build:
	# Wayland scanner generates hidden symbols; neutralize before linking
	sed -i 's/^#define WL_PRIVATE __attribute__ ((visibility("hidden")))/#define WL_PRIVATE/' build/protocols/wayland_xdg_shell.c || true
	dh_auto_build --builddirectory=build -- -j$(nproc)

override_dh_auto_install:
	dh_auto_install --builddirectory=build --destdir=$(CURDIR)/debian/tmp
	# 等同于：DESTDIR=$(CURDIR)/debian/tmp cmake --install build

override_dh_auto_clean:
	# 保持与 out-of-source 一致的清理
	dh_auto_clean --builddirectory=build || true

override_dh_auto_test:
```

添加install文件

liblvgl9.install

```
usr/lib/*/liblvgl.so.*
usr/lib/*/liblvgl_demos.so.*
usr/lib/*/liblvgl_examples.so.*
usr/lib/*/liblvgl_thorvg.so.*
```
liblvgl-dev.install
```shell
usr/include/lvgl/
usr/share/pkgconfig/lvgl.pc
usr/lib/*/liblvgl.so
usr/lib/*/liblvgl_demos.so
usr/lib/*/liblvgl_examples.so
usr/lib/*/liblvgl_thorvg.so
usr/lib/liblvgl_linux.a
```

####  liblvgl_linux.a为什么不改成动态库？

- 目标在 CMake 中被明确声明为 STATIC：项目把平台封装/后端适配层做成了内部静态库（add_library(lvgl_linux STATIC ...)），默认不会随 -DBUILD_SHARED_LIBS=ON 变成动态库。要修改，则会大量的地方修改适配。

- 仅供内部链接、非稳定 ABI：lvgl_linux 属于本仓库的“Linux 适配层”，主要被可执行程序或 liblvgl.so 内部使用，没承诺对外 ABI 稳定，不适合作为独立 .so 提供。

- Wayland 生成代码的符号可见性问题：wayland-scanner 生成的 wayland_xdg_shell.c 使用 WL_PRIVATE 把接口标记为隐藏，直接做成 .so 会出现“hidden symbol is referenced by DSO”的链接问题，需要额外处理（我们在打包时通过 -DWL_PRIVATE= 和 --export-dynamic 才解决）。

- 避免循环依赖与依赖膨胀：如果把 lvgl_linux 做成共享库，很容易与 liblvgl.so、liblvgl_examples.so、liblvgl_demos.so 形成相互引用，导致复杂的运行时依赖和 shlibdeps 噪音。当前方案是把对外 .so 限定在核心和演示库，其它平台层静态链接使用。

#### liblvgl.so 和 liblvgl_linux.a 的区别?

##### liblvgl.so- LVGL 核心库

主要功能：

- LVGL 核心功能：包含所有 LVGL 的核心 API 和功能
- 对象系统：lv_obj_* 系列函数，如 lv_obj_create, lv_obj_add_flag 等
- 绘图系统：lv_draw_* 系列函数，如 lv_draw_triangle, lv_draw_triangle_dsc_init 等
- 字体系统：lv_font_* 系列函数和字体数据，如 lv_font_montserrat_24 等
- 显示驱动接口：lv_linux_drm_create, lv_wayland_window_create 等 Linux 特定驱动函数
- 输入设备接口：lv_indev_* 系列函数
- 组管理：lv_group_* 系列函数

##### liblvgl_linux.a - Linux 平台特定库

主要功能：

- 驱动后端管理：driver_backends_* 系列函数
- 后端初始化：backend_init_* 系列函数
- Wayland 协议接口：包含 Wayland XDG 协议的接口定义
- 平台特定实现：Linux 特定的驱动初始化和运行循环

### 3.构建deb包

```shell
cd lv_port_linux/
sudo apt build-dep .
dpkg-buildpackage -us -uc -tc  -b
```

生成三个包：

liblvgl9_9.3.0-1_amd64.deb，带动态库。

liblvgl-dev_9.3.0-1_amd64.deb，带头文件和liblvgl_linux.a，里面的动态库链接到liblvgl9的动态库。

liblvgl9-dbgsym_9.3.0-1_amd64.deb，带动态库调试符号。

理论上来说，开发接口是固定的，因此liblvgl-dev名字中是不带版本9，而提供的动态库liblvgl9则需要带版本9，最新liblvgl-dev通过对liblvgl9的依赖，保证liblvgl-dev可以使用新的库。



### 4.编译测试

先安装开发环境

```shell
sudo dpkg -i ../liblvgl9_9.3.0-1_amd64.deb  ../liblvgl-dev_9.3.0-1_amd64.deb
```

进入example编译例子

```shell
cd lv_port_linux/example/
cmake -B build -S .
make -C build -j
#运行例子
./bin/main
```

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="pic/lvgl.gif">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">example例子</div>
</center>
