# Dell 280W (`LA280PM240`) Packet & Bitfield Reference

[Home](../../../README.md) / [Dell LA280PM240](README.md) | [English](packet-reference.md) · [简体中文](packet-reference_CN.md)

Bit-level wire reference for every USB-PD message, PDO, RDO, SVDM, SCEDB, SKEDB, and Extended Control frame observed or required in the **Dell 280W (`LA280PM240`)** negotiation flow.

---

## 1. 16-Bit USB-PD Message Header

USB-PD headers are 16-bit little-endian fields transmitted before the data objects (`N = Count` 32-bit words).

| Bits | Field | Values Used in This Protocol |
|---|---|---|
| `15` | **Extended** | `0` = Control / Data message; `1` = Extended message (`SCEDB`, `SKEDB`, `EPR_Source_Capabilities`, `Extended_Control`) |
| `14:12` | **Number of Data Objects** | `0` = Control message; `1..7` = Data / Extended objects (`0` in extended chunk requests uses `1` object containing the 2-byte ext header) |
| `11:9` | **MessageID** | `0..7` rolling counter per sender, incremented upon GoodCRC completion |
| **`8`** | **Port Power Role** | `0` = Sink (always `0` for Sink TX); `1` = Source (Adapter TX) |
| `7:6` | **Spec Revision** | `2` (`0b10`) = USB-PD Revision 3.0 / 3.1 / 3.2 |
| **`5`** | **Port Data Role** | `0` = UFP (initial attach through `DR_Swap`), **`1` = DFP (mandatory for all TX after `DR_Swap` Accept)** |
| `4:0` | **Message Type** | See Message Type lookup table below |

> [!IMPORTANT]
> **Common Header Bitfield Pitfall**: `bit 8` is **Port Power Role** and `bit 5` is **Port Data Role** (not the reverse). Also, **never mask off `bit 15` (Extended)** before decoding a packet, or 25-byte `SCEDB` payloads will be misdecoded as bogus PDOs.

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

## 2. Message Types Used

Message type IDs (`header & 0x1F`) are disambiguated by `Extended` (`bit 15`) and `Count` (`bits 14:12`):

| Type ID | Control (`ext=0, count=0`) | Data (`ext=0, count>0`) | Extended (`ext=1`) |
|---|---|---|---|
| `0x01` | `GoodCRC` | `Source_Capabilities` | `Source_Capabilities_Extended` (`SCEDB`) |
| `0x03` | `Accept` | — | — |
| `0x04` | `Reject` | `Sink_Capabilities` | — |
| `0x06` | `PS_RDY` | — | — |
| `0x07` | `Get_Source_Cap` | — | — |
| `0x08` | `Get_Sink_Cap` | — | — |
| `0x09` | `DR_Swap` | `EPR_Request` (`count=2`) | — |
| `0x0A` | `PR_Swap` | `EPR_Mode` (`count=1`) | — |
| `0x0B` | `VCONN_Swap` | — | — |
| `0x0C` | `Wait` | — | — |
| `0x0D` | `Soft_Reset` | — | — |
| `0x0F` | — | `Vendor_Defined` (`VDM`) | `Sink_Capabilities_Extended` (`SKEDB`, `count=7`) |
| `0x10` | `Not_Supported` | — | `Extended_Control` (`EPR_KeepAlive` / `Ack`) |
| `0x11` | `Get_Source_Cap_Extended` | — | `EPR_Source_Capabilities` (chunked 0 & 1) |
| `0x16` | `Get_Sink_Cap_Extended` | — | — |

---

## 3. Source PDO Encoding & Observed Values

### 3.1 Fixed Supply PDO (`bits 31:30 = 0b00`)

| Bits | Field | Unit / Meaning |
|---|---|---|
| `31:30` | Supply Type | `0b00` = Fixed Supply |
| `27` | Unconstrained Power | `1` on `PDO1` (`0x0A81912C`) |
| `25` | Dual-Role Data | `1` on `PDO1` (`0x0A81912C`) |
| `23` | EPR Mode Capable | `1` on `PDO1` (`0x0A81912C`) |
| `19:10` | Voltage | `50 mV` units (`mv / 50`) |
| `9:0` | Max Current | `10 mA` units (`ma / 10`) |

