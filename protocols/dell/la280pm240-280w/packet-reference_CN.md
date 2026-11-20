# Dell 280W (`LA280PM240`) 报文与位域编码参考

[首页](../../../README_CN.md) / [Dell LA280PM240](README_CN.md) | [English](packet-reference.md) · [简体中文](packet-reference_CN.md)

本文档汇总 **Dell 280W（`LA280PM240`）** 诱骗与握手流程中涉及的所有 USB-PD 报文头、PDO、RDO、SVDM、SCEDB、SKEDB 与 Extended Control 的逐位编码定义及实测十六进制常量。

---

## 1. 16-bit USB-PD 报文头（Message Header）

USB-PD 报文头为 16-bit 小端整数，紧跟其后发送 `Count` 个 32-bit 数据对象。

| 位域 | 字段 | 本协议取值说明 |
|---|---|---|
| `15` | **Extended** | `0` = Control / Data 消息；`1` = Extended 扩展消息（`SCEDB`、`SKEDB`、`EPR_Source_Capabilities`、`Extended_Control`） |
| `14:12` | **Number of Data Objects** | `0` = Control 消息；`1..7` = 数据对象个数（扩展消息分片请求在 `ext=1` 时也使用 `count=1` 携带 2 字节扩展头） |
| `11:9` | **MessageID** | `0..7` 递增发送计数器，每成功收到对端 `GoodCRC` 后加 1（注意 `GoodCRC` 本身不消耗接收序列号） |
| **`8`** | **Port Power Role** | `0` = Sink（Sink 端发包恒为 `0`）；`1` = Source（适配器发包） |
| `7:6` | **Spec Revision** | `2`（`0b10`）= USB-PD 3.0 / 3.1 / 3.2 |
| **`5`** | **Port Data Role** | `0` = UFP（刚上电至发起 `DR_Swap` 前）；**`1` = DFP（`DR_Swap` 收到 `Accept` 后所有发包必须置 1）** |
| `4:0` | **Message Type** | 见下方消息类型对照表 |

> [!IMPORTANT]
> **关键位域避坑**：`bit 8` 是 **Port Power Role（电源角色）**，`bit 5` 是 **Port Data Role（数据角色）**，切勿弄反。同时解析收包时**绝对不要清除 `bit 15`（Extended 标志）**，否则会把 25 字节 `SCEDB` 误解析为假的 PDO 列表。

```c
static inline uint16_t pd_header(uint8_t count, uint8_t msg_id, uint8_t type,
                                 uint8_t power_role, uint8_t data_role,
                                 uint8_t spec_rev, uint8_t extended) {
    return (uint16_t)(((extended & 1u) << 15) | ((count & 7u) << 12) |
                      ((msg_id & 7u) << 9) | ((power_role & 1u) << 8) |
                      ((spec_rev & 3u) << 6) | ((data_role & 1u) << 5) |
                      (type & 0x1Fu));
}
```

---

## 2. 协议流程涉及的消息类型（Message Type）

消息类型号（`header & 0x1F`）通过 `Extended`（`bit 15`）与 `Count`（`bits 14:12`）共同区分含义：

| Type | Control (`ext=0, count=0`) | Data (`ext=0, count>0`) | Extended (`ext=1`) |
|---|---|---|---|
| `0x01` | `GoodCRC` | `Source_Capabilities` | `Source_Capabilities_Extended` (`SCEDB`) |
| `0x03` | `Accept` | — | — |
| `0x04` | `Reject` | `Sink_Capabilities` | — |
| `0x06` | `PS_RDY` | — | — |
| `0x07` | `Get_Source_Cap` | — | — |
| `0x08` | `Get_Sink_Cap` | — | — |
| `0x09` | `DR_Swap` | `EPR_Request`（`count=2`） | — |
| `0x0A` | `PR_Swap` | `EPR_Mode`（`count=1`） | — |
| `0x0B` | `VCONN_Swap` | — | — |
| `0x0C` | `Wait` | — | — |
| `0x0D` | `Soft_Reset` | — | — |
| `0x0F` | — | `Vendor_Defined` (`VDM`) | `Sink_Capabilities_Extended`（`SKEDB`，`count=7`） |
| `0x10` | `Not_Supported` | — | `Extended_Control`（`EPR_KeepAlive` / `Ack`） |
| `0x11` | `Get_Source_Cap_Extended` | — | `EPR_Source_Capabilities`（分片 0 与 1） |
| `0x16` | `Get_Sink_Cap_Extended` | — | — |

