# 实测经验、避坑指南与阴性排查结论

[首页](../../../README_CN.md) / [Dell LA280PM240](README_CN.md) | [English](notes-and-pitfalls.md) · [简体中文](notes-and-pitfalls_CN.md)

本文档汇总在 **Dell 280W（`LA280PM240`）** 实机协议验证过程中踩过的关键坑点、适配器固件特性、穷举验证结论与高压安全准则，帮助后续实现者少走弯路。

---

## 1. Dell SVDM 解锁前必须先发起 `DR_Swap` 切换为 DFP

- USB-PD Sink 上电初始数据角色为 `UFP`（`Port Data Role`，Header `bit 5 = 0`）。
- 实测在 `UFP` 状态下直接发送 Dell 私有 SVDM，适配器不会完成解锁。
- **正确做法**：
  1. 先以 `data_role = 0` 请求初始 `20V / 4.50A`（`0x438709C2`）并等待 `PS_RDY`；
  2. 发送 `DR_Swap`（Control `0x09`，`data_role = 0`）；
  3. 收到 `Accept`（`0x03`）后，将后续**所有发包的 PD Header `bit 5` 置为 `1`（DFP）**，同时保持 `bit 8 = 0`（Sink）。

---

## 2. `EPR_KeepAlive` 必须发 `0x00038002`，不能发 `0x00000000`

- 进入 EPR 模式后，`Extended_Control`（`ext=1, count=1, type=0x10`）心跳帧的 32-bit 对象必须包含完整的 2 字节扩展头（`0x8002`：`Chunked=1, Data Size=2`）及 ECDB Type `0x03`（`EPR_KeepAlive`），即 **`0x00038002`**。
- **常见错误**：若发送 `0x00000000`（`Data Size=0`、无 ECDB），`LA280PM240` **连硬件 `GoodCRC` 都不会回复**，并在约 `1.0–1.6s` 后超时撤回 EPR、强行跌回 `5V`。
- **发送节奏**：稳态下每 **`75ms`** 发送一次 `0x00038002`（正常回包 `0x00048002`）；在发出 `EPR_Request` 等待 `PS_RDY` 升压的过渡期（约 `710ms`）内建议暂停发送，待 `PS_RDY` 到达后再恢复。

---

## 3. `EPR_Request` 第 2 对象必须严格等于适配器广播的 PDO 原值

- `EPR_Request`（`Data type=0x09, count=2`）包含两个对象：`[RDO, PDO 原值副本]`。
- 适配器会逐位比对第 2 个对象与对应槽位广播出的 PDO：
  - 若篡改副本（例如填入伪造的 `12V` 底线 `0xD3C078FF`、`0xC190786A` 或 `0x00000000`），适配器不会返回 `Reject`，而是**整条报文完全静默丢弃**。
- 因此务必原封不动回填从 `EPR_Source_Capabilities` 中收到的 32-bit PDO 原值（如 `0x0A81912C`、`0x0008C24D`、`0x000B4247`、`0x000F0247`、`0xD3C096FF`）。

---

## 4. EPR AVS 存在严格的 `15.00V` 底线与 `5.00A` 上限

- **严格 `15.00V` 底线**：在 `25mV` 分辨率下实测（包括初次请求及在 `15V` 活契约内直接降压）：
  - `15.00V`（`0xB3C4B064` + `0xD3C096FF`）→ `Accept` + `PS_RDY`；
  - `14.90V`、`14.75V`、`12.00V` → **全部直接 `Reject`（`0x04`）**，更改任何 RDO 保留标志位均无法绕过。
- **严格 `5.00A` 电流上限**：虽然 Dell 解锁后的 EPR 固定档允许 `5.89A`（28V）与 `5.83A`（36V/48V），但 USB-PD 3.1 规范规定 EPR AVS 的工作电流受限于：
  $$I_{\text{max}} \le \min\left(5.00\text{ A},\; \left\lfloor \frac{\text{PDP}_W}{V} \right\rfloor_{50\text{mA}}\right)$$
  AVS RDO 请求电流一旦超过 `5.00A` 会被适配器直接拒绝。
- **如何输出 `12.00–14.98V`**：不能走 EPR AVS，必须走**分支 A（隐藏 SPR PPS `12.00–20.00V / 5.30A`，`0xC190786A`）**——通过在 130W 解锁态应答 `0xE004B12C` + SKEDB `0x20` 触发。

