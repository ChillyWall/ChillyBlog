---
title: 本地编译Emacs
date: 2026-03-10
tags:
  - Emacs
categories:
  - [Editor, Emacs]
index_img: /img/categories/Emacs.png
---

Emacs 支持很多编译时可选的特性，当从包管理器或其他预编译的二进制版本无法满足你的 需求时，就需要自己在本地编译 Emacs 的源码。本篇文章记录编译 Emacs 源码的过程。

<!-- more -->

本文介绍流程在 Docker 创建的纯净的 Ubuntu 24.04 容器中进行实际操作，仅给出 Ubuntu24.04 中的命令，其余操作系统仅供参考。

## 获取源码

第一步是获取源码，我们可以直接从其[官网](https://www.gnu.org/software/emacs/download.html#gnu-linux)上获取其源码，我们选择当前最新版本 30.2 作为 后续使用的源码。

我们将其解压，即可开始编译。

## 配置编译选项及依赖

Emacs 的源码使用 Makefile 作为构建系统，使用 Autotools 实现自动化。文档是用的 `makeinfo` 需要安装 texinfo 编译前先安装基础编译工具链。通过以下命令安装：

```sh
sudo apt install build-essential autoconf texinfo
```

之后我们就可以运行编译脚本了，在源码根目录运行 `./configure` 命令，这会检查你的环 境依赖并生成最终的 Makefile，并且会显示最终被启用的特性列表。

如果出现以下错误：

```sh
checking for X... no
checking for X... false
configure: error: You seem to be running X, but no X development libraries
were found.  You should install the relevant development files for X
and for the toolkit you want, such as Gtk+ or Motif.  Also make
sure you have development files for image handling, i.e.
tiff, gif, jpeg, png and xpm.
If you are sure you want Emacs compiled without X window support, pass
  --without-x
to configure.
```

这表示 Emacs 将会不支持图形界面，只有命令行界面，同时也会不支持处理 png 等图片格 式。原因是没有检测到系统中有图形界面相关的开发库，如果是在容器中进行编译这是必然 的，因为容器内不会安装图形界面，自然也不会有相关的开发库。但哪怕是在桌面系统中， 也可能会出现这个问题，因为一般桌面只会安装运行库，而源码编译需要开发库，在 Ubuntu 中即 `-dev` 后缀的库。

在 Linux 系统上我们一般使用 GTK 作为图形库，我们安装 GTK3 的开发库，以及一些处理图片所 需的库，以及 GnuTLS，Emacs 内置的网络模块依赖于它。以及 ncurses 开发的库，用于提供终端支持。

```sh
sudo apt install libgtk-3-dev libgif-dev libxpm-dev libgnutls28-dev libncurses-dev
```

上述即编译 Emacs 的必须库，确保都安装之后我们再次运行 configure 命令，运行成功后，我们查看末尾当前特性列表的开启情况：

```text
Configured for 'x86_64-pc-linux-gnu'.

  Where should the build process find the source code?    .
  What compiler should emacs be built with?               gcc -g3 -O2
  Should Emacs use the GNU version of malloc?             no
    (The GNU allocators don't work with this system configuration.)
  Should Emacs use a relocating allocator for buffers?    no
  Should Emacs use mmap(2) for buffer allocation?         no
  What window system should Emacs use?                    x11
  What toolkit should Emacs use?                          GTK3
  Where do we find X Windows header files?                Standard dirs
  Where do we find X Windows libraries?                   Standard dirs
  Does Emacs use -lXaw3d?                                 no
  Is Emacs being built for Android?                       no
  Does Emacs use the X Double Buffer Extension?           yes
  Does Emacs use -lXpm?                                   yes
  Does Emacs use -ljpeg?                                  yes
  Does Emacs use -ltiff?                                  yes
  Does Emacs use a gif library?                           yes -lgif
  Does Emacs use a png library?                           yes -lpng16
  Does Emacs use -lrsvg-2?                                no
  Does Emacs use -lwebp?                                  yes
  Does Emacs use -lsqlite3?                               no
  Does Emacs use cairo?                                   yes
  Does Emacs use -llcms2?                                 no
  Does Emacs use imagemagick?                             no
  Does Emacs use native APIs for images?                  no
  Does Emacs support sound?                               yes
  Does Emacs use -lgpm?                                   no
  Does Emacs use -ldbus?                                  yes
  Does Emacs use -lgconf?                                 no
  Does Emacs use GSettings?                               yes
  Does Emacs use a file notification library?             yes (inotify)
  Does Emacs use access control lists?                    no
  Does Emacs use -lselinux?                               yes
  Does Emacs use -lgnutls?                                yes
  Does Emacs use -lxml2?                                  no
  Does Emacs use -lfreetype?                              yes
  Does Emacs use HarfBuzz?                                yes
  Does Emacs use -lm17n-flt?                              no
  Does Emacs use -lotf?                                   no
  Does Emacs use -lxft?                                   no
  Does Emacs use -lsystemd?                               no
  Does Emacs use -ltree-sitter?                           no
  Does Emacs use the GMP library?                         yes
  Does Emacs directly use zlib?                           yes
  Does Emacs have dynamic modules support?                yes
  Does Emacs use toolkit scroll bars?                     yes
  Does Emacs support Xwidgets?                            no
  Does Emacs have threading support in lisp?              yes
  Does Emacs support the portable dumper?                 yes
  Does Emacs support legacy unexec dumping?               no
  Which dumping strategy does Emacs use?                  pdumper
  Does Emacs have native lisp compiler?                   no
  Does Emacs use version 2 of the X Input Extension?      yes
  Does Emacs generate a smaller-size Japanese dictionary? no
```

这是一个很基础的 Emacs，很多特性没有支持，这里介绍一下比较重要的几个特性，其余特 性保持默认即可，已经是推荐选项。

### 图形系统

首先是图形系统，目前使用 X11 和 Gtk3，如果你正在使用 X11 则不用管，但如果你只用 Wayland，并且希望 Emacs 可以原生运行在 Wayland 上，可以通过加入`--with-pgtk`参数 设置。同时需要安装 Wayland 的开发库，之前安装 GTK 开发库时已经作为依赖被安装。结 果显示如下表示成功。

```text
  What window system should Emacs use?                    pgtk
```

上面 Xpm 和 X Double Buffer Extension 都是 X11 下才会生效的特性，当使用 PGTK 时 会自动关闭，无需在意。

### 图片支持

Emacs 原生支持处理图片，你可以看到上面已经包含了 gif，png，jpeg，tiff，webp 等格 式的原生支持都一起用，但是 `-lrsvg-2` 显示为 no 表示矢量图形即 svg 渲染未启用， 这是由于系统中缺少这个开发库，安装后一般会自动启用，这里建议安装并启用。

而 `-llcms2` 也是未启用，该库与图片色彩管理相关，主要影响图片浏览。该特性以来满足 时默认开启，建议安装以启用。

```sh
sudo apt install librsvg2-dev liblcms2-dev
```

在之前（Emacs 26 以前），往往使用 ImageMagick 进行图片相关的处理，如缩放等，但是 在新版本中推荐不使用它而是使用各种原生 api，比如上面的 libjpeg，libpng 等，以及 Cairo 来替代。Cairo 是一个跨平台的开源 2D 矢量图形渲染库，建议使用它而非 ImageMagick，安装 gtk 时一般已经安装了它的开发库，确保上面 `-lcairo` 为 yes。而

在 Linux 上并没有统一的图片 API 而是通过一个个动态库实现，因此 `native APIs for images` 显示为 no，这是正常的，无需理会。

### 原生编译

在 Emacs 28 之后，添加了一项新的特性成为原生编译（Native Comp），用于 ELisp 的运 行。ELisp 原先是先编译为字节码再通过虚拟机运行。而原生编译则是通过 libgccjit 将 ELisp 的字节码直接转化为机器指令，即 AOT。强烈建议将其开启。

首先安装 libgccjit，建议安装与系统默认版本 gcc 相同版本的。之后运行 configure，加入如下参数。

```sh
sudo apt install libgccjit-13-dev
./configure --with-pgtk --with-native-compilation=aot
```

### 杂项

**强烈建议**开启 sqlite3 的支持，为一些插件提供数据库支持。如果在系统中安装了 sqlite3 的开发库，该选项将会默认开启。

还有 xml2 的支持，Emacs 内置的一些功能，如浏览器，RSS 阅读器等功能，会需要 xml 解析功能，**强烈建议**安装相应的开发库，会默认开启。

`-lotf` 这一项为 no，表示不会使用 libotf 这个库来处理字体，但是我们已经启用了 HaffBuzz，其功能更加强大，即使没有也无需在意。如果系统了安装该开发库，该选项会默 认启用，启用了也没有坏处。

`-lsystemd` 这一项为 no，如果要启用需要安装对应开发库。启用后可以和 systemd 更好集 成，一般没什么用，默认启用。

`-ltree-sitter` **强烈建议启用**，安装对应开发库之后会自动启用。该特性用于 tree-sitter 支持，其用于解析代码的 AST，让 Emacs 可以理解代码的结构，非常重要。

XWidget 相关支持即通过 webkit2gtk 来实现将浏览器嵌入到 Emacs 中，webkit2gtk 非常沉重，且容易导致 Emacs 崩溃，一般没有必要启用。

还有一个参数，虽然不在特性列表上，但是可能会在末尾进行警告，与电子邮件相关。 Emacs 默认使用 movemail，这是一个过时的不安全的 POP3 客户端，Emacs 会推荐你用更安全更现代的来替代它，比如 Gnu Mailutils。一般不必在意，目前主流的 Emacs 邮件客户端这两个都不需要。

其余特性一般要么已经过时，要么没有必要，无需在意。

安装命令：

```sh
sudo apt install libsqlite3-dev libotf-dev libsystemd-dev libtree-sitter-dev
```

### 最终列表

```text
Configured for 'x86_64-pc-linux-gnu'.

  Where should the build process find the source code?    .
  What compiler should emacs be built with?               gcc -g3 -O2
  Should Emacs use the GNU version of malloc?             no
    (The GNU allocators don't work with this system configuration.)
  Should Emacs use a relocating allocator for buffers?    no
  Should Emacs use mmap(2) for buffer allocation?         no
  What window system should Emacs use?                    pgtk
  What toolkit should Emacs use?                          GTK3
  Where do we find X Windows header files?                Standard dirs
  Where do we find X Windows libraries?                   Standard dirs
  Does Emacs use -lXaw3d?                                 no
  Is Emacs being built for Android?                       no
  Does Emacs use the X Double Buffer Extension?           no
  Does Emacs use -lXpm?                                   no
  Does Emacs use -ljpeg?                                  yes
  Does Emacs use -ltiff?                                  yes
  Does Emacs use a gif library?                           yes -lgif
  Does Emacs use a png library?                           yes -lpng16
  Does Emacs use -lrsvg-2?                                yes
  Does Emacs use -lwebp?                                  yes
  Does Emacs use -lsqlite3?                               yes
  Does Emacs use cairo?                                   yes
  Does Emacs use -llcms2?                                 yes
  Does Emacs use imagemagick?                             no
  Does Emacs use native APIs for images?                  no
  Does Emacs support sound?                               yes
  Does Emacs use -lgpm?                                   no
  Does Emacs use -ldbus?                                  yes
  Does Emacs use -lgconf?                                 no
  Does Emacs use GSettings?                               yes
  Does Emacs use a file notification library?             yes (inotify)
  Does Emacs use access control lists?                    no
  Does Emacs use -lselinux?                               yes
  Does Emacs use -lgnutls?                                yes
  Does Emacs use -lxml2?                                  yes
  Does Emacs use -lfreetype?                              yes
  Does Emacs use HarfBuzz?                                yes
  Does Emacs use -lm17n-flt?
  Does Emacs use -lotf?                                   yes
  Does Emacs use -lxft?
  Does Emacs use -lsystemd?                               yes
  Does Emacs use -ltree-sitter?                           yes
  Does Emacs use the GMP library?                         yes
  Does Emacs directly use zlib?                           yes
  Does Emacs have dynamic modules support?                yes
  Does Emacs use toolkit scroll bars?                     yes
  Does Emacs support Xwidgets?                            no
  Does Emacs have threading support in lisp?              yes
  Does Emacs support the portable dumper?                 yes
  Does Emacs support legacy unexec dumping?               no
  Which dumping strategy does Emacs use?                  pdumper
  Does Emacs have native lisp compiler?                   yes
  Does Emacs use version 2 of the X Input Extension?      no
  Does Emacs generate a smaller-size Japanese dictionary? no
```

到此，所有参数配置基本结束。可以进行编译了。

在特性之外，如果要指定安装路径，只需要在 configure 中加入如下参数：

```sh
./configure --with-pgtk --with-native-compilation=aot --prefix=/path/to/install
```

## 构建部署

剩下的过程就是构建了，建议使用多线程并行编译加快速度。使用 `nproc` 命令获取核心数量。构建完成后安装，如果显示权限不足就加上 `sudo` 或者改到用户目录。

```sh
make -j$(nproc) && make install
```