---

## 3. Source PDO 位域与适配器实测值

### 3.1 Fixed Supply PDO（`bits 31:30 = 0b00`）

| 位域 | 字段 | 单位 / 含义 |
|---|---|---|
| `31:30` | Supply Type | `0b00` = 固定档（Fixed Supply） |
| `27` | Unconstrained Power | `PDO1`（`0x0A81912C`）置 `1` |
| `25` | Dual-Role Data | `PDO1`（`0x0A81912C`）置 `1` |
| `23` | EPR Mode Capable | `PDO1`（`0x0A81912C`）置 `1` |
| `19:10` | 电压 Voltage | 步进 `50 mV`（`mv / 50`） |
| `9:0` | 最大电流 Max Current | 步进 `10 mA`（`ma / 10`） |

### 3.2 SPR PPS APDO（`bits 31:28 = 0xC`）

| 位域 | 字段 | 单位 / 含义 |
|---|---|---|
| `31:28` | 类型 / 子类 | `0b1100`（`0xC`）= SPR PPS APDO |
| `27` | PPS Power Limited | `0` = 恒流限幅（`0xC190786A` 该位为 `0`） |
| `24:17` | 最高电压 Max Voltage | 步进 `100 mV`（`0xC8` = `200` → `20.0V`） |
| `15:8` | 最低电压 Min Voltage | 步进 `100 mV`（`0x78` = `120` → `12.0V`） |
| `6:0` | 最大电流 Max Current | 步进 `50 mA`（`0x6A` = `106` → `5.30A`） |

### 3.3 EPR AVS APDO（`bits 31:28 = 0xD`）

| 位域 | 字段 | 单位 / 含义 |
|---|---|---|
| `31:28` | 类型 / 子类 | `0b1101`（`0xD`）= EPR AVS APDO |
| `25:17` | 最高电压 Max Voltage | 步进 `100 mV`（`0x1E0` = `480` → `48.0V`） |
| `15:8` | 最低电压 Min Voltage | 步进 `100 mV`（`0x96` = `150` → `15.0V`） |
| `7:0` | PDP 额定功率 | 步进 `1 W`（`0xF0` = `240W`，`0xFF` = `255W`） |

### 3.4 适配器完整 PDO 对照表

| 阶段 / 语境 | 槽位 | 原始十六进制 | 解码档位 |
|---|---|---|---|
| **SPR（未解锁初始态）** | PDO 1 | `0x0A81912C` | Fixed `5.00V / 3.00A`（`15.00W`） |
| | PDO 2 | `0x0002D12C` | Fixed `9.00V / 3.00A`（`27.00W`） |
| | PDO 3 | `0x0004B12C` | Fixed `15.00V / 3.00A`（`45.00W`） |
| | PDO 4 | `0x000641E0` | Fixed `20.00V / 4.80A`（`96.00W`） |
| **SPR（`0x413C` Mode 1 解锁后）** | PDO 4 | `0x0006428A` | **Fixed `20.00V / 6.50A`（`130.00W`）** |
| **SPR（应答 Sink AVS 触发隐藏档）** | PDO 5 | `0xC190786A` | **SPR PPS `12.00–20.00V / 5.30A`（`106.00W`）** |
| **EPR（未解锁标准态）** | PDO 11 | `0xD3C096F0` | EPR AVS `15.0–48.0V`，PDP `240W` |
| **EPR（`0x413C` Mode 1 解锁后）** | PDO 1–3 | `0x0A81912C`、`0x0002D12C`、`0x0004B12C` | Fixed `5V/3A`、`9V/3A`、`15V/3A` |
| | PDO 4 | `0x000641F4` | Fixed `20.00V / 5.00A`（`100.00W`） |
| | PDO 5–7 | `0x00000000` | 空槽（未启用） |
| | PDO 8 | `0x0008C24D` | **Fixed `28.00V / 5.89A`（`164.92W`）** |
| | PDO 9 | `0x000B4247` | **Fixed `36.00V / 5.83A`（`209.88W`）** |
| | PDO 10 | `0x000F0247` | **Fixed `48.00V / 5.83A`（`279.84W`）** |
| | PDO 11 | `0xD3C096FF` | **EPR AVS `15.0–48.0V`，PDP `255W`** |

---

## 4. Sink 能力应答与 SKEDB（触发隐藏 PPS 档）

### 4.1 `Sink_Capabilities`（`Data type=0x04, count=5`）

