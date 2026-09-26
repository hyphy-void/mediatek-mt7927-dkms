# Linux 6.8 MT6639 蓝牙移植初步计划

日期：2026-09-26。开发分支：`dev/mt6639-bluetooth-linux-6.8`。
状态：仅完成资料调查与计划；未进行驱动移植、构建或安装验证。

## 目标

在 Ubuntu 22.04.5、`6.8.0-138-generic`、Secure Boot 开启状态下，为 MSI B850MPOWER 的 MT7927 / MT6639（USB `0489:e110`）实现可维护的蓝牙支持。先不移植 Wi-Fi，不改动正常工作的 r8126、nct6687d 和 NVIDIA 驱动。

## 文档依据与适用边界

- [Linux 6.8 外部模块构建](https://www.kernel.org/doc/html/v6.8/kbuild/modules.html)：使用目标内核的 Kbuild、配置与 Module.symvers；仅 modules_prepare 不能提供启用 CONFIG_MODVERSIONS 时所需的符号版本信息。
- [Linux 6.8 固件请求接口](https://www.kernel.org/doc/html/v6.8/driver-api/firmware/request_firmware.html)：核实请求上下文、设备引用、错误返回与 release_firmware；固件可读取不代表设备端下载成功。
- [Linux 6.8 模块签名](https://www.kernel.org/doc/html/v6.8/admin-guide/module-signing.html)：签名和信任链是加载条件；不得在签名后修改模块内容。
- [Linux 6.8 USB 电源管理](https://www.kernel.org/doc/html/v6.8/driver-api/usb/power-management.html)：检查 USB runtime PM、挂起恢复及接口引用的配对处理。
- [Linux v6.8 btusb 源码](https://github.com/torvalds/linux/blob/v6.8/drivers/bluetooth/btusb.c)及 [btmtk 源码](https://github.com/torvalds/linux/blob/v6.8/drivers/bluetooth/btmtk.c)：用于理解原始初始化路径，后者仍需下载后完整阅读。

上游 v6.8 文档不是 Ubuntu ABI 的替代品。最终以本机头文件、Module.symvers 以及对应 Ubuntu 内核源码为准。

当前仓库从较新内核提取 Bluetooth 源码，并默认连同 mt76 构建。已有 pre-7.0 补丁处理分配函数及 hci_discovery_active 导出差异，尚未验证 6.8。linux-stable 符号链接目标缺失，不能依赖维护者的私有分支流程。

## 阶段一：建立可复现的源码与差异清单

- 记录 Git 提交、目标 ABI、工具链、内核配置及 headers/Module.symvers 可用性。
- 获取对应 Ubuntu 6.8 源码和仓库指定的来源内核蓝牙源码，记录版本与校验信息。
- 核对 MT6639 设备匹配、WMT 命令、固件文件名和下载协议、ISO 接口、复位与清理路径。
- 对照目标头文件和导出符号，列出新源码依赖而 6.8 缺少或签名不同的 API、结构体字段和 quirks。
- 同时检查 btusb 对 btrtl/btintel/btbcm 的跨模块接口，不只检查 btmtk。

产物：接口差异清单、源码来源记录和未经修改源码的构建失败日志。

## 阶段二：蓝牙独立准备与构建

- 增加明确的蓝牙独立模式，例如 BUILD_WIFI=no；默认行为保持与上游一致。
- 蓝牙模式只准备 Bluetooth 源码与必要固件，不要求下载或编译 Wi-Fi 源码。
- DKMS 的 MAKE 与 BUILT_MODULE_NAME 数组必须一致，继续支持目标 kernelver，并兼顾已有 BUILD_BT 开关。
- 将所有修改保存为受版本控制的补丁/兼容文件；修改准备流程后从干净生成目录验证可复现性。

验收：普通用户能准备源码并执行蓝牙构建，独立模式不产生 mt76 构建依赖。

## 阶段三：选择最小兼容方案

先评估“较新 btusb/btmtk + 6.8 兼容层”。按真实编译错误及符号差异逐项适配，不能简单关闭关键初始化逻辑或伪造核心 API。

如果新驱动深度依赖较新蓝牙核心，转为“对应 Ubuntu 6.8 的 btusb/btmtk + 最小 MT6639 支持回移”。两条路线先比较成本和行为，再选择一种，不默认替换整个 bluetooth 核心。

重点检查固件数据边界、命令长度、超时、URB 生命周期、工作队列取消、断开及挂起恢复。仅添加 USB ID 不构成完整移植。

验收：两个模块通过目标 ABI 的编译与 modpost，无未解决符号；vermagic 和依赖正确。兼容修改应在可用的 6.17+ 头文件上回归构建，未测试版本必须标注。

## 阶段四：固件与 DKMS 集成

- 从可追溯来源获取匹配固件，核实许可、提取脚本、实际请求路径及校验值；不提交未经许可的固件二进制。
- 验证本机 DKMS 2.8.7 的配置解析及签名流程，沿用已注册 MOK。
- 准备安装包或脚本、旧模块备份、DKMS 卸载/depmod/initramfs 恢复步骤；安装内容应明确只覆盖 btusb/btmtk。
- 安装后检查实际解析路径、signer 及是否出现签名拒绝或 symbol version mismatch。

验收：DKMS 能为明确指定的目标 ABI 构建和安装两个模块，自动安装配置有效。系统写入依照用户授权执行；sudo 密码由用户在终端输入。

## 阶段五：本机硬件验证

依次验证并记录：

1. USB 设备稳定存在，初始化完成，BlueZ 列出有效控制器。
2. 上电与扫描成功，配对并实际连接一个可用设备。
3. 按实际设备测试耳机音频或手柄输入，不以“已配对”替代功能验证。
4. 正常重启、冷启动、休眠恢复；必要时检查双系统热重启后的固件状态。
5. 已有网卡、显卡、硬件监控及其他蓝牙设备无功能回退。
6. 实际执行一次卸载恢复测试，确认可以回到发行版驱动。

固件锁死时停止反复重载，记录现象并由用户执行必要的断电恢复。不会为测试自行改 BIOS 或禁用 Secure Boot。

## PR 拆分与完成标准

建议拆为：蓝牙独立构建；6.8 API 兼容补丁；固件/安装说明与测试证据。保持每个提交可审查，保留补丁许可证及来源。

本机 AGENT.md、AGENTS.md 与此初步计划留在 fork，最终上游 PR 只包含适用的实现及通用文档。先与维护者确认是否接受扩大旧内核支持范围；未合并不影响 fork 的本地使用。

“可用”要求目标 ABI、Secure Boot 开启时模块加载、扫描和实际配对连接、重启后仍正常。休眠恢复、其他内核或其他硬件未验证时单独列出，不宣称全面兼容。

下一步：只执行阶段一与蓝牙独立构建准备，先得到实际 API 差异和构建日志，再细化实现任务；本计划不承诺编译或硬件验证必然成功。
