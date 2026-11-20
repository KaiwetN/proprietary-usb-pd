# 私有 USB-PD 协议研究

[English](README.md) | [简体中文](README_CN.md)

[![License: CC BY 4.0](https://img.shields.io/badge/Docs-CC%20BY%204.0-lightgrey.svg)](LICENSE-CC-BY-4.0)
[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](LICENSE-MIT)

本仓库整理并公开各厂商 **USB Power Delivery（USB-PD 3.0 / 3.1 / 3.2）私有协议扩展** 经实机抓包与物理链路验证的技术规范，涵盖结构化/非结构化 VDM（SVDM / UVDM）解锁、能力查询与身份握手（`Sink_Capabilities` / `SKEDB` / `SCEDB`）、高功率扩展档位以及隐藏电压/电流输出通道。

> [!WARNING]
> **高压安全提示**：本仓库记录的部分私有 USB-PD 协议可诱骗输出最高 **48V / 280W** 直流高压大功率。测试 `> 20V` 档位前务必断开不耐压的下游负载；调试新实现时建议先停留在安全 `5V` 契约（例如进入 EPR 后先请求 5V）再按需切换高压；退出或断开前务必先将契约回落至 `5V`，再停止保活心跳。

---

## 已收录协议

| 厂商 | 设备 / 专题 | VID / SVID | 核心内容 | 文档入口 |
|---|---|---|---|---|
| **Dell** | **`LA280PM240`（280W 适配器）** | `0x413C` | `DR_Swap` + SVDM Mode 1 解锁、130W SPR（`20V/6.5A`）、隐藏 `12–20V / 5.30A` PPS（`SPR AVS Sink APDO` + `SKEDB`）、280W EPR（`28/36/48V` + `255W AVS`） | [English](protocols/dell/la280pm240-280w/README.md) / [简体中文](protocols/dell/la280pm240-280w/README_CN.md) |

---

## 仓库目录结构与约定

各厂商协议按 `protocols/<厂商>/<设备或专题>/` 组织：

```text
protocols/
└── <vendor>/
    └── <device-or-topic>/
        ├── README.md                 # 设备概览、能力对照表与文档索引 (EN)
        ├── README_CN.md              # 设备概览、能力对照表与文档索引 (ZH)
        ├── protocol-flow.md          # 分阶段握手流程与时序图 (EN)
        ├── protocol-flow_CN.md       # 分阶段握手流程与时序图 (ZH)
        ├── packet-reference.md       # Header、PDO、RDO、VDM 逐位编码与时序表 (EN)
        ├── packet-reference_CN.md    # Header、PDO、RDO、VDM 逐位编码与时序表 (ZH)
        ├── notes-and-pitfalls.md     # 实测避坑、阴性结论与安全准则 (EN)
        └── notes-and-pitfalls_CN.md  # 实测避坑、阴性结论与安全准则 (ZH)
```

### 线缆报文表示约定

- USB-PD 在物理层按**小端序（Little-Endian）**传输 16-bit 报文头与 32-bit 数据对象。
- 本仓库所有 32-bit 对象（`PDO`、`RDO`、`VDM`、`Extended_Control`）均按标准 32-bit 十六进制字（`0xXXXXXXXX`）给出，位域编号遵循 USB-PD 规范（`bit 31` 为最高位 MSB，`bit 0` 为最低位 LSB）。
- 多字节扩展数据块（`SCEDB`、`SKEDB`）按线上传输顺序标注 0 基字节偏移。

---

## 开源许可说明

本仓库采用**文档与代码分离**的双许可模式：

- **文档部分（`*.md`、`protocols/**`）**：采用 [知识共享署名 4.0 国际许可协议 (CC BY 4.0)](LICENSE-CC-BY-4.0)。允许自由转载、翻译、引用及商业使用，只需按要求注明原作者署名与仓库出处。
- **代码片段与编解码公式**：采用 [MIT 许可证](LICENSE-MIT)。

完整许可条款请参阅 [LICENSE](LICENSE)。