| 槽位 | 原始值 | 含义 |
|---|---|---|
| PDO 1 | `0x0001912C` | Fixed `5.00V / 3.00A` |
| PDO 2 | `0x0002D12C` | Fixed `9.00V / 3.00A` |
| PDO 3 | `0x0004B12C` | Fixed `15.00V / 3.00A` |
| PDO 4 | `0x0006412C` | Fixed `20.00V / 3.00A` |
| **PDO 5** | **`0xE004B12C`** | **PD 3.2 `SPR AVS Sink APDO`**：`bits 31:28 = 0xE`（`1110b`），`bits 19:10 = 300`（`9–15V @ 3.00A`），`bits 9:0 = 300`（`15–20V @ 3.00A`） |

### 4.2 `Sink_Capabilities_Extended`（`Extended ext=1, type=0x0F, count=7`）

共发送 7 个小端 32-bit 对象（2 字节扩展头 + 24 字节 SKEDB + 2 字节尾补零 = 28 字节）：

```text
Word 0: 0x00000018   Word 1: 0x00000000   Word 2: 0x00000000   Word 3: 0x00000001
Word 4: 0x20000000   Word 5: 0x003C2805   Word 6: 0x00000000
```

| 24B SKEDB 字节偏移 | 字段 | 值 | 说明 |
|---|---|---|---|
| `0..1` | VID | `0x0000` | Sink Vendor ID 占位（按需填写） |
| `2..3` | PID | `0x0000` | Sink Product ID 占位（按需填写） |
| `4..7` | XID | `0x00000000` | USB-IF XID |
| `8..9` | FW / HW 版本 | `0x00 / 0x00` | 固件/硬件版本 |
| `10` | SKEDB 版本 | `0x01` | SKEDB v1 |
| `11..16` | 负载 / 合规 / 电池字段 | `0x00` | 置零 |
| **`17`** | **Sink Modes** | **`0x20`** | **Bit 5（`0x20`）= AVS Support（核心触发位）** |
| `18..20` | SPR Min / Op / Max PDP | `0x05 / 0x28 / 0x3C` | `5W / 40W / 60W` |
| `21..23` | EPR Min / Op / Max PDP | `0x00 / 0x00 / 0x00` | `0W`（SPR Sink） |

*（报文首 2 字节为非分片扩展头 `0x0018`：`Chunked=0, Data Size=24`。）*

---

## 5. Request Data Objects (RDO) 编码表

### 5.1 RDO 标志位（`bits 27:22`）

| 位 | 掩码 | 含义 | 本协议取值 |
|---|---|---|---|
| `25` | `0x02000000` | USB Communications Capable | `1` |
| `24` | `0x01000000` | No USB Suspend | `1` |
| `23` | `0x00800000` | Unchunked Extended Messages Supported | SPR 与 EPR AVS 置 `1`（`0x03800000` / `0x03C00000`）；EPR 固定档置 `0`（`0x03400000`） |
| `22` | `0x00400000` | EPR Mode Capable | 仅跑 SPR 置 `0`（`0x03800000`）；准备进 EPR 或处于 EPR 内置 `1`（`0x03C00000` / `0x03400000`） |

### 5.2 计算公式与实测向量

1. **Fixed RDO**（`bits 19:10` 为工作电流、`bits 9:0` 为最大电流，单位 `10 mA`）：
   `RDO = (pos << 28) | flags | ((I_mA / 10) << 10) | (I_mA / 10)`
2. **SPR PPS RDO**（`bits 20:9` 为电压步进 `20 mV`，`bits 6:0` 为电流步进 `50 mA`）：
   `RDO = (pos << 28) | 0x03800000 | ((V_mV / 20) << 9) | (I_mA / 50)`
3. **EPR AVS RDO**（`bits 20:9` 为电压步进 `25 mV`，`bits 6:0` 为电流步进 `50 mA`）：
   `RDO = (pos << 28) | 0x03C00000 | ((V_mV / 25) << 9) | (I_mA / 50)`

