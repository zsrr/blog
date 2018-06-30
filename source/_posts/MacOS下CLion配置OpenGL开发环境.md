---
title: MacOS下CLion配置OpenGL开发环境
date: 2018-02-06 10:31:34
tags: 配置
categories: OpenGL

---
讲一下Mac下配置OpenGL的过程，不要用Xcode，这个IDE写C++体验真是太不好了……

<!--more-->

# 配置glew和glfw
这个没什么好说的，直接在终端运行以下两句：

    brew install glew
    brew install glfw3

安装完后在/usr/local/Cellar/下可以找到对应的目录。
# 配置glad
这一步不是必须的。

glad是为了简化开发而设计的，不是必须的，是一个function loader。在[glad文件生成网站](http://glad.dav1d.de/)生成恰当的glad压缩文件，解压缩后将头文件放到/usr/local/include目录下(glad和KHR文件夹)，将源文件拷贝到工程目录下。
# 配置CMakeLists文件
下面是CMakeLists的详细配置：

    cmake_minimum_required(VERSION 3.8)
    project(opengl_demo)

    set(CMAKE_CXX_STANDARD 11)

    set(GLEW_H /usr/local/Cellar/glew/2.1.0/include/GL)
    set(GLFW_H /usr/local/Cellar/glfw/3.2.1/include/GLFW)
    set(GLAD_H /usr/local/include/glad)
    set(KH_H /usr/local/include/KHR)
    include_directories(${GLEW_H} ${GLFW_H} ${GLAD_H} ${KH_H})
    # 引用MacOS本身自带的OpenGL.framework
    find_library(OPENGL OpenGL)
    # 引用动态库
    set(GLEW_LINK /usr/local/Cellar/glew/2.1.0/lib/libGLEW.2.1.dylib)
    set(GLFW_LINK /usr/local/Cellar/glfw/3.2.1/lib/libglfw.3.dylib)
    link_libraries(${OPENGL} ${GLEW_LINK} ${GLFW_LINK})
    # 以下是我自己的项目配置
    set(HELLO_WINDOW hello-window.cpp glad.c)
    set(TRIANGLE triangle.cpp glad.c)

    add_executable(hello-window ${HELLO_WINDOW})
    add_executable(triangle ${TRIANGLE})
注意这样配置之后要更改/usr/local/include/glad文件夹下的glad.h中的#include &lt;KHR/khrplatform.h&gt;更改为#include &lt;khrplatform.h&gt;，具体情况看头文件查找路径是怎么写的了。

