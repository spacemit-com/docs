---
sidebar_position: 3
---

# Bianbu Linux 2.2rc4 Release Notes

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