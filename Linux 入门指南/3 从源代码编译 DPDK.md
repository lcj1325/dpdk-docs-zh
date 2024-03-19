# 3. 从源代码编译 DPDK

## 3.1.解压DPDK并浏览源代码

首先，解压存档并移动到未压缩的 DPDK 源目录：

```
tar xJf dpdk-<version>.tar.xz
cd dpdk-<version>
```

DPDK由几个目录组成，包括：
- doc：DPDK 文档
- license：DPDK许可证信息
- lib：DPDK库的源代码
- drivers：DPDK轮询模式驱动程序的源代码
- app：DPDK应用程序的源代码（自动测试）
- examples：DPDK应用示例源代码
- config、buildtools：框架相关的脚本和配置
- usertools：供 DPDK 应用程序最终用户使用的实用程序脚本
- devtools：供 DPDK 开发人员使用的脚本
- kernel：某些操作系统所需的内核模块

## 3.2.全系统编译和安装DPDK

可以使用 `meson` 和 `ninja` 工具在您的系统上配置、构建和安装 DPDK。

### 3.2.1. DPDK配置

要配置 DPDK 构建，请使用：

```
meson setup <options> build
```

其中“build”是所需的输出构建目录，“<options>”可以为空，也可以为多个 meson 或特定于 DPDK 的构建选项之一，如本节后面所述。配置过程将总结要构建和安装的 DPDK 库和驱动程序，以及禁用的每个项目的原因。例如，此信息可用于识别驱动程序所需的任何丢失的包。

配置完成后，构建并安装 DPDK 系统范围内的使用：

```
cd build
ninja
meson install
ldconfig
```

上面的最后两个命令通常需要以 root 身份运行，meson 安装步骤将构建的对象复制到其最终的系统范围位置，最后一步导致动态加载器 ld.so 更新其缓存以考虑 新对象。

***note: 在某些 Linux 发行版上，例如 Fedora 或 Redhat，`/usr/local` 中的路径不在加载程序的默认路径中。因此，在这些发行版上，在运行 ldconfig 之前，应将 `/usr/local/lib` 和 `/usr/local/lib64` 添加到 `/etc/ld.so.conf.d/` 中的文件中。***

### 3.2.2.调整构建选项

