---
title: CMake进阶教程
date: 2025-05-15 16:00
tags:
  - CMake
  - C/C++
categories:
  - [Languages, CMake]
index_img: /img/categories/cmake.png
---

上一篇文章[CMake基础教程](/ChillyBlog/2025/05/11/CMake基础教程)中介绍了CMake的一些基础用法，本文是该系列的进阶教程，将会介绍一些CMake的进阶用法，帮助我们更好地管理更复杂的项目。

<!-- more -->

## CMake的进阶使用

之前已经介绍了如何使用CMake添加目标并为其配置头文件包含目录和依赖库，但并没有涉及如何使用第三方库，以及更精细的配置。本节将会在这些方面进一步深入。

上一节已经解读了[MineSweeper](https://github.com/ChillyChill/MineSweeper)的`msutils`和`ui`两个模块的CMakeLists.txt，本节将主要根据`qt_ui`模块的CMakeLists.txt来介绍。

这是`qt_ui`模块的CMakeLists.txt

```CMake
find_package(Qt6 COMPONENTS Core Gui Widgets Svg REQUIRED)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

set(SOURCES
    src/main.cpp
    src/mainwindow.cpp
    src/diff_menu.cpp
    src/cust_diff_form.cpp
    src/game_field.cpp
)

set(HEADERS
    include/mainwindow.hpp
    include/diff_menu.hpp
    include/cust_diff_form.hpp
    include/game_field.hpp
)

set(RUNTIME_LIBS Qt6::Core Qt6::Gui Qt6::Widgets Qt6::Svg)

set(RESOURCES resources.qrc)

add_executable(MineSweeper-qt ${SOURCES} ${HEADERS} ${RESOURCES})
target_link_libraries(MineSweeper-qt PRIVATE msutils ${RUNTIME_LIBS})
target_include_directories(MineSweeper-qt PRIVATE include)

install(TARGETS MineSweeper-qt DESTINATION bin)

if(${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
    # fill in the path to your Qt installation, like C:/Qt/6.8.1/mingw_64
    set(QT_INSTALL_PATH "C:/Qt/6.8.1/mingw_64")
    install(
        FILES ${QT_INSTALL_PATH}/plugins/platforms/qwindows.dll
        DESTINATION bin/plugins/platforms
    )
    install(
        FILES ${QT_INSTALL_PATH}/plugins/iconengines/qsvgicon.dll
        DESTINATION bin/plugins/iconengines
    )
    set_target_properties(MineSweeper-qt PROPERTIES WIN32_EXECUTABLE TRUE)
    install(IMPORTED_RUNTIME_ARTIFACTS ${RUNTIME_LIBS} DESTINATION bin)
endif()
```

### 使用第三方库

上一篇文章中介绍了如何链接库，但链接的只是我们自己创建的库，而真正开发时自然少不了需要使用第三方库的时候。

使用第三方库，最常用的还是使用`find_package`函数来查找。

其基础的语法是

```CMake
find_package(<PackageName> [<version>] [REQUIRED] [COMPONENTS <components>...])
```

其寻找库的方式分为两种，模块模式和配置模式。

#### 模块模式

模块模式通常使用外部提供而不是库本身提供的`Find<PackageName>.cmake`文件来按照特定方式去寻找模块。会在所有在变量`CMAKE_MODULE_PATH`中列出的目录中寻找。如果我们要使用非标准方式提供的模块文件，则只需要将其所在目录添加到该变量中即可。

CMake会设置相应的变量来存储查找结果，变量比较多，详情可以参阅[官方文档](https://cmake.com.cn/cmake/help/latest/manual/cmake-developer.7.html#standard-variable-names)，这里只介绍几个

- `<PackageName>_FOUND`：是否找到
- `<PackageName>_INCLUDE_DIRS`：模块中所有包含目录的最终集合
- `<PackageName>_LIBRARIES`：模块中所有的库

如果是按照这种方式查找的，我们可以通过这三个变量判断库有没有找到，并将其包含目录和库全部添加到我们的目标上。

这种模式已经被逐渐弃用，现在主流的是使用配置模式。其使用更符合Modern CMake的基于目标的实现形式。

#### 配置模式

在此模式下，CMake 搜索名为`<lowercasePackageName>-config.cmake`或`<PackageName>Config.cmake`的文件。如果指定了版本详细信息，它还会查找`<lowercasePackageName>-config-version.cmake`或`<PackageName>ConfigVersion.cmake`文件，去做版本相关的处理。这些文件通常在`lib/cmake/PackageName`下面。

该模式下，允许包将自己分为多个组件(Component)，一个组件相当于一个子包，其中也可以包含多个目标。一个组件共享一个配置文件，便于选择性安装和包配置。

> 组件不等于目标，一个组件可能含有多个目标，上面Qt6::Gui是Qt6 Gui组件中的一个目标，虽然同名，但不是一个概念。一些比较小的库往往不使用组件，这时候如果以为组件就是目标而擅自在`find_package`中加上`COMPONENTS target REQUIRED`可能会导致找不到组件而报错。

一个包往往会有一个以包名命名的名称空间，引用包中一个目标一般可以通过`PackageName::Component`实现。而如果包中添加了组件，那么每一个组件可能也有各自名称空间，但也有可能设置为和包共用名称空间。这由库的开发者决定。

上面的CMakeLists.txt中使用了`find_package`函数来查找Qt6，便是使用了配置模式。Qt6是个庞大的框架，我们使用其中四个组件，要确保这些组件被找到的方式便是加上`COMPONENTS component1 ...`，这些组件如果有一个没有找到，则视为这个包没有找到。最后的`REQUIRED`表示这是必须的，找不到无法完成配置。

### 控制语句

这是`msutils`模块的CMakeLists.txt的后半部分内容

```CMake
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "MSVC")
    target_compile_options(msutils PUBLIC /W4)
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
    target_compile_options(msutils PUBLIC -Wall -Wextra -Wpedantic)
elseif(${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
    target_compile_options(msutils PUBLIC -Wall -Wextra -Wpedantic)
endif()
```

由于C/C++有众多编译器，每个编译器支持的编译选项也不同，因此我们通常需要根据编译器的不同来选择不同的编译参数。上面部分就使用了if语句来分情况添加编译选项。

CMake的if语句写法大致如上，通过`STREQUAL`函数来比较两个字符串是否相同，内置变量`CMAKE_CXX_COMPILER_ID`表示C++使用的编译器名字。具体列表见[官网](https://cmake.com.cn/cmake/help/latest/variable/CMAKE_LANG_COMPILER_ID.html)。

我们还可以像`qt_ui`的CMakeLists.txt中那样通过`CMAKE_SYSTEM_NAME`变量判断操作系统（交叉编译的话，该值应是目标系统）。具体列表见[官网](https://cmake.com.cn/cmake/help/latest/variable/CMAKE_SYSTEM_NAME.html)。

### 编译选项(compile option)

在编译时，编译器提供了很多选项（options），我们可以通过CMake为目标设置特定的选项。

我们使用`target_compile_options`函数来设置编译选项。语法上与之前介绍的两个没有什么区别，只是后面的列表中是编译器的参数，无需使用字符串，直接写出来空格隔开。

上面我们便为`msutils`设置了编译时打印所有warnings的选项。

在`msutils`中我们使用了`PUBLIC`作用域，因此链接了该静态库的其他两个可执行文件也都会开启这些选项。

### 安装(install)

CMake要配置安装一般使用`install`命令，该命令可以安装多种对象，本文将介绍以下这些。

```CMake
install(TARGETS <target>... [...])
install(IMPORTED_RUNTIME_ARTIFACTS <target>... [...])
install({FILES | PROGRAMS} <file>... [...])
install(DIRECTORY <dir>... [...])
```

安装命令较为复杂，但大多都是写开发库和框架时需要的，本文中不会涉及，因此删去大部分参数，只保留少数常用的参数。

比如所有的安装命令都有指定文件访问权限的参数，本文中都将其删去。要了解其详细用法，请参考[官网文档](https://cmake.com.cn/cmake/help/latest/command/install.html)

#### 目标安装

这是最常用到的命令，将目标安装到指定路径下。路径若使用相对路径，则相对`CMAKE_INSTALL_PREFIX`变量的值，该值在linux系统下一般默认为`/usr`或`/usr/local`。

基础语法如下

```CMake
install(TARGETS <target>...
        [<artifact-option>...]
        [<artifact-kind> <artifact-option>...]...
        )
```

其中`<artifact-option>`包含很多内容，我们主要使用`DESTINATION <dir>`用来指定输出目录。但对于一般的静态库、动态库或可执行文件，这并不是必要的，因为一般默认值就是推荐值。

其中`<artifact-kind>`表示目标中对象的类型，亦包括很多种，但我们主要介绍三种：

1. `ARCHIVE`：静态库或动态库的导入库。
2. `LIBRARY`：动态库（除了DLL）。
3. `RUNTIME`：可执行文件或DLL。

第一次设置是为所有类型设置。第二次设置是为指定类型单独设置选项。

`qt_ui`中便使用此命令安装`MineSweeper-qt`目标，并指定了输出目录为bin（实际上并不需要指定，因为这就是默认值）。

windows下动态库默认安装到bin，和可执行文件在同一个目录，因此不会有找不到动态库的问题。在linux默认安装到lib，你也可以强行指定为bin，这样省去了配置的过程。

#### 动态库依赖安装

我们的程序若使用了动态库，且该动态库没有安装到系统全局中，我们可能需要在安装自己的程序时，将其依赖的动态库也安装到对应位置上。

如果这个动态库是我们项目中的一个目标，自然很简单，直接将其作为目标安装即可。但如果是通过`find_package`导入的第三方库，是无法通过这个方法安装的。

比如上面`qt_ui`在windows下时，由于一般windows下面安装qt都不会将qt安装到系统全局中，所以我们必须在安装程序的同时安装其依赖的qt相关动态库文件。

CMake提供了相应的安装命令，其基础语法如下

```CMake
install(IMPORTED_RUNTIME_ARTIFACTS <target>...
        [[LIBRARY|RUNTIME|FRAMEWORK|BUNDLE]
         [DESTINATION <dir>]
        ] [...]
        )
```

整体与安装目标的命令一样，该命令用于安装运行时工件，最常用的便是用于安装动态库依赖（该命令不会安装动态库的导入库）。

上面`qt_ui`便通过此命令安装了qt6相关动态库。

#### 文件安装

如果要安装一些文件，可以使用命令

```CMake
install(<FILES|PROGRAMS> <file>...
        TYPE <type> | DESTINATION <dir>
        [RENAME <name>])
```

`FILES`和`PROGRAMS`使用相同的形式，`PROGRAMS`一般用于安装脚本文件，与`FILES`的区别在于其可以设置执行权限。

我们应该在指定`TYPE`和`DESTINATION`中二选一，若指定了类型，则会根据类型决定安装位置。常用类型有

| Type      | Destination |
| --------- | ------------------- |
| `BIN`     | `bin`               |
| `LIB`     | `lib`               |
| `INCLUDE` | `include`           |
| `SYSCONF` | `etc`               |

上面`qt_ui`在windows下将会通过此命令安装Qt6的一些必须的plugins到安装路径中。（可以通过指定环境变量让程序找到这些plugins，但直接自己装上一劳永逸）。

#### 文件夹安装

要安装文件夹的基础命令如下

```CMake
install(DIRECTORY dirs...
        TYPE <type> | DESTINATION <dir>
        [FILES_MATCHING]
        [PATTERN <pattern> | REGEX <regex>] [...])
```

`DIRECTORY`可以分别设置目录权限和文件权限，并且可以使用匹配机制只安装文件夹下匹配到的文件。

比如如果我们想要将`qt_ui`模块下的resources文件夹中的svg图片都安装到指定位置（原项目使用qrc文件，通过rcc工具将资源文件直接打包进可执行文件中）可以这样写

```CMake
install(DIRECTORY resources DESTINATION share/MineSweeper
        FILES_MATCHING PATTERN "*.svg")
```

### 编译特性(compile feature)

编译特性是编译器支持的特性，比如C++11、C++14、C++17标准，或者对constexpr、decltype、final的单独支持等。我们可以通过`target_compile_features`函数来设置编译器特性。

```CMake
target_compile_features(<target> <PRIVATE|PUBLIC|INTERFACE> <feature> [...])
```

编译器支持的特性可以通过变量`CMAKE_CXX_COMPILE_FEATURES`或`CMAKE_C_COMPILE_FEATURES`获取。所有的已知的特性都可以通过[`CMAKE_CXX_KNOWN_FEATURES`](https://cmake.com.cn/cmake/help/latest/prop_gbl/CMAKE_CXX_KNOWN_FEATURES.html)或[`CMAKE_C_KNOWN_FEATURES`](https://cmake.com.cn/cmake/help/latest/prop_gbl/CMAKE_C_KNOWN_FEATURES.html)来查询。

比如要为`msutils`启用C++20的特性，可以这样写`target_compile_features(msutils PRIVATE cxx_std_20)`。

单个特性主要是C++11和C++17的，为其单独设置在现在已经没有太大的必要。因此这个命令主要用于指定标准。最后实际上是通过传递`-std=`标志实现。

### 编译定义(compile definition)

编译定义主要指宏的定义。为C/C++源文件动态的注入宏的值。基础命令如下

```CMake
target_compile_definitions(<target>
  <INTERFACE|PUBLIC|PRIVATE> [items1...]
  [<INTERFACE|PUBLIC|PRIVATE> [items2...] ...])
```

比如我们可以通过`target_compile_definitions(msutils PRIVATE FOO=1)`来为`msutils`添加一个宏定义。该宏定义会在预编译时注入到源文件中。

也可以使用`add_compile_definitions`函数来添加。

### 属性(property)

CMake中属性可以影响到方方面面，从编译到构建过程到测试等都会有影响，CMake的所有信息基本都保存在各种对象的属性中。属性分为多种，有全局属性、目录属性、目标属性测试属性、源文件属性等等。

比如我们通过各种命令为目标添加的包含目录，最终都会被存储到目标的`INCLUDE_DIRECTORIES`属性中去。链接的库都会被记录到`LINK_LIBRARIES`属性中去。

还有一些其他的，起到特定作用的属性。比如上面`qt_ui`中当操作系统是windows时，会将`MineSweeper-qt`的`WIN32_EXECUTABLE`属性设为`TRUE`。这个属性为TRUE时会为程序构建一个带有WinMain入口的可执行文件，这使得其成为GUI程序而不是控制台程序。

读取和设置单个属性的命令为`get_property()`和`set_property()`。或者使用`set_target_properties`, `set_source_files_properties`, `set_tests_properties`, `set_directory_properties`为单个目标、源文件、测试、目录设置属性。将`set`改为`get`即对应的读取属性的命令。

这里只介绍最为常用的设置目标属性，其语法如下。

```CMake
set_target_properties(<targets> ...
                      PROPERTIES <prop1> <value1>
                      [<prop2> <value2>] ...)
```

全部的属性列表见[官网](https://cmake.com.cn/cmake/help/latest/manual/cmake-properties.7.html#properties-on-targets)。其他命令的具体用法可自行到[CMake官网](https://cmake.com.cn/cmake/help/latest/manual/cmake-commands.7.html)查询。

### 输出信息

有时我们想要在CMake的输出中查看一些信息，可以使用`message`命令。该命令可以输出一些信息到终端中，语法如下

```CMake
message([<mode>] "message text" ...)
```

`<mode>`中可以指定输出的类型，常用的有`STATUS`、`WARNING`、`AUTHOR_WARNING`、`SEND_ERROR`、`FATAL_ERROR`、`VERBOSE`等。具体列表见[官网](https://cmake.com.cn/cmake/help/latest/command/message.html)。

比如我们想要查看`MineSweeper-qt`的`LINK_LIBRARIES`属性的值，可以这样写

```CMake
get_property(ms_qt_link_libs TARGET MineSweeper-qt PROPERTY LINK_LIBRARIES)
message(VERBOSE "LINK_LIBRARIES: ${ms_qt_link_libs}")
```

生成构建系统时加上参数`--log-level VERBOSE`，cmake就会输出上面的信息。如果没有输出，删除原本的构建系统重新运行再看看。

输出结果中应有这一行

```text
-- LINK_LIBRARIES: msutils;Qt6::Core;Qt6::Gui;Qt6::Widgets;Qt6::Svg
```

## 总结

以上就是本文全部内容。目前介绍了如何使用CMake进行更细致的配置，如何使用第三方库，如何将程序安装到目标位置等等。

但如果想要开发的不是一般工程，而是开发库，甚至是大型的框架，这些显然是不够的。

但我掌握的内容暂时就到这里了，我也正在学习中，后续随着我的学习，可能还会再写一些专题教程。