| 报文类型 | 目标契约 | RDO 十六进制 | 第 2 对象（EPR PDO 副本） |
|---|---|---|---|
| `Request`（`0x02`, 1 obj） | 初始 `20.00V / 4.50A`（pos 4, SPR） | `0x438709C2` | — |
| `Request`（`0x02`, 1 obj） | 解锁 `20.00V / 6.50A`（pos 4, SPR） | `0x438A2A8A` | — |
| `Request`（`0x02`, 1 obj） | 解锁 `20.00V / 6.50A`（pos 4, EPR Cap=1） | `0x43CA2A8A` | — |
| `Request`（`0x02`, 1 obj） | 安全回落 `5.00V / 3.00A`（pos 1, SPR） | `0x1384B12C` | — |
| `Request`（`0x02`, 1 obj） | PPS `12.00V / 5.30A`（pos 5） | `0x5384B06A` | — |
| `Request`（`0x02`, 1 obj） | PPS `12.00V / 3.00A`（pos 5） | `0x5384B03C` | — |
| `Request`（`0x02`, 1 obj） | PPS `16.00V / 5.00A`（pos 5） | `0x53864064` | — |
| `EPR_Request`（`0x09`, 2 obj） | 安全回落 `5.00V / 3.00A`（pos 1, EPR） | `0x1344B12C` | `0x0A81912C` |
| `EPR_Request`（`0x09`, 2 obj） | Fixed `28.00V / 5.89A`（pos 8, EPR） | `0x8349364D` | `0x0008C24D` |
| `EPR_Request`（`0x09`, 2 obj） | Fixed `36.00V / 5.83A`（pos 9, EPR） | `0x93491E47` | `0x000B4247` |
| `EPR_Request`（`0x09`, 2 obj） | Fixed `48.00V / 5.83A`（pos 10, EPR） | `0xA3491E47` | `0x000F0247` |
| `EPR_Request`（`0x09`, 2 obj） | AVS `15.00V / 5.00A`（pos 11, EPR） | `0xB3C4B064` | `0xD3C096FF` |
| `EPR_Request`（`0x09`, 2 obj） | AVS `20.00V / 5.00A`（pos 11, EPR） | `0xB3C64064` | `0xD3C096FF` |
| `EPR_Request`（`0x09`, 2 obj） | AVS `48.00V / 5.00A`（pos 11, EPR） | `0xB3CF0064` | `0xD3C096FF` |

---

## 6. SVDM 头定义与 Dell `0x413C` 命令表

### 6.1 Structured VDM Header（`32-bit`）

| 位域 | 字段 | 说明 |
|---|---|---|
| `31:16` | **SVID / VID** | `0xFF00`（标准发现命令）或 `0x413C`（Dell 私有 SVID） |
| `15` | **VDM Type** | `1` = Structured VDM |
| `14:13` | **VDM 主版本** | `01b` = SVDM 2.0（`0x2000`）；叠加 bit 15 为 `0xA000` |
| `12:11` | **VDM 次版本** | 请求侧填 `00b`；适配器应答报文 `bit 11 = 1`（`0xA8xx` / `0xA9xx`） |
| `10:8` | **Object Position** | 发现命令为 `0`；Enter/Exit Dell Mode 1 为 `1` |
| `7:6` | **Command Type** | `0` (`00b`) = REQ，`1` (`01b`) = ACK，`2` (`10b`) = NAK，`3` (`11b`) = BUSY |
| `4:0` | **Command** | `0x01`–`0x05` 为标准命令；`0x06`–`0x1F` 为厂商命令 |

### 6.2 命令交互与适配器实测应答

| 命令 | 请求字（TX） | 适配器应答（RX） | 作用说明 |
|---|---|---|---|
| `Discover Identity` (`0x01`) | `0xFF00A001` | ACK `0xFF00A841, 0x05E0413C, ...` | 确认适配器 VID `0x413C` |
| `Discover SVIDs` (`0x02`) | `0xFF00A002` | ACK `0xFF00A842, 0x413C0000` | 返回私有 SVID `0x413C` |
| `Discover Modes` (`0x03`) | `0x413CA003` | ACK `0x413CA843, 0x00000002` | 返回 Mode 1 VDO `0x00000002` |
| `Enter Mode` pos 1 (`0x04`) | `0x413CA104` | ACK `0x413CA944` | **解锁 130W SPR 与 280W EPR 功率上限** |
| `Enter Mode` pos 2 (`0x04`) | `0x413CA204` | NAK `0x413CAA84` | 不存在第二个模式 |
| `Exit Mode` pos 1 (`0x05`) | `0x413CA105` | ACK `0x413CA945` | 退出私有模式（`PDO4` 从 `6.50A` 退回 `4.80A`） |
| `Attention` (`0x06`) | `0x413CA106` | 无应答（静默） | — |
| 私有命令 `0x07–0x1F`（含 `0x12`） | `0x413CA112` / `0x413C8112` | NAK `0x413CA887..0x413CA89F`（`0x413CA892`） | 不改变任何供电能力 |

