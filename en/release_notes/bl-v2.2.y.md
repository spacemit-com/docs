---
sidebar_position: 3
---

# Bianbu Linux 2.2 Release Notes

## v2.2.2 release note

Release date: 2025-5-23

Compared to 2.1.1, 2.2.2 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Fixed the issue of the remote login interface crashes during remote access 

## v2.2.1 release note

Release date: 2025-5-15

Compared to 2.2, 2.2.1 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support Realtek network phy status detection function 
- Added support for drm yuv444 format
- Added support standard v4l2 via shared dmabuf 
- Added support for spi high frequency clock phase calibration 
- Fix the out-of-bounds issue caused by v2d dmabuf memory exceeding 4GB
- Fix the issue that the baud rate setting fails in some scenarios of UART 
- Fix the issue of chromium crashing on the bianbu cloud platform 
- Fix the issue that rtl8852bs wifi iperf scenario runs away
- Fix the abnormal data transmission problem during the aic8800 wifi firmware download process

## v2.2 release note

Release date: 2025-4-23

Compared to 2.1, 2.2 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for printing esos version function
- Added support for HIDRAW function
- Fixed the issue of hub initialization timing errors when usb asynchronous stanby
- Fixed the issue of occasional errors in USB flash
- Fixed the issue of spi0/spi1 dma transfer erros

## v2.2rc4 release note

Release date: 2025-4-11

Compared to 2.1, 2.2 fixes several issues and provides a new kernel branch k1-bl-v2.2.y (based on 6.6.63), including every modification.

### Major Updates

- Added support for GPU DVFS
- Added support for RTL8125 and RTL8168 modules
- Added support for USB network sharing device function
- Added support for configure ddr priority on USB2.0 
- Fixed the issue of dtb data exception
- Fixed the issue of uart driver drop frame
- Fixed low-probability display issues in system reboot scenarios
- Fixed display errors caused by frequent HDMI hot-swapping in some scenarios
- Fixed low-probability EMAC errors and the last CPU could not enter suspend during suspend/resume
- Fixed compatibility issues during sdio clock switching