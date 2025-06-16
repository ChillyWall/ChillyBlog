---
title: CMake专题教程-导入导出
date: 2025-05-16 22:00
tags:
  - CMake
  - C/C++
categories:
  - [Languages, CMake]
index_img: /img/categories/cmake.png
---

本文将介绍如何直接导入二进制文件，或导出项目，使得其他项目可以通过CMake的`find_package`直接导入我们的项目并基于其开发。

<!-- more -->

虽然CMake已经是事实上的标准，但我们有时还是会遇到只有二进制和头文件还没有模组文件和配置文件的情况（比如windows下我们通过cmake使用EasyX时就只能手动导入）。这时我们可以通过cmake将其导入，使其成为一个`IMPORTED`目标。

而实际上，库的配置文件就是一个cmake脚本，其可以将库导入成为导入库目标。`find_package`会找到这些配置文件并运行。

现在我们将先介绍如何导入，这部分比较简单。之后介绍如何导出项目。

## 导入

导入分为两种情况，导入可执行文件和导入库。一下将分别介绍。

### 导入可执行文件

导入可执行文件也使用`add_executable`命令，语法如下

```CMake
add_executable(<name> IMPORTED [GLOBAL])
```

默认情况下，导入目标仅在当前和子文件夹中生效，要使其全局生效，可以在最后添加`GLOBAL`关键字。

创建导入目标之后，要为其设置可执行文件的路径，一般直接设置其`IMPORTED_LOCATION`属性的值（需要绝对路径，我们可以使用cmake一些变量的值构建出来）。

```CMake
set_target_properties(<target> PROPERTIES IMPORTED_LOCATION path/to/myexe)
```

导入的可执行目标在`add_custom_command()`一类的命令中引用比较方便，这些命令需要需要一个可执行文件来进行一些操作，比如调用clang-format对头文件进行排版。

### 导入库

导入库和导入可执行文件差不多，也是先创建导入目标再设置相应属性的值。

```CMake
add_library(<name> SHARED|STATIC IMPORTED [GLOBAL])
```

设置`IMPORTED_LOCALTION`属性的值以使cmake找到该库文件。

> 在Windows平台，动态库dll还需要导入库方可链接，导入库的后缀通常为`.lib`，导入库的位置通过`IMPORTED_IMPLIB`属性指定。

## 导出

导出通常是导出库。CMake的 `install` 命令提供了相应的支持，可以在安装库的同时生成导出目标，之后我们可以安装该导出目标。这个导出目标安装结果是一个CMake脚本文件，用于导入该库（即创建导入库）。

之后为了使CMake可以便捷的找到这些导出目标，我们还需要一个配置文件用于包含这些CMake脚本，这个配置文件通常由模板生成，我们需要创建一个模板文件，用CMake填充，并将生成的完整文件一同安装。

除此之外，我们在创建目标时还需要考虑到在构建过程中和被导入时的包含目录是不同的，所以我们在给目标添加包含目录时要添加两个不同情况下的路径。

### 包含目录

以下面的语句为例。

```cmake
target_include_directories(
    demo
    PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include>"
)
```

`$<...>` 称作生成器表达式，它们在构建时会被替换为相应的值。我们添加了两个生成器表达式

- `$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>` ：用于指定构建过程中的包含目录，不能像之前那样使用相对路径，但我们可以通过 `CMAKE_CURRENT_SOURCE_DIR` 这个变量获取当前 `CMakeLists.txt` 文件所在的路径，再相对这个路径进行指定。
- `$<INSTALL_INTERFACE:include>` ：用于指定安装后的包含目录，可以使用相对路径，相对于 `CMAKE_INSTALL_PREFIX` 。

> 不要忘记将公开的头文件也安装到指定位置。

### 导出目标

我们在使用 `install` 命令安装目标时，可以通过 `EXPORT` 参数同时创建导出目标，命名习惯是 `libraryName-targets` 。命令类似下面

```cmake
install(
    TARGETS demo
    EXPORT demo-targets
)
```

我们可以在多个 `install` 命令中使用同一个 `EXPORT` 名称，这样会将多个目标导出到同一个导出目标中。

之后我们需要安装导出目标，这样会将生成的导入脚本安装到指定位置，其他项目就可以通过运行这些CMake脚本创建导入目标。命令如下

```cmake
install(
    EXPORT demo-targets
    FILE demo-targets.cmake
    NAMESPACE demo::
    DESTINATION lib/cmake/demo
)
```

