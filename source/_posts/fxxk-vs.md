---
title: Windows平台下vcpkg+code+cmake配置踩坑实录
date: 2023-09-08 20:33:32
tags:
---

以配置glut环境为例

---

已预安装：visual studio、visual studio code、llvm（clang、clangd）、msys2（gcc）、cmake

安装vcpkg：git克隆GitHub仓库到本地，按照官方流程执行命令（踩坑点：visual studio要安装英文语言包）

vcpkg库目录搜索opengl的glut，发现两个库：`opengl`和`freeglut`，先把两个全部install一下（下载缓慢或者失败要解决网络问题）

visual studio code打开，在设置文件中照vcpkg的官方文档添加一行设置，然后写一个CMakeLists.txt（踩坑点：漏写了内容）

```cmake
cmake_minimum_required(VERSION 3.8)

project(first)

find_package(GLUT REQUIRED)
add_executable(first a.cpp)
target_link_libraries(first PRIVATE GLUT::GLUT)
```

CMake切换到非msvc（gcc、clang）时报错，需要安装`ninja`和`make`。网上教程找到了choco一把梭式安装的教程，不然ninja要从源码编译，更加艰难。

cpp文件内写`#include <GLUT/GLUT.h>`，clangd（语言服务器、LSP）报错无法解析路径，怀疑是路径问题，进入`vcpkg`的安装目录下，发现头文件的路径实际为：`GL/freeglut.h`。修改，照着老师给出的demo代码一行行写下去，发现clangd自动添加了另外三个头文件🤣。最终成功启动了demo，运行截图如下：

![运行截图](2023-09-08-20-30-09-image.png)
