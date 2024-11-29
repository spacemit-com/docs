# 图形编程指南

## 1. OpenGLES编程指南

### 1.1 简介

OpenGLES是一种专为 嵌入式设备 设计的轻量级3D图形库，是OpenGL标准的一个子集。它提供了跨平台的API ，适用于移动设备和资源受限的系统。OpenGLES相较于OpenGL的主要区别有：

- 数据类型 ：不支持double型，新增高性能定点小数类型
- 绘图命令 ：取消glBegin/glEnd/glVertex，仅保留glDrawArrays等
- 纹理处理 ：要求直接提供压缩贴图，提高渲染效率

EGL（Embedded Systems Graphics Library）是OpenGL ES生态系统中的关键组件，充当 OpenGL ES渲染API和本地窗口系统之间的桥梁 。作为一个独立于平台的接口层，EGL屏蔽了不同硬件平台和操作系统之间的差异，为OpenGL ES提供了统一的操作环境。EGL的主要功能包括：

- 图形上下文管理 ：创建和维护OpenGL ES渲染上下文
- 表面/缓冲区创建 ：负责创建绘图表面和缓冲区
- 渲染同步 ：协调OpenGL ES与显示设备进行同步，确保渲染结果正确显示
- 资源管理 ：管理纹理贴图等渲染资源

### 1.2 渲染流程

在OpenGLES和EGL的协同工作中，渲染流程是整个图形渲染系统的核心，下图简要展示了 OpenGLES 进行图形渲染的过程。