### 3.2 SPR Programmable Power Supply (PPS) APDO (`bits 31:28 = 0xC`)

| Bits | Field | Unit / Meaning |
|---|---|---|
| `31:28` | Type / Subtype | `0b1100` (`0xC`) = SPR PPS APDO |
| `27` | PPS Power Limited | `0` = Current-limited (`0xC190786A` has bit 27 = `0`) |
| `24:17` | Max Voltage | `100 mV` units (`0xC8` = `200` → `20.0V`) |
| `15:8` | Min Voltage | `100 mV` units (`0x78` = `120` → `12.0V`) |
| `6:0` | Max Current | `50 mA` units (`0x6A` = `106` → `5.30A`) |

### 3.3 EPR Adjustable Voltage Supply (AVS) APDO (`bits 31:28 = 0xD`)

| Bits | Field | Unit / Meaning |
|---|---|---|
| `31:28` | Type / Subtype | `0b1101` (`0xD`) = EPR AVS APDO |
| `25:17` | Max Voltage | `100 mV` units (`0x1E0` = `480` → `48.0V`) |
| `15:8` | Min Voltage | `100 mV` units (`0x96` = `150` → `15.0V`) |
| `7:0` | PDP | `1 W` units (`0xF0` = `240W`, `0xFF` = `255W`) |

### 3.4 Complete Adapter PDO Table

| Context | Slot | Raw Hex | Decoded Capability |
|---|---|---|---|
| **SPR (Initial / Locked)** | PDO 1 | `0x0A81912C` | Fixed `5.00V / 3.00A` (`15.00W`) |
| | PDO 2 | `0x0002D12C` | Fixed `9.00V / 3.00A` (`27.00W`) |
| | PDO 3 | `0x0004B12C` | Fixed `15.00V / 3.00A` (`45.00W`) |
| | PDO 4 | `0x000641E0` | Fixed `20.00V / 4.80A` (`96.00W`) |
| **SPR (Unlocked `0x413C` Mode 1)** | PDO 4 | `0x0006428A` | **Fixed `20.00V / 6.50A` (`130.00W`)** |
| **SPR (+ Sink AVS Triggered)** | PDO 5 | `0xC190786A` | **SPR PPS `12.00–20.00V / 5.30A` (`106.00W`)** |
| **EPR (Locked Standard PD)** | PDO 11 | `0xD3C096F0` | EPR AVS `15.0–48.0V`, PDP `240W` |
| **EPR (Unlocked `0x413C` Mode 1)** | PDO 1–3 | `0x0A81912C`, `0x0002D12C`, `0x0004B12C` | Fixed `5V/3A`, `9V/3A`, `15V/3A` |
| | PDO 4 | `0x000641F4` | Fixed `20.00V / 5.00A` (`100.00W`) |
| | PDO 5–7 | `0x00000000` | Empty SPR slots |
| | PDO 8 | `0x0008C24D` | **Fixed `28.00V / 5.89A` (`164.92W`)** |
| | PDO 9 | `0x000B4247` | **Fixed `36.00V / 5.83A` (`209.88W`)** |
| | PDO 10 | `0x000F0247` | **Fixed `48.00V / 5.83A` (`279.84W`)** |
| | PDO 11 | `0xD3C096FF` | **EPR AVS `15.0–48.0V`, PDP `255W`** |

---

## 4. Sink Capabilities & SKEDB (Hidden PPS Trigger)

### 4.1 `Sink_Capabilities` (`Data type=0x04, count=5`)

| Slot | Raw Hex | Decoded Sink PDO |
|---|---|---|
| PDO 1 | `0x0001912C` | Fixed `5.00V / 3.00A` |
| PDO 2 | `0x0002D12C` | Fixed `9.00V / 3.00A` |
| PDO 3 | `0x0004B12C` | Fixed `15.00V / 3.00A` |
| PDO 4 | `0x0006412C` | Fixed `20.00V / 3.00A` |
| **PDO 5** | **`0xE004B12C`** | **PD 3.2 `SPR AVS Sink APDO`**: `bits 31:28 = 0xE` (`1110b`), `bits 19:10 = 300` (`9–15V @ 3.00A`), `bits 9:0 = 300` (`15–20V @ 3.00A`) |

