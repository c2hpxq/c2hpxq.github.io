---
layout: post
title:  "llvm tutor cmake file explained"
date:   2021-08-08 23:31:38 +0800
categories: jekyll update
---

# Intro
LLVM提供基于cmake的配置方式。配置完成后（或可能是build完成后，没有分析这个时机），关于LLVM的一些信息，如版本，头文件目录等将会被存储在指定的cmake文件中，以便若使用LLVM的项目也采用cmake进行配置，可以方便地“include”并使用。
结合相关cmake文件，学习如何写LLVM subproject的最小配置文件，以及cmake相关语法。

# Main cmake file

```cmake
cmake_minimum_required(VERSION 3.13.4)
```
[cmake_minimum_required](https://cmake.org/cmake/help/latest/command/cmake_minimum_required.html)：指定所需的cmake最低版本。
```cmake
project(llvm-tutor)
```
[project](https://cmake.org/cmake/help/latest/command/project.html)：指定项目名称。同时会设置PROJECT_NAME、PROJECT_SOURCE_DIR等变量

```cmake
#===============================================================================
# 1. VERIFY LLVM INSTALLATION DIR
# This is just a bit of a sanity checking.
#===============================================================================
set(LT_LLVM_INSTALL_DIR "" CACHE PATH "LLVM installation directory")

# 1.1 Check the "include| directory
set(LT_LLVM_INCLUDE_DIR "${LT_LLVM_INSTALL_DIR}/include/llvm")
```
[set](https://cmake.org/cmake/help/latest/command/set.html)：设定变量值。这里CACHE说明这一项是用户可设置的。如果用户指定了`LT_LLVM_INSTALL_DIR`的值，那么这里不会覆写。

```cmake
if(NOT EXISTS "${LT_LLVM_INCLUDE_DIR}")
message(FATAL_ERROR
  " LT_LLVM_INSTALL_DIR (${LT_LLVM_INCLUDE_DIR}) is invalid.")
endif()
```
cmake提供条件控制[if](https://cmake.org/cmake/help/latest/command/if.html)。
[message](https://cmake.org/cmake/help/latest/command/message.html)打印log信息。同时可以指定“错误”等级，根据不同等级采取终止、继续、报警等操作。这里的`FATAL_ERROR`就将终止配置过程。

```cmake
# 1.2 Check that the LLVMConfig.cmake file exists (the location depends on the
# OS)
set(LT_VALID_INSTALLATION FALSE)

# Ubuntu + Darwin
if(EXISTS "${LT_LLVM_INSTALL_DIR}/lib/cmake/llvm/LLVMConfig.cmake")
  set(LT_VALID_INSTALLATION TRUE)
endif()

# Fedora
if(EXISTS "${LT_LLVM_INSTALL_DIR}/lib64/cmake/llvm/LLVMConfig.cmake")
  set(LT_VALID_INSTALLATION TRUE)
endif()

if(NOT ${LT_VALID_INSTALLATION})
  message(FATAL_ERROR
    "LLVM installation directory, (${LT_LLVM_INSTALL_DIR}), is invalid. Couldn't
    find LLVMConfig.cmake.")
endif()
```
sanity check，粗略检查我们指定的LLVM安装目录下有没有对应的LLVMConfig.cmake这一配置文件。

```cmake
#===============================================================================
# 2. LOAD LLVM CONFIGURATION
#    For more: http://llvm.org/docs/CMake.html#embedding-llvm-in-your-project
#===============================================================================
# Add the location of LLVMConfig.cmake to CMake search paths (so that
# find_package can locate it)
# Note: On Fedora, when using the pre-compiled binaries installed with `dnf`,
# LLVMConfig.cmake is located in "/usr/lib64/cmake/llvm". But this path is
# among other paths that will be checked by default when using
# `find_package(llvm)`. So there's no need to add it here.
list(APPEND CMAKE_PREFIX_PATH "${LT_LLVM_INSTALL_DIR}/lib/cmake/llvm/")

find_package(LLVM 12.0.0 REQUIRED CONFIG)
```
[find_package](https://cmake.org/cmake/help/latest/command/find_package.html)：寻找本地package对应的cmake配置文件，并导入对应的设置。
指定`REQUIRED`则表明该package是必须的，缺少则会终止配置。

```cmake
# Another sanity check
if(NOT "12" VERSION_EQUAL "${LLVM_VERSION_MAJOR}")
  message(FATAL_ERROR "Found LLVM ${LLVM_VERSION_MAJOR}, but need LLVM 12")
endif()

message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")
message(STATUS "Using LLVMConfig.cmake in: ${LT_LLVM_INSTALL_DIR}")

message("LLVM STATUS:
  Definitions ${LLVM_DEFINITIONS}
  Includes    ${LLVM_INCLUDE_DIRS}
  Libraries   ${LLVM_LIBRARY_DIRS}
  Targets     ${LLVM_TARGETS_TO_BUILD}"
)
```

```cmake
# Set the LLVM header and library paths
include_directories(SYSTEM ${LLVM_INCLUDE_DIRS})
```
[include_directories](https://cmake.org/cmake/help/latest/command/include_directories.html)：将路径加入编译器搜索头文件的目录


```cmake
link_directories(${LLVM_LIBRARY_DIRS})
```
[link_directories](https://cmake.org/cmake/help/latest/command/link_directories.html)：将路径加入linker搜索lib文件的目录

```cmake
add_definitions(${LLVM_DEFINITIONS})
```
[add_definitions](https://cmake.org/cmake/help/latest/command/add_definitions.html)：将`-D`指定的flag加入编译命令中

```cmake
#===============================================================================
# 3. LLVM-TUTOR BUILD CONFIGURATION
#===============================================================================
# Use the same C++ standard as LLVM does
set(CMAKE_CXX_STANDARD 14 CACHE STRING "")
```
[CMAKE_CXX_STANDARD](https://cmake.org/cmake/help/latest/variable/CMAKE_CXX_STANDARD.html)指定编译过程中使用什么c++标准。

```cmake
# Build type
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug CACHE
      STRING "Build type (default Debug):" FORCE)
endif()
```

```cmake
# Compiler flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall\
    -fdiagnostics-color=always")

# LLVM is normally built without RTTI. Be consistent with that.
if(NOT LLVM_ENABLE_RTTI)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
endif()
```
编译器选项通过[CMAKE_CXX_FLAGS](https://cmake.org/cmake/help/latest/envvar/CXXFLAGS.html)指定

```cmake
# -fvisibility-inlines-hidden is set when building LLVM and on Darwin warnings
# are triggered if llvm-tutor is built without this flag (though otherwise it
# builds fine). For consistency, add it here too.
include(CheckCXXCompilerFlag)
check_cxx_compiler_flag("-fvisibility-inlines-hidden" SUPPORTS_FVISIBILITY_INLINES_HIDDEN_FLAG)
if (${SUPPORTS_FVISIBILITY_INLINES_HIDDEN_FLAG} EQUAL "1")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility-inlines-hidden")
endif()
```
[include](https://cmake.org/cmake/help/latest/command/include.html)导入一个cmake file/module。[CheckCXXCompilerFlag](https://cmake.org/cmake/help/latest/module/CheckCXXCompilerFlag.html)是cmake的一个module，其中定义了`check_cxx_compiler_flag`这个command。该command检查编译器是否支持特定选项。

```cmake
# Set the build directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${PROJECT_BINARY_DIR}/lib")
```
[PROJECT_BINARY_DIR](https://cmake.org/cmake/help/v3.0/variable/PROJECT_BINARY_DIR.html)是由最近的`project`命令指定的
[CMAKE_LIBRARY_OUTPUT_DIRECTORY](https://cmake.org/cmake/help/latest/variable/CMAKE_LIBRARY_OUTPUT_DIRECTORY.html)指定存放生成的lib文件的路径。
[CMAKE_RUNTIME_OUTPUT_DIRECTORY](https://cmake.org/cmake/help/latest/variable/CMAKE_RUNTIME_OUTPUT_DIRECTORY.html)指定存放生成的可执行文件的路径

```cmake
#===============================================================================
# 4. ADD SUB-TARGETS
# Doing this at the end so that all definitions and link/include paths are
# available for the sub-projects.
#===============================================================================
add_subdirectory(lib)
add_subdirectory(tools)
add_subdirectory(test)
add_subdirectory(HelloWorld)
```
[add_subdirectory](https://cmake.org/cmake/help/latest/command/add_subdirectory.html)继续配置子目录。相对路径会在当前目录下寻找。subdirectory会“继承”parent cmake file的处理结果

# Cmake file for HelloWorld
```cmake
#...

#===============================================================================
# 3. ADD THE TARGET
#===============================================================================
add_library(HelloWorld SHARED HelloWorld.cpp)
```
[add_library](https://cmake.org/cmake/help/latest/command/add_library.html)：用指定的source code生成lib文件。这里SHARED指定生成shared library，HelloWorld是lib的“logical name”。具体生成的lib name，若在linux下则为libHelloWorld.so。

```cmake
# Allow undefined symbols in shared objects on Darwin (this is the default
# behaviour on Linux)
target_link_libraries(HelloWorld
  "$<$<PLATFORM_ID:Darwin>:-undefined dynamic_lookup>")
```
[target_link_libraries](https://cmake.org/cmake/help/latest/command/target_link_libraries.html)：指定需要链接的库、链接参数。
这是cmake file的结尾。没有显式指定target要链接哪些动态链接库，unsolved symbol等到运行时再进行解决。根据注释，运行时resolve是Linux下的默认行为；而mac下则需要指定选项：`-undefined dynamic_lookup`来显式命令runtime search。

update:
引号中的表达式是cmake的[cmake-generator-expressions](https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html)
> $<PLATFORM_ID:platform_ids>
> where platform_ids is a comma-separated list. 1 if the CMake's platform id matches any one of the entries in platform_ids, otherwise 0.

所以在mac下`$<$<PLATFORM_ID:Darwin>:-undefined dynamic_lookup>`会解析成`$<1:-undefined dynamic_lookup>`；

> $<condition:true_string>
> Evaluates to true_string if condition is 1. Otherwise evaluates to the empty string.

所以`$<1:-undefined dynamic_lookup>`就解析成`undefined dynamic_lookup`

# Reference
LLVM提供的[cmake primer](https://llvm.org/docs/CMakePrimer.html)