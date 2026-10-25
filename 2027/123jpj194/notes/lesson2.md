# 第二堂课笔记：从编译到 CMake 与 catkin

> 主题脉络：`gcc/g++` → `Makefile` → `CMake` → `catkin`（ROS 1）。每一层都是为了解决上一层在项目变大后的维护难题。

## 目录

- [1. 引言](#1-引言)
- [2. 从源代码到可执行程序](#2-从源代码到可执行程序)
- [3. Makefile](#3-makefile)
- [4. CMake](#4-cmake)
- [5. catkin](#5-catkin)

---

## 1. 引言

- 源代码必须经过**编译**才能在系统上运行，Linux 下的编译器有 `gcc`、`g++`。
- 源文件增多后，直接敲 `gcc/g++` 命令效率低下，于是用 **Makefile** 管理编译。
- 工程体量进一步增大，Makefile 也不够用了，于是出现了 **CMake**。
- CMake 是 `make` 的**生成器**，是更高层的管理工具：简化构建流程，能管理大型项目，扩展性好。ROS 这样的大体量平台就采用 CMake。
- ROS 对 CMake 做了扩展，由此有了 **catkin** 编译系统。

| 层级 | 工具 | 解决什么问题 | 遇到的瓶颈 |
|---|---|---|---|
| 1 | `gcc` / `g++` | 把源码编译成可执行程序 | 文件多了，命令敲不过来 |
| 2 | Makefile + `make` | 自动化编译、构建 | 规则越来越多，难维护，不够跨平台 |
| 3 | CMake | 用简单配置文件生成各平台的构建文件 | ROS 有自己的包管理与规范 |
| 4 | catkin | 在 CMake 之上包一层 ROS 规范 | 仅适用于 ROS 1 |

---

## 2. 从源代码到可执行程序

计算机只能看懂二进制代码（机器语言）。我们写的 C++ 等高级语言代码，要靠**编译器**转换成机器语言，编译器相当于一个"翻译官"。

转换分为四个阶段：

**预处理 → 编译 → 汇编 → 链接**

![从源代码到可执行程序的四个阶段](images/compile_pipeline.png)

| 阶段 | 执行程序 | 输入 | 输出 |
|---|---|---|---|
| 预处理 | 预处理器（`cpp`） | `hello.c`（源程序，文本） | `hello.i`（修改了的源程序，文本） |
| 编译 | 编译器（`cc1`） | `hello.i` | `hello.s`（汇编程序，文本） |
| 汇编 | 汇编器（`as`） | `hello.s` | `hello.o`（可重定位目标程序，二进制） |
| 链接 | 链接器（`ld`） | `hello.o` 及 `printf.o` 等 | `hello`（可执行目标程序，二进制） |

执行这四个阶段的程序合在一起，构成**编译系统**。

---

## 3. Makefile

- Makefile 就像一份**自动编译说明书**，告诉 `make` 工具：
  - 哪些文件要编译
  - 编译的顺序是什么
  - 需要执行哪些命令
- Makefile 实现了编译、构建的自动化。写好之后，每次改完代码只需敲一次 `make`，就能得到目标程序。

---

## 4. CMake

### 4.1 为什么需要 CMake

- 文件和依赖增多后，需要维护的 Makefile 规则越来越多，手写很不方便。
- 只靠 Makefile 无法灵活地**跨平台**，CMake 解决了这个问题。

### 4.2 CMake 是什么

- CMake 是一种过程式语言，也是一个**跨平台**的编译工具，用简单的语句描述所有平台的编译过程。
- 它能产生标准的 Makefile 或 project 文件，所以在 Linux 上写好的项目，在 Windows 上也能编译。
- 做法：编写一个**独立于平台和编译器**的配置文件，由 CMake 生成本地的 Makefile。不论什么平台、什么编译器，CMake 都能生成对应的构建文件。
- 注意：CMake 只能管理 C/C++ 项目的编译过程。

### 4.3 CMakeLists.txt

- 它是 CMake 的**核心配置文件**，每个 CMake 项目至少需要一个。
- 通过它可以定义项目的构建规则、依赖关系、编译选项；CMake 读取其中的指令，了解项目结构、源文件列表和编译要求，再生成对应平台的构建文件。
- 基本语法：由一系列指令组成，每条指令的格式为 `命令名(参数列表)`。

### 4.4 最常用的指令

| 指令 | 作用 | 示例 | 备注 |
|---|---|---|---|
| `cmake_minimum_required` | 指定 CMake 最低版本 | `cmake_minimum_required(VERSION 3.10)` | 必须放在 `CMakeLists.txt` 最顶部，表示要求 CMake 版本不低于 3.10 |
| `project` | 定义项目名称和语言 | `project(项目名 编程语言)` | 调用后自动设置 `PROJECT_NAME` 等变量；语言参数可以不写，默认是 C 和 C++ |
| `add_executable` | 添加可执行文件 | `add_executable(目标名 源文件...)` | 把源文件编译成可执行文件，文件名就是目标名，源文件可以写多个 |

### 4.5 自动扫描源文件

每新增一个源文件就手动添加，既麻烦，又会让文件变得冗长。如果想让 CMake 自己扫描目录里的源文件，可以这样写：

```cmake
aux_source_directory("${CMAKE_CURRENT_SOURCE_DIR}/src" SOURCES)
add_executable(demo ${SOURCES})
target_include_directories(demo PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/include")
```

逐段说明：

| 语句 | 含义 |
|---|---|
| `aux_source_directory(... SOURCES)` | `CMAKE_CURRENT_SOURCE_DIR` 表示当前 `CMakeLists.txt` 所在的文件夹；`src` 只是示例，换别的目录名也行。该语句把 `src` 目录下的源文件列表保存到变量 `SOURCES` |
| `add_executable(demo ${SOURCES})` | 把收集到的源文件编译成可执行程序 `demo`，由此实现对 `src` 里文件的自动扫描 |
| `target_include_directories(demo PRIVATE ...)` | 添加头文件搜索目录，使源文件可以写 `#include "add.h"` 这样的形式 |

> ⚠️ 注意：自动扫描**只在每次运行 `cmake` 时生效**。新增或删除源文件后，需要重新运行 `cmake ..` 才会更新文件列表。

---

## 5. catkin

### 5.1 catkin 是什么

- catkin 是在 CMake 之上**更高一层的抽象**，简单说就是包了一层 ROS 规范的 CMake，让我们不用手写完整的 `CMakeLists.txt`。
- catkin 是 **ROS 1 专属**的编译系统，只适用于 Ubuntu 20.04（ROS 1 Noetic）。ROS 2 已经换成了 `colcon` / `ament`。
- 这也是学长要求模拟 Ubuntu 20.04 环境的原因。

### 5.2 工作空间

使用 catkin 之前，需要先搭建一个**工作空间**。目录结构如下：

```text
catkin_ws/
├── src/      # 源码空间：写的所有功能包都放这里（唯一需要动手的地方）
├── build/    # 编译空间：CMake / catkin 的缓存和中间文件，基本不用管
└── devel/    # 开发空间：编译出来的可执行文件、库
```

| 目录 | 名称 | 内容 | 是否需要手动改动 |
|---|---|---|---|
| `src/` | 源码空间 | 所有功能包 | **是**，唯一需要动手的地方 |
| `build/` | 编译空间 | CMake / catkin 的缓存和中间文件 | 否，基本不用管 |
| `devel/` | 开发空间 | 编译出来的可执行文件、库 | 否 |

### 5.3 搭建工作空间的流程

```bash
mkdir -p ~/catkin_ws/src     # 创建工作空间以及 src 目录
cd ~/catkin_ws/src
catkin_init_workspace        # 初始化工作空间
```

- `catkin_init_workspace` 会在 `src` 下生成 `CMakeLists.txt`，执行完后在终端输入 `ls` 就能看到。

接下来，在 `src` 目录下创建具体的功能包：

```bash
catkin_create_pkg test_pkg std_msgs rospy roscpp
```

- `test_pkg`：包名，可以换成别的。
- `std_msgs rospy roscpp`：这个包的依赖。
