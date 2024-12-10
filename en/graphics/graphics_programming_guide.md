# Graphics Programming Guide

## 1. OpenGL ES Programming Guide

### 1.1 Introduction

OpenGLES is a lightweight 3D graphics library designed for embedded devices and is a subset of the OpenGL standard. It provides a cross-platform API suitable for mobile devices and resource-constrained systems. The main differences between OpenGLES and OpenGL are:

- Data type: double type is not supported, and a new high-performance fixed-point decimal type is added
- Drawing commands: cancel glBegin/glEnd/glVertex, only keep glDrawArrays, etc.
- Texture processing: require compressed maps to be provided directly to improve rendering efficiency

EGL (Embedded Systems Graphics Library) is a key component in the OpenGL ES ecosystem, acting as a bridge between the OpenGL ES rendering API and the local window system. As a platform-independent interface layer, EGL shields the differences between different hardware platforms and operating systems, and provides a unified operating environment for OpenGL ES. The main functions of EGL include:

- Graphics context management: create and maintain OpenGL ES rendering context
- Surface/buffer creation: responsible for creating drawing surfaces and buffers
- Rendering synchronization: Coordinate rendering operations between OpenGL ES and synchronize with display devices to ensure that rendering results are displayed correctly
- Resource management: manage rendering resources such as texture maps

### 1.2 Rendering Process

In the collaborative work of OpenGLES and EGL, the rendering process is the core of the entire graphics rendering system. The following figure briefly shows the graphics rendering process of OpenGLES.

![mesa3d](./static/opengles_process_en.png)

The entire graphics rendering process specifically includes the following five steps:

1. EGL initialization
EGL initialization is the first step in the rendering process, which lays the foundation for subsequent rendering operations. At this stage, developers need to complete the following key operations:

    - Get the EGLDisplay object: Get the EGLDisplay object connected to the local window system through the eglGetDisplay() function.
    - Initialize the EGL environment: Call the eglInitialize() function to initialize the EGL environment and obtain the EGL version information.
    - Select EGLConfig: Use the eglChooseConfig() function to select the appropriate EGL configuration and specify the required rendering properties, such as color depth, alpha channel, etc.
    - Create EGLContext: Create an EGL rendering context through the eglCreateContext() function, specify the OpenGL ES version and other required options.

2. Create a rendering surface
Next, you need to create a rendering surface as the target for OpenGLES rendering. EGL provides several methods for creating rendering surfaces:

    - Window surface: Use the eglCreateWindowSurface() function to create a rendering surface associated with the window system.
    - Off-screen surface: Use the eglCreatePbufferSurface() function to create an off-screen rendering surface, which is often used to generate textures.
    - Bitmap surface: Use the eglCreatePixmapSurface() function to create a bitmap-based rendering surface.

3. Setting up the rendering context
After creating the rendering surface, you need to associate the EGLContext with the EGLSurface so that OpenGLES can use the correct rendering context for rendering. This step is usually done through the eglMakeCurrent() function.

4. OpenGLES rendering operation
Once the EGL environment is set up, you can start rendering with OpenGLES. The OpenGLES rendering process usually includes the following steps:

    - Set the viewport: Use the glViewport() function to define the rendering area.
    - Clear the buffer: Call the glClear() function to clear the color, depth, and template buffers.
    - Activate the shader program: Activate the pre-prepared shader program through the glUseProgram() function.
    - Pass rendering parameters: Use functions such as glVertexAttribPointer() to pass parameters such as vertex coordinates and texture coordinates.
    - Draw primitives: Call glDrawArrays() or glDrawElements() to draw geometric shapes.

5. Swap buffers
After rendering is complete, the rendering results need to be swapped from the back buffer to the front buffer so that they can be displayed on the screen. This step is usually completed by calling the eglSwapBuffers() function. In addition, EGL also provides synchronization mechanisms, such as the eglWaitSyncKHR() function, which is used to wait for a specific rendering operation to complete. This can help developers better control the timing and order of the rendering process when dealing with complex rendering scenes or multi-threaded rendering.

### 1.3 EGL API

The EGL API is implemented in Mesa3D. The k1x-gpu-test Demo provides examples and encapsulation of calls on the k1 platform. The previous rendering process has mentioned several EGL APIs that are mainly used in a rendering process. For more detailed usage, please refer to:

1. [EGL Reference Pages](https://registry.khronos.org/EGL/sdk/docs/man/)

2. [EGL Spec](https://registry.khronos.org/EGL/specs/eglspec.1.5.pdf)

### 1.4 OpenGLES API

The current GPU DDK supports the latest OpenGLES 3.2. Here are some main API usage examples:

- `void glShaderSource(GLuint shader, GLsizei count, const GLchar *const *string, const GLint *length)` is used to load the specified source code string into the specified shader object.

```txt
shader: specifies the shader object to load the source code.
count: specifies the number of source code strings to load.
string: a pointer to an array containing source code strings.
length: an optional pointer to an array of the length of each source code string. If this parameter is provided, it should contain the same number of elements as the string array, and each element specifies the length of the corresponding source code string. If length is NULL, it is assumed that all source code strings end with a null character '\0'.
```

- After loading the source code, call `void glCompileShader(GLuint shader)` to compile the shader. It compiles the source code into executable machine code and stores the result in a shader object. shader is the identifier of the shader object to be compiled. After calling the glCompileShader function, you can get the error information during the compilation process by calling the `glGetShaderInfoLog` function.

- `glCreateProgram` is used to create a new OpenGL program object and return a handle to the object. The program object is a container for storing and managing OpenGL programs.

    ```c
    GLuint program = glCreateProgram(); //Create a new Open GL program object
    ```

- `void glAttachShader(GLuint program, GLuint shader)` is used to bind a shader object to a shader program. program: specifies the identifier of the program object to which the shader is to be attached; shader: specifies the identifier of the shader object to be attached.

- `void glLinkProgram(GLuint program)` is used to connect the vertex and fragment shader programs of the programmable rendering pipeline (OpenGL Shading Language) to an executable program object. If the connection is successful, the function will return GL_TRUE, otherwise it returns GL_FALSE.

- glGenVertexArrays & glGenBuffers are used to create Vertex Array Objects (VAO) and Vertex Buffer Objects (VBO), and send vertex data to the GPU. These two functions can send vertex data from the CPU to the GPU so that the GPU can directly access this data during the rendering process instead of getting the data from the CPU every time.

For detailed usage of OpenGL ES 3.2 API, please refer to:

1. [OpenGLES Reference Pages](https://registry.khronos.org/OpenGL-Refpages/es3/)

2. [OpenGLES Spec 3.2](https://registry.khronos.org/OpenGL/specs/es/3.2/es_spec_3.2.pdf)

## 2. OpenGLES Demo

### 2.1 Introduction

Source code location on bianbu-linux: xxx/bianbu-linux/package-src/k1x-gpu-test/openGLDemo

You can install k1x-gpu-test on bianbu-desktop to get the relevant demos: `sudo apt install k1x-gpu-test`
The directory structure is as follows:

```c
.
|-- CMakeLists.txt //cmake file for compiling and building
|-- README.md
|-- common
|   |-- include //Header Files
|   |   |-- stb_image.h //Single header file image loading library
|   |   |-- utils_opengles.h
|   |   `-- xdg-shell-client-protocol.h
|   `-- source
|       |-- utils_opengles.c //Encapsulation of egl and other function calls
|       `-- xdg-shell-protocol.c //xdg-shell protocol
|-- cube_demo.c //Cube Rotation
|-- cube_externalTexture_demo.c //Cube exterior texture map
|-- cube_texture_demo.c //Cube 2D Texture Map
|-- data //2D Image File
|   |-- bianbu.png
|   `-- bianbu2.png
|-- square_demo.c //rectangle
|-- texture_square_rotation_demo.c //2D texture rotated rendered rectangle
`-- triangle_demo.c //triangle

4 directories, 15 files
```

### 4.2 Compile & Run

```shell
~ cd k1x-gpu-test/openGLDemo
~ cmake .
~ make
```

After the compilation is complete, the gpu-xxxDemo file will be generated in the current directory, which can be run directly: `./gpu-cubeDemo`. If you need to install, you can execute the `make install` command in the current directory (that is, the source code directory), which will automatically install the executable file to the /usr/local/bin/ directory. After the installation is complete, you can directly run: `gpu-cubeDemo`.
Under bianbu-linux, you need to manually set the following environment variables before running:

```bash
XDG_RUNTIME_DIR=/root WAYLAND_DISPLAY=wayland-1 MESA_LOADER_DRIVER_OVERRIDE=pvr ./gpu-cubeDemo
```

The correct running effect is shown in the figure below:

![gpu-cubeTextureDemo](./static/gpu-cubeTextureDemo.gif)

### 4.3 Add one Demo

If you add a file named testDemo.c and want to compile an executable file named testDemo, you can modify CMakeLists.txt as follows:

```Cmake
add_definitions(-DSTB_IMAGE_IMPLEMENTATION)
add_definitions(-DNDEBUG)

# Define a list of common source files
set(COMMON_SOURCE_FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/common/source/xdg-shell-protocol.c
    ${CMAKE_CURRENT_SOURCE_DIR}/common/source/utils_opengles.c
)

include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/common/include/
)

# Add Link Library
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
add_executable(testDemo ${CMAKE_CURRENT_SOURCE_DIR}/testDemo.c ${COMMON_SOURCE_FILES}) # Create a new executable file

target_link_libraries(gpu-cubeDemo ${LINK_LIBRARIES})
target_link_libraries(testDemo ${LINK_LIBRARIES})

# Install, install the executable file to the specified directory
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

Then execute `make` again.

## 3. Vulkan

### 3.1 Introduction

Vulkan is a new generation of cross-platform graphics rendering API released by Khronos Group in 2015, aiming to provide low-level access to modern GPUs to achieve more efficient performance. Compared with OpenGL, Vulkan allows developers to have more granular control over GPU resources, such as precisely specifying when to submit tasks to the GPU, how to allocate memory, and schedule concurrent execution. This direct access to the hardware brings significant benefits. Performance improvement, but the price is that the API is too verbose.

PVR GPU Driver provides a specific implementation of the Vulkan API. Applications can complete drawing tasks by calling the corresponding API.

For detailed usage of Vulkan API, please refer to:

1. [Vulkan Specs](https://registry.khronos.org/vulkan/specs/1.3/html/)

### 3.2 Vulkan Demo

To be continue