我们使用`install(EXPORT ...)` 命令来安装导出目标，`FILE` 参数指定导出目标的脚本文件名。

`NAMESPACE` 用于指定命名空间，包含在该导出目标中的目标导入后其名称前会被加上命名空间作为前缀，一般使用C++的命名习惯，使用两个冒号 `::` 作为间隔。导入后像这样引用 `link_library(demo::demo)`。当然也可以留空，这样导入后的目标的名称前不会加任何前缀。

`DESTINATION` 参数指定导出目标的安装位置，一般放在 `lib/cmake/libraryName` 目录下。

### 配置文件

之后要生成配置文件， `find_package` 命令会寻找这个文件作为库个配置文件。这个文件一般用于包含我们导出的那些导出目标文件。

#### 模板

首先我们需要先写一个模板文件，命名为 `Config.cmake.in`，类似下面

```cmake
@PACKAGE_INIT@

include(CMakeFindDependencyMacro)
find_dependency(demo_dep REQUIRED)

include("${CMAKE_CURRENT_LIST_DIR}/demo-targets.cmake")

check_required_components(demo)
```

第一行将会被处理并替换为相应的初始化内容。

第二、三行用于包含依赖的包。如果我们的库有依赖，那么我们不仅在构建时需要处理好，在导出时也需要在配置文件里面处理好依赖。首先我们需要包含 `CMakeFindDependencyMacro` 模块，这个模块提供了 `find_dependency` 函数用于查找依赖的包。我们可以通过 `REQUIRED` 参数来指定该依赖是必须的。`find_dependency` 的用法与 `find_package` 基本相同，但一般只推荐在配置文件使用。

第四行用于包含我们安装的导出文件，我们通过 `CMAKE_CURRENT_LIST_DIR` 变量来获取当前处理的文件的路径，也就是该配置文件的路径。

第五行用于检查是否包含了必要的组件。我们目前没有使用组件功能，因此只有一个组件也就是 `demo`

#### 生成

CMake提供了一个 `configure_package_config_file` 函数用于生成配置文件。要使用这个文件我们需要先包含CMake提供的工具 `CMakePackageConfigHelpers`。这个命令可以通过模板文件生成配置文件。

```cmake
include(CMakePackageConfigHelpers)
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/demo-config.cmake"
    INSTALL_DESTINATION lib/cmake/demo
)
```

第一个参数是模板文件的路径，第二个参数是生成的配置文件的路径，我们将其放在当前构建文件夹而不是直接放在安装目录，第三个参数 `INSTALL_DESTINATION` 是安装位置，一般在 `lib/cmake/libraryName`。

#### 安装

最后我们将配置文件安装到我们刚刚指定的安装位置（上一个命令虽然指定了但没有执行安装）。

```cmake
install(
    FILES "${CMAKE_CURRENT_BINARY_DIR}/demo-config.cmake"
    DESTINATION lib/cmake/demo
)
```

到此配置就完成了。

### 完整代码示例

`CMakeLists.txt`：

```cmake
# 创建目标
add_library(demo SHARED demo.cpp)

# 链接依赖库
target_link_libraries(demo PUBLIC demo_dep)

# 添加包含目录
target_include_directories(
    demo
    PUBLIC
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
        "$<INSTALL_INTERFACE:include>"
)

# 安装库
install(
    TARGETS demo
    EXPORT demo-targets
)

# 安装头文件
install(DIRECTORY include/ DESTINATION include)

# 安装导出
install(
    EXPORT demo-targets
    FILE demo-targets.cmake
    NAMESPACE demo::
    DESTINATION lib/cmake/demo
)

# 生成配置文件
include(CMakePackageConfigHelpers)
configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/demo-config.cmake"
    INSTALL_DESTINATION lib/cmake/demo
)

# 安装配置文件
install(
    FILES "${CMAKE_CURRENT_BINARY_DIR}/demo-config.cmake"
    DESTINATION lib/cmake/demo
)
```

`Config.cmake.in`：

```cmake
@PACKAGE_INIT@

include(CMakeFindDependencyMacro)
find_dependency(demo_dep REQUIRED)

include("${CMAKE_CURRENT_LIST_DIR}/demo-targets.cmake")

check_required_components(demo)
```

## 总结

本文简单介绍了如何使用CMake进行导入导出。足以让我们开发一些简单的共享库并使其他开发者可以方便地使用。

对于开发一些更复杂的库乃至框架，可能需要拆分成多个组件，本文并未涉及。