### 4.2 `Sink_Capabilities_Extended` (`Extended ext=1, type=0x0F, count=7`)

Transmitted as 7 little-endian 32-bit objects (`2` bytes extended header + `24` bytes SKEDB + `2` bytes zero pad = `28` bytes):

```text
Word 0: 0x00000018   Word 1: 0x00000000   Word 2: 0x00000000   Word 3: 0x00000001
Word 4: 0x20000000   Word 5: 0x003C2805   Word 6: 0x00000000
```

| Byte Offset (in 24B SKEDB) | Field | Hex / Value | Purpose |
|---|---|---|---|
| `0..1` | VID | `0x0000` | Sink Vendor ID placeholder (fill as needed) |
| `2..3` | PID | `0x0000` | Sink Product ID placeholder (fill as needed) |
| `4..7` | XID | `0x00000000` | USB-IF XID |
| `8..9` | FW / HW Rev | `0x00 / 0x00` | Version bytes |
| `10` | SKEDB Version | `0x01` | SKEDB v1 |
| `11..16` | Load / Compliance / Battery | `0x00` | Default zeros |
| **`17`** | **Sink Modes** | **`0x20`** | **Bit 5 (`0x20`) = AVS Support (critical trigger)** |
| `18..20` | SPR Min / Op / Max PDP | `0x05 / 0x28 / 0x3C` | `5W / 40W / 60W` |
| `21..23` | EPR Min / Op / Max PDP | `0x00 / 0x00 / 0x00` | `0W` (SPR sink) |

*(Preceded on the wire by 2-byte unchunked extended header `0x0018` = `Chunked=0, Data Size=24`.)*

---

## 5. Request Data Objects (RDO)

### 5.1 RDO Flag Bits (`bits 27:22`)

| Bit | Mask | Name | Usage in Flow |
|---|---|---|---|
| `25` | `0x02000000` | USB Communications Capable | `1` |
| `24` | `0x01000000` | No USB Suspend | `1` |
| `23` | `0x00800000` | Unchunked Extended Messages Supported | `1` in SPR & EPR AVS (`0x03800000` / `0x03C00000`); `0` in Fixed EPR (`0x03400000`) |
| `22` | `0x00400000` | EPR Mode Capable | `0` for SPR-only (`0x03800000`), `1` when entering or operating in EPR (`0x03C00000` / `0x03400000`) |

### 5.2 Formulas & Verified RDO Vectors

1. **Fixed RDO** (`Operating Current` at `bits 19:10`, `Max Current` at `bits 9:0`, `10 mA` units):
   `RDO = (pos << 28) | flags | ((I_mA / 10) << 10) | (I_mA / 10)`
2. **SPR PPS RDO** (`Output Voltage` at `bits 20:9` in `20 mV` units, `Operating Current` at `bits 6:0` in `50 mA` units):
   `RDO = (pos << 28) | 0x03800000 | ((V_mV / 20) << 9) | (I_mA / 50)`
3. **EPR AVS RDO** (`Output Voltage` at `bits 20:9` in `25 mV` units, `Operating Current` at `bits 6:0` in `50 mA` units):
   `RDO = (pos << 28) | 0x03C00000 | ((V_mV / 25) << 9) | (I_mA / 50)`

