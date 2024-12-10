# Graphics Driver Framework

## 1. Overall framework

![Linux Graphical Framework](./static/linuxGraphicsFramework_en.png#pic_center)

## 2. Environment configuration and dependencies

### 2.1 kernel

Enter the Linux kernel source directory and enable PowerVR in config:

```bash
CONFIG_POWERVR_ROGUE=y #enable PowerVR_rogue
```

Or configure in menuconfig: Execute make menuconfig, as shown in the figure below, enter the configuration interface, and enable the PowerVR GPU option in Device Drivers -> Graphics support -> PowerVR GPU.

```shell
<*>   Lontium LT9711 DSI to DP
< > null drm disp
<*> PowerVR GPU #enable PowerVR GPU
[ ] Enable legacy drivers (DANGEROUS)  ----
    Frame buffer Devices  --->
```

### 2.2 PVR DDK

Due to copyright issues, the PVR GPU driver development kit (Device Development Kit, DDK) cannot be provided. The user-level closed source code of the PVR GPU driver uses the so dynamic library to provide the API implementation of OpenGLES, Vulkan and OpenCL. The way to enable the PVR GPU driver on bianbu-linux and bianbu-desktop is as follows:

1. On bianbu-linux, you need to set BR2_PACKAGE_IMG_GPU_POWERVR=y in the config file when compiling the build root; or execute make menuconfig in the bianbu-linux source directory and enable the img-gpu-powervr option in the External options -> Bianbu config path.

    ```shell
    [*] rtk hciattach
        *** spacemit mpp package ***
    -*- spacemit mpp
    [*] img-gpu-powervr #enable img-gpu-powervr
        Output option (Wayland)  --->
    [ ]   install examples

2. On bianbu-desktop, closed-source code with copyright issues is provided in the form of so dynamic libraries through the img-gpu-powervr deb package. Users can use apt install img-gpu-powervr to obtain the PVR GPU driver. It should be noted that the GPU kernel driver and user driver versions must be consistent. The version check method is as follows:

    ```bash
    # View kernel driver
    ➜  ~ journalctl -b | grep "Initialized pvr"
    11月 21 14:11:37 spacemit-k1-x-MUSE-Pi-board kernel: [drm] Initialized pvr 24.2.6603887 20170530 for cac00000.imggpu on minor 1 # Version 24.2
    # View User Layer Drivers
    ➜  ~ dpkg -s img-gpu-powervr
    Package: img-gpu-powervr
    Status: install ok installed
    Priority: optional
    Section: graphics
    Installed-Size: 70314
    Maintainer: bianbu <bo.deng@spacemit.com>
    Architecture: all
    Version: 24.2-6603887bb1 # Version 24.2
    ```

### 2.3 Mesa3D

Mesa3D provides the OpenGLES API interface and software rendering implementation, connecting to the application's call to the OpenGLES rendering API and connecting to the PowerVR GPU driver.

1. On bianbu-linux, you need to configure the config file as follows to enable mesa3d:

    ```bash
    Symbol: BR2_PACKAGE_MESA3D [=y]
    Symbol: BR2_PACKAGE_MESA3D_DRIVER [=y] 
    Symbol: BR2_PACKAGE_MESA3D_GALLIUM_DRIVER [=y]
    Symbol: BR2_PACKAGE_MESA3D_GALLIUM_DRIVER_PVR [=y]
    Symbol: BR2_PACKAGE_MESA3D_GBM [=y]
    Symbol: BR2_PACKAGE_MESA3D_OPENGL_EGL [=y]
    ```

    Or manually check it when executing make menuconfig, under Target packages > Graphic libraries and applications (graphic/text) > mesa3d path:

    ```shell
    -*-   Gallium pvr driver
    *** Gallium VDPAU state tracker needs X.org and gallium drivers r300, r600, radeonsi or nouveau ***
    *** Vulkan drivers ***
    *** Off-screen Rendering ***
    [ ]   OSMesa (Gallium) library
        *** OpenGL API Support ***
    -*-   gbm
    *** OpenGL GLX support needs X11 ***
    -*-   OpenGL EGL
    [ ]   OpenGL ES
    ```

2. On bianbu-desktop , the PowerVR GPU hardware dependencies are available from the Mesa deb packages in the bianbu repository and are installed in the following order:

    ```bash
        "libgl1-mesa-dev_*_riscv64.deb"
        "libegl1-mesa_*_riscv64.deb"
        "libegl1-mesa-dev_*_riscv64.deb"
        "libglapi-mesa_*_riscv64.deb"
        "libgbm1_*_riscv64.deb"
        "libgbm-dev_*_riscv64.deb"
        "libegl-mesa0_*_riscv64.deb"
        "libgl1-mesa-dri_*_riscv64.deb"
        "libgles2-mesa_*_riscv64.deb"
        "libgles2-mesa-dev_*_riscv64.deb"
        "libgl1-mesa-glx_*_riscv64.deb"
        "libglx-mesa0_*_riscv64.deb"
        "libosmesa6_*_riscv64.deb"
        "libosmesa6-dev_*_riscv64.deb"
        "libwayland-egl1-mesa_*_riscv64.deb"
        "mesa-common-dev_*_riscv64.deb"
    ```

    Additionally, you need to install libglvnd: `sudo apt install libglvnd`

## 3. Additional Notes

### 3.1 BlobCache

The GPU's BlobCache is a data cache mechanism for storing and reusing GPU compilation results. This cache mechanism is designed to improve performance, reduce the time of repeated calculations and data transfer, and thus speed up rendering. You can enable or disable the BlobCache function by configuring /etc/powervr.ini. vim /etc/powervr.ini, set EnableBlobCache=1 to enable the BlobCache function.

```bash
[default]
EnableBlobCache=1 #enable BlobCache

[mpv]
EnableWorkaroundMPVScreenTearing=1

[totem]
EnableWorkaroundMPVScreenTearing=1

[gst-launch-1.0]
EnableWorkaroundMPVScreenTearing=1
```

### 3.2 GPU node debugging

By monitoring the GPU node, you can view the running status of the GPU in real time:

```shell
~ su
~ cd /sys/kernel/debug/pvr/
~ watch cat status #(Press Ctrl + C to exit)
```

As shown in the follows:

```shell
Every 2.0s: cat status                                    spacemit-k1-x-deb1-board: Thu Nov 28 09:00:00 2024

Driver Status:   OK

Device ID: 0:128
Firmware Status: OK
Server Errors:   24
HWR Event Count: 0
CRR Event Count: 0
SLR Event Count: 0
WGP Error Count: 0
TRP Error Count: 0
FWF Event Count: 0
APM Event Count: 0
GPU Utilisation: 0%
DM Utilisation:  VM0
           2D:   0%
         GEOM:   0%
           3D:   0%
          CDM:   0%
          RAY:   0%
        GEOM2:   0%
```