![mesa3d](./static/opengles_process.png#pic_center)

整个图形渲染的流程具体包括以下5个步骤：

1. EGL初始化
EGL初始化是渲染流程的第一步，它为后续的渲染操作奠定基础。在这个阶段，开发者需要完成以下关键操作：
    - 获取EGLDisplay对象：通过eglGetDisplay()函数获取与本地窗口系统相连的EGLDisplay对象。
    - 初始化EGL环境：调用eglInitialize()函数初始化EGL环境，获取EGL的版本信息。
    - 选择EGLConfig：使用eglChooseConfig()函数选择合适的EGL配置，指定所需的渲染属性，如颜色深度、alpha通道等。
    - 创建EGLContext：通过eglCreateContext()函数创建EGL渲染上下文，指定OpenGL ES的版本和所需的其他选项。

2. 创建渲染表面
接下来，需要创建一个渲染表面，作为OpenGLES渲染的目标。EGL提供了几种创建渲染表面的方法：
    - 窗口表面 ：使用eglCreateWindowSurface()函数创建与窗口系统关联的渲染表面。
    - 离屏表面 ：通过eglCreatePbufferSurface()函数创建离屏渲染表面，常用于生成纹理。
    - 位图表面 ：利用eglCreatePixmapSurface()函数创建基于位图的渲染表面。

3. 设置渲染上下文
在创建完渲染表面后，需要将EGLContext与EGLSurface关联起来，以便OpenGLES可以使用正确的渲染上下文进行渲染。这一步骤通常通过eglMakeCurrent()函数完成。

4. OpenGLES渲染操作
一旦EGL环境设置完毕，就可以开始使用OpenGLES进行渲染了。OpenGLES渲染流程通常包括以下步骤：
    - 设置视口 ：使用glViewport()函数定义渲染的区域。
    - 清除缓冲区 ：调用glClear()函数清除颜色、深度和模板缓冲区。
    - 激活着色器程序 ：通过glUseProgram()函数激活预先准备好的着色器程序。
    - 传递渲染参数 ：使用glVertexAttribPointer()等函数传递顶点坐标、纹理坐标等参数。
    - 绘制图元 ：调用glDrawArrays()或glDrawElements()函数绘制几何形状。

5. 交换缓冲区
渲染完成后，需要将渲染结果从后缓冲区交换到前缓冲区，以便在屏幕上显示。这一步骤通常通过调用eglSwapBuffers()函数完成。此外，EGL还提供了同步机制，如eglWaitSyncKHR()函数，用于等待特定的渲染操作完成。这在处理复杂的渲染场景或多线程渲染时，可以帮助开发者更好地控制渲染流程的时间和顺序。

### 1.3 EGL API

EGL API通过Mesa3D提供具体实现，在 k1x-gpu-test Demo 中提供了k1平台上的调用示例和封装。前面的渲染流程中已经提及了在一次渲染过程中主要使用的几个EGL API，更详细的用法可以参考：

1. [EGL Reference Pages](https://registry.khronos.org/EGL/sdk/docs/man/)

2. [EGL Spec](https://registry.khronos.org/EGL/specs/eglspec.1.5.pdf)

### 1.4 OpenGLES API

目前的GPU DDK支持最新的OpenGLES 3.2, 以下列举几个主要的API用法示例：

- `void glShaderSource(GLuint shader, GLsizei count, const GLchar *const *string, const GLint *length)` 用于将指定的源码字符串加载到指定的着色器对象中。

    ```txt
    shader：指定要加载源代码的着色器对象。
    count：指定要加载的源代码字符串的数量。
    string：一个指向包含源代码字符串的数组的指针。
    length：一个可选的指向每个源代码字符串长度的数组的指针。如果提供了这个参数，那么它应该包含与 string 数组相同数量的元素，并且每个元素都指定了相应源代码字符串的长度。如果 length 为 NULL，则假定所有的源代码字符串都是以空字符 '\0' 结尾的。
    ```

- 在加载源代码后，需调用 `void glCompileShader(GLuint shader)` 来编译着色器。它将源码编译成可执行的机器代码，并将结果存在一个着色器对象中。shader 是要编译的着色器对象的标识符。调用 glCompileShader 函数后，可通过调用 `glGetShaderInfoLog` 函数来获取编译过程中的错误信息。

- `glCreateProgram` 用于创建一个新的 OpenGL 程序对象，并返回一个指向该对象的句柄。程序对象是一个用于存储和管理 OpenGL 程序的容器。

    ```c
    GLuint program = glCreateProgram(); //创建一个新的 Open GL 程序对象
    ```

- `void glAttachShader(GLuint program, GLuint shader)` 用于绑定着色器对象到着色器程序。program：指定要附加着色器的程序对象的标识符；shader：指定要附加的着色器对象的标识符。

- `void glLinkProgram(GLuint program)` 用于将可编程渲染管道（OpenGL Shading Language）的顶点和片段着色器程序连接到一个可执行的程序对象中。如果连接成功，函数将返回 GL_TRUE，否则返回 GL_FALSE。

- glGenVertexArrays & glGenBuffers 用于创建顶点数组对象（Vertex Array Object，VAO）和顶点缓冲对象（Vertex Buffer Object，VBO），并将顶点数据发送到 GPU。这两个函数可以将顶点数据从 CPU 发送到 GPU，这样 GPU 就可以在渲染过程中直接访问这些数据，而不是每次都从 CPU 获取数据。

关于OpenGL ES 3.2 API 的详细用法，可以参考：

1. [OpenGLES Reference Pages](https://registry.khronos.org/OpenGL-Refpages/es3/)

2. [OpenGLES Spec 3.2](https://registry.khronos.org/OpenGL/specs/es/3.2/es_spec_3.2.pdf)

## 2. OpenGLES Demo

### 2.1 简介

bianbu-linux 上的源码位置：xxx/bianbu-linux/package-src/k1x-gpu-test/openGLDemo

bianbu-desktop 上可以安装 k1x-gpu-test 来获取相关demo：`sudo apt install k1x-gpu-test`
目录结构如下：

```c
.
|-- CMakeLists.txt //cmake文件，用于编译构建
|-- README.md
|-- common
|   |-- include //头文件
|   |   |-- stb_image.h //单头文件图像加载库
|   |   |-- utils_opengles.h
|   |   `-- xdg-shell-client-protocol.h
|   `-- source
|       |-- utils_opengles.c //对egl等函数调用的封装
|       `-- xdg-shell-protocol.c //xdg-shell协议
|-- cube_demo.c //立方体旋转
|-- cube_externalTexture_demo.c //立方体外部纹理贴图
|-- cube_texture_demo.c //立方体2D纹理贴图
|-- data //2D图片文件
|   |-- bianbu.png
|   `-- bianbu2.png
|-- square_demo.c //矩形
|-- texture_square_rotation_demo.c //2D纹理+旋转渲染的矩形
`-- triangle_demo.c //三角形

4 directories, 15 files
```

### 2.2 编译 & 运行

```shell
~ cd k1x-gpu-test/openGLDemo
~ cmake .
~ make
```

编译完成后会在当前目录下生成 gpu-xxxDemo 文件，可直接运行：`./gpu-cubeTextureDemo`。如需安装，可在当前目录（即源码目录）下执行 `make install` 命令，会自动将可执行文件安装到 /usr/local/bin/ 目录下，安装完成后可直接运行：`gpu-cubeTextureDemo`。

在 bianbu-linux 系统上，需要手动设置以下环境变量后方可运行：

```bash
XDG_RUNTIME_DIR=/root WAYLAND_DISPLAY=wayland-1 MESA_LOADER_DRIVER_OVERRIDE=pvr ./gpu-cubeTextureDemo
```

正确的运行效果如下图所示：

![gpu-cubeTextureDemo](./static/gpu-cubeTextureDemo.gif)

### 2.3 添加一个Demo

如果增加一个名为 testDemo.c 的文件，并希望编译得到名为 testDemo 的可执行文件，可按以下示例修改CMakeLists.txt

```Cmake
add_definitions(-DSTB_IMAGE_IMPLEMENTATION)
add_definitions(-DNDEBUG)

# 定义公共源文件列表
set(COMMON_SOURCE_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/common/source/xdg-shell-protocol.c
    ${CMAKE_CURRENT_SOURCE_DIR}/common/source/utils_opengles.c
)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/common/include/
)

# 添加链接库
set(LINK_LIBRARIES
    EGL
    GLESv2
    wayland-client
    wayland-server
    wayland-egl
    m
    wayland-cursor
    gbm
)

add_executable(gpu-cubeDemo ${CMAKE_CURRENT_SOURCE_DIR}/cube_demo.c ${COMMON_SOURCE_FILES})
add_executable(testDemo ${CMAKE_CURRENT_SOURCE_DIR}/testDemo.c ${COMMON_SOURCE_FILES}) # 创建新的可执行文件

target_link_libraries(gpu-cubeDemo ${LINK_LIBRARIES})
target_link_libraries(testDemo ${LINK_LIBRARIES})

# 安装， 将可执行文件安装到指定目录
install(
    TARGETS
    gpu-cubeDemo
    testDemo
    DESTINATION "${CMAKE_INSTALL_PREFIX}/bin")

file(GLOB IMAGE_FILES "data/*.png")
install(
    FILES ${IMAGE_FILES}
    DESTINATION "${CMAKE_INSTALL_PREFIX}/data")
```

然后再次执行 `make` 即可。

## 3. Vulkan

### 3.1 简介

Vulkan是Khronos Group在2015年发布的新一代跨平台图形渲染API，旨在提供对现代 GPU 的低级访问，从而实现更高效的性能。相比于OpenGL，Vulkan允许开发者更精细地控制GPU资源，如精确地指定何时提交任务给GPU、如何分配内存以及对并发执行进行调度等，这种直接访问硬件的权限带来了显著的性能提升，但代价是API过于冗长。

PVR GPU驱动提供了对Vulkan API的具体实现，应用程序可以通过调用相应的API来完成绘制任务。

关于Vulkan API 的详细用法，可以参考：

1. [Vulkan Specs](https://registry.khronos.org/vulkan/specs/1.3/html/)

### 3.2 Vulkan Demo

To be continue
