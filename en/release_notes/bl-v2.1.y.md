---
sidebar_position: 3
---

# Bianbu Linux 2.1 Release Notes

## v2.1 release note

Release date: 2025-1-20

Compared to 2.0, 2.1 fixes several issues and provides a new kernel branch k1-bl-v2.1.y (based on 6.6.63), including every modification.

### Major Updates

- Upgraded kernel to 6.6.63
- Added support for hardware random number generator
- Added support for suspend to disk
- Fixed the issue where PCIe memory allocation failed when booting the kernel via EFI
- Fixed the issue of husb239 interrupt loss
- Fixed the issue where husb239 could not enter suspend mode
- Fixed occasional errors in i2c
- Fixed PWM clock configuration errors
- Fixed system crash caused by repeated TCM release