| Message | Target | RDO Hex | Object 2 (EPR Copy) |
|---|---|---|---|
| `Request` (`0x02`, 1 obj) | Initial `20.00V / 4.50A` (pos 4, SPR) | `0x438709C2` | — |
| `Request` (`0x02`, 1 obj) | Unlocked `20.00V / 6.50A` (pos 4, SPR) | `0x438A2A8A` | — |
| `Request` (`0x02`, 1 obj) | Unlocked `20.00V / 6.50A` (pos 4, EPR Cap=1) | `0x43CA2A8A` | — |
| `Request` (`0x02`, 1 obj) | Safe `5.00V / 3.00A` (pos 1, SPR) | `0x1384B12C` | — |
| `Request` (`0x02`, 1 obj) | PPS `12.00V / 5.30A` (pos 5) | `0x5384B06A` | — |
| `Request` (`0x02`, 1 obj) | PPS `12.00V / 3.00A` (pos 5) | `0x5384B03C` | — |
| `Request` (`0x02`, 1 obj) | PPS `16.00V / 5.00A` (pos 5) | `0x53864064` | — |
| `EPR_Request` (`0x09`, 2 obj) | Safe `5.00V / 3.00A` (pos 1, EPR) | `0x1344B12C` | `0x0A81912C` |
| `EPR_Request` (`0x09`, 2 obj) | Fixed `28.00V / 5.89A` (pos 8, EPR) | `0x8349364D` | `0x0008C24D` |
| `EPR_Request` (`0x09`, 2 obj) | Fixed `36.00V / 5.83A` (pos 9, EPR) | `0x93491E47` | `0x000B4247` |
| `EPR_Request` (`0x09`, 2 obj) | Fixed `48.00V / 5.83A` (pos 10, EPR) | `0xA3491E47` | `0x000F0247` |
| `EPR_Request` (`0x09`, 2 obj) | AVS `15.00V / 5.00A` (pos 11, EPR) | `0xB3C4B064` | `0xD3C096FF` |
| `EPR_Request` (`0x09`, 2 obj) | AVS `20.00V / 5.00A` (pos 11, EPR) | `0xB3C64064` | `0xD3C096FF` |
| `EPR_Request` (`0x09`, 2 obj) | AVS `48.00V / 5.00A` (pos 11, EPR) | `0xB3CF0064` | `0xD3C096FF` |

---

## 6. SVDM Header & Dell `0x413C` Commands

### 6.1 Structured VDM Header (`32-bit`)

| Bits | Field | Notes |
|---|---|---|
| `31:16` | **SVID / VID** | `0xFF00` (Standard PD Discovery) or `0x413C` (Dell) |
| `15` | **VDM Type** | `1` = Structured VDM |
| `14:13` | **VDM Version (Major)** | `01b` = SVDM 2.0 (`0x2000`); combined with bit 15 gives `0xA000` |
| `12:11` | **VDM Version (Minor)** | TX uses `00b`; adapter replies carry `bit 11 = 1` (`0xA8xx` / `0xA9xx`) |
| `10:8` | **Object Position** | `0` for discovery; `1` for Dell Mode 1 |
| `7:6` | **Command Type** | `0` (`00b`) = REQ, `1` (`01b`) = ACK, `2` (`10b`) = NAK, `3` (`11b`) = BUSY |
| `4:0` | **Command** | `0x01`–`0x05` standard; `0x06`–`0x1F` vendor-defined |

### 6.2 Observed Commands & Adapter Responses

| Command | TX Word | Adapter RX | Effect |
|---|---|---|---|
| `Discover Identity` (`0x01`) | `0xFF00A001` | ACK `0xFF00A841, 0x05E0413C, ...` | Identifies Dell VID `0x413C` |
| `Discover SVIDs` (`0x02`) | `0xFF00A002` | ACK `0xFF00A842, 0x413C0000` | Reports Dell SVID `0x413C` |
| `Discover Modes` (`0x03`) | `0x413CA003` | ACK `0x413CA843, 0x00000002` | Reports Mode 1 VDO `0x00000002` |
| `Enter Mode` pos 1 (`0x04`) | `0x413CA104` | ACK `0x413CA944` | **Unlocks 130W SPR & 280W EPR limits** |
| `Enter Mode` pos 2 (`0x04`) | `0x413CA204` | NAK `0x413CAA84` | No second mode exists |
| `Exit Mode` pos 1 (`0x05`) | `0x413CA105` | ACK `0x413CA945` | Reverts `PDO4` from `6.50A` back to `4.80A` |
| `Attention` (`0x06`) | `0x413CA106` | Silence (no response) | — |
| Vendor `0x07–0x1F` (incl. `0x12`) | `0x413CA112` / `0x413C8112` | NAK `0x413CA887..0x413CA89F` (`0x413CA892`) | Does not modify capabilities |

---