---

## 5. Dell 私有 SVDM 命令穷举（`0x00–0x1F`）的阴性结论

在 130W SPR 与 EPR 活契约两种状态下，对 5-bit SVDM 命令空间（`0x00–0x1F`、裸头与多对象载荷、SVDM v1.0/v2.0）进行完整扫测后确认：

- **`SVID 0x413C` 仅标准命令 `0x03`、`0x04` (pos 1)、`0x05` (pos 1) 会返回 ACK**：
  - `0x03` (`Discover Modes`)：ACK `0x413CA843, 0x00000002`
  - `0x04` (`Enter Mode`, `pos=1`)：ACK `0x413CA944`（解锁 130W SPR 与 280W EPR 能力）
  - `0x04` (`Enter Mode`, `pos=2`)：**NAK** `0x413CAA84`（不存在第二个模式）
  - `0x05` (`Exit Mode`, `pos=1`)：ACK `0x413CA945`（**会将 `PDO4` 从 `6.50A` 退回未解锁的 `4.80A`**）
- **`0x06` (`Attention`)**：始终静默无应答。
- **厂商命令 `0x07–0x1F`（含 `0x12`）**：全部返回 **NAK**（`0x413CA887..0x413CA89F`，如 `0x12` 返回 `0x413CA892`）。真实工作站抓包中也发过 `0x413C8112` 并同样收到 `0x413CA892` NAK，重放该命令不会增加新档位，非解锁必需步骤。

---

## 6. 适配器 PD 静默锁定与软件恢复方式

- 若初始握手超时（错过 `24ms` `tSenderResponse` 窗口）或连续收到非法请求，`LA280PM240` 会进入 **PD 静默状态**：不再广播 `Source_Capabilities`，且对 `Get_Source_Cap` / `Soft_Reset` 均无应答。
- **恢复方法**：
  1. 彻底撤除 CC 引脚 `Rd` 下拉 **`600ms–1500ms`**，待 `VBUS` 完全归零后重新挂载单路 `Rd`；
  2. 若连续多次触发 EPR 异常，适配器内部保护可能要求断开并冷却 **`10–25s`** 才能重新恢复 PD 广播。

---

## 7. 解析报文时切勿清除 Header `bit 15`（Extended 标志）

- PD Header 的 `bit 15` 是标准 **Extended Message** 标志位。
- `Get_Source_Cap_Extended`（`0x11`）返回的 25 字节 `SCEDB` 带有扩展头 `0x8019`。若解析器错误清除了 `bit 15`，会把扩展头与 `SCEDB` 字节流强行拆解成 6–7 个假 PDO，从而产生"线上在没触发前就发过 `12–20V 5.30A` PPS"的误读（实际上 `0x6A` 只是 `SCEDB` 偏移 23 的 `SPR Source PDP = 106W` 字段）。
- 正确实现应先判断 `(header >> 15) & 1`，按扩展消息跳过前 2 字节扩展头再解析载荷。

---

## 8. 被动消息应答规范

- **及时应答 `Get_Sink_Cap`（`0x08`）与 `Get_Sink_Cap_Extended`（`0x16`）**：忽略适配器的能力查询可能导致握手停滞或静默。
- **拒绝非预期角色切换**：在协商期间收到 Source 发来的 `DR_Swap`（`0x09`）、`PR_Swap`（`0x0A`）或 `VCONN_Swap`（`0x0B`）请求时，直接回 `Reject`（Control `0x04`）。
- **保守处理 `Soft_Reset`（`0x0D`）**：收到 `Soft_Reset` 后切勿盲目在原状态上直接重发高压请求，应重置本地契约状态并通过 CC 软断开重新建链。

---

## 9. 高压安全操作准则

1. **隔离负载**：测试 `> 20V`（`28V / 36V / 48V` 或 `AVS`）输出前，务必断开不耐压的下游设备。
2. **EPR 先落 5V**：重组 `EPR_Source_Capabilities` 后建议先请求 `5V`（`[0x1344B12C, 0x0A81912C]`），再由用户或上层确认后升压至高压档位。
3. **退出先降 5V 再停心跳**：断开或退出前先发送 `5V` 请求并等待 `PS_RDY` 确认 `VBUS` 跌落，最后停止 `EPR_KeepAlive`。