DPDK 有许多选项可以在构建配置过程中进行调整。可以通过在配置的构建文件夹中运行 `meson configure` 来列出这些选项。其中许多选项来自“meson”工具本身，可以在 [Meson Website](https://mesonbuild.com/Builtin-options.html) 上看到记录。

例如，要将构建类型从默认的“debugoptimized”更改为常规“debug”构建，您可以：
- 最初配置构建文件夹时将 `-Dbuildtype=debug` 或 `--buildtype=debug` 传递给meson
- 初始meson运行后，在构建文件夹内运行 `meson configure -Dbuildtype=debug`。

其他选项特定于 DPDK 项目，但可以类似地进行调整。“platform”选项指定一组将使用的配置参数。有效值为：
- `-Dplatform=native` 将根据构建机器定制配置。
- `-Dplatform=generic` 将使用适用于与构建机器具有相同架构的所有机器的配置。
- `-Dplatform=<SoC> `将使用针对特定 SoC 优化的配置。请查阅 `config/arm/meson.build` 中的“socs”字典以了解支持哪些 SoC。

默认情况下，指令集将根据以下规则自动设置：
- `-Dplatform=native` 将 `cpu_instruction_set` 设置为 `native`，这会将 `-march` (x86_64)、`-mcpu` (ppc)、`-mtune` (ppc) 配置为 `native`。
- `-Dplatform=generic` 将 `cpu_instruction_set` 设置为 `generic`，它将 `-march` (x86_64)、`-mcpu` (ppc)、`-mtune` (ppc) 配置为 DPDK 所需的通用最小基线。

要覆盖将使用的指令集，请将 `cpu_instruction_set` 参数设置为您选择的指令集（例如 `corei7`、`power8` 等）。

Arm 构建中不使用 `cpu_instruction_set`，因为在没有其他参数的情况下设置指令集会导致较差的构建。定制 Arm 构建的方法是使用上面提到的 `-Dplatform=<SoC>` 来构建 SoC。

由 `platform`  参数确定的值可以被覆盖。例如，要将 `max_lcores` 值设置为 256，您可以：
- 最初配置构建文件夹时将 `-Dmax_lcores=256` 传递给 meson
- 初始 `meson` 运行后，在构建文件夹内运行 `meson configure -Dmax_lcores=256`

`examples` 目录中的一些 DPDK 示例应用程序也可以作为 meson build 的一部分自动构建。为此，请将要构建的示例的逗号分隔列表传递给 -Dexamples meson 选项，如下所示：

```
meson setup -Dexamples=l2fwd,l3fwd build
```

与其他 meson 选项一样，也可以使用build目录中的 meson configure 进行初始配置后设置。还有一个特殊值“all”，用于请求构建在当前系统上满足依赖关系的所有示例应用程序。当 -Dexamples=all 设置为 meson option 时，meson 将检查每个示例应用程序以查看是否可以构建它，并将所有可以构建的添加到 ninja build configuration file 中的任务列表中。

### 3.2.3.在 64 位系统上构建 32 位 DPDK

要在 64 位操作系统上构建 DPDK 的 32 位副本，应将 `-m32` 标志传递给编译器和链接器以强制生成 32 位对象和二进制文件。这可以通过在环境中设置 CFLAGS 和 LDFLAGS 或使用 `-Dc_args=-m32` 和 `-Dc_link_args=-m32` 将值传递给 meson 来完成。为了正确识别和使用任何依赖包，还必须将 pkg-config 工具配置为在适当的目录中查找 32 位库的 .pc 文件。这是通过将 `PKG_CONFIG_LIBDIR` 设置为适当的路径来完成的。

假设安装了相关的 32 位开发包（例如 32 位 libc），可以在 RHEL/Fedora 系统上使用以下 meson 命令来配置 32 位构建：

```
PKG_CONFIG_LIBDIR=/usr/lib/pkgconfig \
    meson setup -Dc_args='-m32' -Dc_link_args='-m32' build
```

对于 Debian/Ubuntu 系统，等效命令是：

```
PKG_CONFIG_LIBDIR=/usr/lib/i386-linux-gnu/pkgconfig \
    meson setup -Dc_args='-m32' -Dc_link_args='-m32' build
```

一旦配置了构建目录，就可以使用 `ninja` 来编译 DPDK，如上所述。

### 3.2.4.使用已安装的 DPDK 构建应用程序

当在系统范围内安装时，DPDK 会提供一个 pkg-config 文件 `libdpdk.pc` 供应用程序在其构建过程中进行查询。建议使用 pkg-config 文件，而不是将 DPDK 的参数 (cflags/ldflags) 硬编码到应用程序构建过程中。

有关如何查询和使用 pkg-config 文件的示例，可以在 DPDK 附带的每个示例应用程序的 `Makefile` 中找到。下面显示了一个简化的示例片段，其中目标二进制文件名称已存储在变量 `$(APP)` 中，该构建的源存储在 `$(SRCS-y)` 中。

```
PKGCONF = pkg-config

CFLAGS += -O3 $(shell $(PKGCONF) --cflags libdpdk)
LDFLAGS += $(shell $(PKGCONF) --libs libdpdk)

$(APP): $(SRCS-y) Makefile
        $(CC) $(CFLAGS) $(SRCS-y) -o $@ $(LDFLAGS)
```

***note: 与旧版 DPDK 中的 make 构建系统不同，meson 系统并非设计为直接从构建目录使用。相反，建议将其安装在系统范围内或安装到用户主目录中的已知位置。可以使用 –prefix meson 选项设置安装位置（默认值：/usr/local）***

使用 meson 作为构建系统的简单 DPDK 应用程序的等效构建方案如下所示：

```
project('dpdk-app', 'c')

dpdk = dependency('libdpdk')
sources = files('main.c')
executable('dpdk-app', sources, dependencies: dpdk)
```