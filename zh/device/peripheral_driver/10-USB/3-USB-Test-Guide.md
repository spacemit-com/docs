# USB 测试测试指南

## USB 2.0 产品一致性测试

USB 2.0 合规测试是针对 USB 外设产品的测试，如 HUB、 U 盘、等 USB 外设产品。使用 K1 开发板开发的基于 Linux Gadget 驱动的 USB 外设成品也属于 USB 产品。 USB2.0 一致性测试包含三个主要测试领域：

### 功能（ Functional）
功能测试环节通过 USB-IF（ USB 实施者论坛）的工具 USB30CV 执行。该工具会针对《 USB 2.0 规范》第 9 章的要求进行常规测试；此外，对于任何实现了 USB 标准类的产品，该工具还会执行相应的类测试。 USB30CV 工具是仅支持 Windows PC，并且要求是标准 xHCI 规范控制器，可在 USF-IF 官方网站下载： https://www.usb.org/document-library/usb3cv。

*Note：更老版本是 USB20CV，这是基于 EHCI 控制器，目前新款 Windows PC 基本都是采用 XHCI，因此使用 USB30CV 测试即可。

### 电气（ Electrical）
经批准的 USB 2.0 示波器供应商
- Keysight
- Rohde & Schwarz
- Tektronix
- Teledyne LeCroy

合规计划的电气测试环节聚焦于物理层，需使用多种工具。在高速信号质量测试中， USB-IF 仅认可使用经批准的信号质量测试治具所采集的测试数据。此外， USB-IF 仅接受通过其工具 USBET 生成的 USB 2.0 信号质量分析报告。

对于其他电气测试，需要参考 USB-IF 的 Low/Full-speed electrical test specification 和 [USB 2.0 Electrical test specification](https://www.usb.org/document-library/usb-20-electrical-compliance-test-specification-version-107)，并联系经批准的示波器供应商，获取相关测试治具及测试方法说明。通常是采用第三方实验室协助进行测试。

《 USB 2.0 电气合规测试规范》可在 USB-IF 的文档库中下载。

### 互操作性（ Interoperability）

合规计划的互操作性测试环节，重点验证被测产品与 “已知合格的 USB 产品” 之间的协同工作能力。 USB 2.0 的互操作性测试方法与 USB 3.2 采用相同标准。相关的工具资料 [xHCI Interoperability Test Procedures For Peripherals, Hubs and Hosts](https://www.usb.org/document-library/xhci-interoperability-test-procedures-peripherals-hubs-and-hosts-version-096)