## 7. `Source_Capabilities_Extended` (SCEDB, 25 Bytes)

Response to `Get_Source_Cap_Extended` (`Control 0x11`): PD Extended message (`ext=1, type=0x01`) starting with unchunked 2-byte extended header `0x8019` (`Chunked=0` or single-block `Data Size = 25` bytes; `0x8019` has `Data Size = 25` and `bit 15 = 1` chunked header format).

| SCEDB Byte Offset | Field | Locked Value | Unlocked Value |
|---|---|---|---|
| `0..1` | VID | `0x413C` | `0x413C` |
| `2..3` | PID | `0x0000` | `0x0000` |
| `4..7` | XID | `0x00000000` | `0x00000000` |
| `8..13` | FW / HW / Reg / Hold / Compliance / Touch | `0x00` | `0x00` |
| `14..19` | Peak Current 1–3 | All `0x00` | `88 2F A8 2E D0 2E` |
| `20` | Touch Temp | `0x00` | `0x00` |
| `21` | Source Inputs | `0x03` | `0x03` |
| `22` | Number of Batteries | `0x00` | `0x00` |
| **`23`** | **SPR Source PDP (W)** | **`0x64` (100W)** | **`0x6A` (106W = `20V × 5.30A`)** |
| **`24`** | **EPR Source PDP (W)** | **`0xF0` (240W)** | **`0xF0` (240W)** |

---

## 8. Extended Control & Chunk Request Objects

| Frame | PD Header Bits | 32-Bit Object | Field Breakdown |
|---|---|---|---|
| **Chunk Request #1** | `ext=1, count=1, type=0x11, data_role=1` | `0x00008C00` | Low 16 bits = `0x8C00` (`Chunked=1`, `ChunkNum=1`, `RequestChunk=1`, `DataSize=0`); high 16 bits = `0x0000` |
| **`EPR_KeepAlive` (TX)** | `ext=1, count=1, type=0x10, data_role=1` | `0x00038002` | Low 16 bits = `0x8002` (`Chunked=1`, `ChunkNum=0`, `RequestChunk=0`, `DataSize=2`); Byte 2 = `0x03` (`EPR_KeepAlive`); Byte 3 = `0x00` |
| **`EPR_KeepAlive_Ack` (RX)** | `ext=1, count=1, type=0x10, power_role=1` | `0x00048002` | Low 16 bits = `0x8002` (`DataSize=2`); Byte 2 = `0x04` (`EPR_KeepAlive_Ack`); Byte 3 = `0x00` |

---

## 9. Timing Reference

| Transition / Window | Typical / Required Timing | Notes |
|---|---|---|
| CC Detach reset duration | `600ms – 1500ms` | Wait until `VBUS` reaches `0V` before re-attaching `Rd` |
| `Source_Capabilities` → `Request` | `~5ms` (must be `< 24ms`) | `tSenderResponse` deadline; applies to initial, 130W, and post-PPS caps |
| `Request` → `Accept` | `~5ms` | Typical adapter response latency |
| `Accept` → `PS_RDY` (SPR) | `~310ms` | `20V` step-up / contract settle |
| `Accept` → `PS_RDY` (EPR HV) | `~710ms` | High-voltage transition (`28V / 36V / 48V`); pause KeepAlive during transition |
| Initial `PS_RDY` → `DR_Swap` | `~6ms` | Before VDM discovery |
| `DR_Swap` `Accept` → `Discover Identity` | `~40ms` | Switch `data_role=1` first |
| `Discover Identity` ACK → `Discover SVIDs` | `~295ms` | Matches captured workstation quiet interval |
| `Discover SVIDs` ACK → `Discover Modes` | `~6ms` | — |
| `Discover Modes` ACK → `Get_Source_Cap_Extended` | `~11ms` | — |
| `SCEDB` → `Enter Mode` | `~31ms` | — |
| `Enter Mode` ACK → `Get_Source_Cap` | `~35ms` | — |
| `EPR_KeepAlive` interval | **`75ms`** | Adapter drops EPR back to `5V` if silent for `~1s` |
| SPR PPS renewal interval (`tPPSRequest`) | **`8000ms`** | Must be `< 10s` per USB-PD spec |