---

## 7. `Source_Capabilities_Extended`（SCEDB，25 字节）

`Get_Source_Cap_Extended`（`Control 0x11`）的应答为 PD Extended 报文（`ext=1, type=0x01`），由 2 字节扩展头 `0x8019`（`Data Size = 25`）与 25 字节 SCEDB 构成。

| SCEDB 字节偏移 | 字段 | 未解锁态 | 解锁态 |
|---|---|---|---|
| `0..1` | VID | `0x413C` | `0x413C` |
| `2..3` | PID | `0x0000` | `0x0000` |
| `4..7` | XID | `0x00000000` | `0x00000000` |
| `8..13` | FW / HW / 调压 / 保持 / 合规 / 接触电流 | 全 `0x00` | 全 `0x00` |
| `14..19` | Peak Current 1–3 | 全 `0x00` | `88 2F A8 2E D0 2E` |
| `20` | Touch Temp | `0x00` | `0x00` |
| `21` | Source Inputs | `0x03` | `0x03` |
| `22` | 电池数量 | `0x00` | `0x00` |
| **`23`** | **SPR Source PDP (W)** | **`0x64`（100W）** | **`0x6A`（106W = `20V × 5.30A`）** |
| **`24`** | **EPR Source PDP (W)** | **`0xF0`（240W）** | **`0xF0`（240W）** |

---

## 8. Extended Control 与分片请求编码

| 报文 | PD Header 配置 | 32-bit 对象 | 字段拆解 |
|---|---|---|---|
| **Chunk Request #1** | `ext=1, count=1, type=0x11, data_role=1` | `0x00008C00` | 低 16 位 = `0x8C00`（`Chunked=1`、`ChunkNum=1`、`RequestChunk=1`、`DataSize=0`）；高 16 位 = `0x0000` |
| **`EPR_KeepAlive`（TX）** | `ext=1, count=1, type=0x10, data_role=1` | `0x00038002` | 低 16 位 = `0x8002`（`Chunked=1`、`ChunkNum=0`、`RequestChunk=0`、`DataSize=2`）；Byte 2 = `0x03`（`EPR_KeepAlive`）；Byte 3 = `0x00` |
| **`EPR_KeepAlive_Ack`（RX）** | `ext=1, count=1, type=0x10, power_role=1` | `0x00048002` | 低 16 位 = `0x8002`（`DataSize=2`）；Byte 2 = `0x04`（`EPR_KeepAlive_Ack`）；Byte 3 = `0x00` |

---

## 9. 实测时序参考表

| 阶段 / 事件间隔 | 实测典型 / 要求时间 | 说明 |
|---|---|---|
| CC 软断开复位持续时间 | `600ms – 1500ms` | 需等 `VBUS` 跌落归零后再重挂 `Rd` |
| 收到 `Source_Capabilities` → 发送 `Request` | `~5ms`（须 `< 24ms`） | 受 `tSenderResponse` 限制（含初始、130W 及隐藏 PPS 重播） |
| `Request` → `Accept` | `~5ms` | 适配器正常确认延迟 |
| `Accept` → `PS_RDY`（SPR） | `~310ms` | `20V` 调压建立时间 |
| `Accept` → `PS_RDY`（EPR 高压） | `~710ms` | `28V / 36V / 48V` 升压建立时间（过渡期内暂停 KeepAlive） |
| 初始 `PS_RDY` → `DR_Swap` | `~6ms` | 在开始 VDM 解锁前执行 |
| `DR_Swap` `Accept` → `Discover Identity` | `~40ms` | 须先完成本地 `data_role=1` 切换 |
| `Discover Identity` ACK → `Discover SVIDs` | `~295ms` | 与真实设备抓包静默窗一致 |
| `Discover SVIDs` ACK → `Discover Modes` | `~6ms` | — |
| `Discover Modes` ACK → `Get_Source_Cap_Extended` | `~11ms` | — |
| `SCEDB` → `Enter Mode` | `~31ms` | — |
| `Enter Mode` ACK → `Get_Source_Cap` | `~35ms` | — |
| `EPR_KeepAlive` 发送周期 | **`75ms`** | 超时约 `1s` 适配器即退出 EPR 回落 `5V` |
| SPR PPS 续约周期（`tPPSRequest`） | **`8000ms`** | 须 `< 10s`（USB-PD 规范上限） |
