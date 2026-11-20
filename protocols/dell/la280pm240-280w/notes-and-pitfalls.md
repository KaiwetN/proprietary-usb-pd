# Empirical Notes, Pitfalls & Negative Findings

[Home](../../../README.md) / [Dell LA280PM240](README.md) | [English](notes-and-pitfalls.md) · [简体中文](notes-and-pitfalls_CN.md)

This document summarizes critical implementation pitfalls, adapter firmware quirks, exhaustive negative test results, and hardware safety rules verified on the **Dell 280W (`LA280PM240`)** adapter.

---

## 1. `DR_Swap` to DFP Is Mandatory Before Dell SVDM

- At initial attach, a USB-PD Sink is `UFP` (`Port Data Role`, header `bit 5 = 0`).
- Sending Dell discovery/unlock SVDMs while still `UFP` fails to unlock the adapter.
- You **must**:
  1. Negotiate an initial `20V / 4.50A` contract (`0x438709C2`) and wait for `PS_RDY`.
  2. Issue `DR_Swap` (Control `0x09`) with `data_role = 0`.
  3. After receiving `Accept` (`0x03`), update **every subsequent TX header** to set `bit 5 = 1` (`DFP`), while keeping `bit 8 = 0` (`Sink`).

---

## 2. `EPR_KeepAlive` Must Be `0x00038002`, Not `0x00000000`

- The `Extended_Control` (`ext=1, count=1, type=0x10`) payload for `EPR_KeepAlive` must carry a valid 2-byte extended header (`0x8002`: `Chunked=1, Data Size=2`) plus ECDB type `0x03` (`EPR_KeepAlive`): **`0x00038002`**.
- **Pitfall**: Sending `0x00000000` (`Data Size=0`, no ECDB) causes the `LA280PM240` to **ignore the frame completely—not even returning `GoodCRC`**. Without valid KeepAlive acknowledgments (`0x00048002`), the adapter exits EPR and drops `VBUS` back to `5V` within ~`1.0–1.6s`.
- **Recommendation**: Send `0x00038002` every **`75ms`** during steady EPR contracts, and pause KeepAlive transmissions briefly while an `EPR_Request -> Accept -> PS_RDY` voltage transition (`~710ms`) is in progress.

---

## 3. `EPR_Request` Object 2 Must Match the Advertised PDO Exactly

- Every `EPR_Request` (`Data type=0x09, count=2`) carries `[RDO, PDO_Copy]`.
- The adapter performs a strict equality check on `PDO_Copy` against its advertised PDO at the requested position:
  - Requesting a modified/spoofed APDO copy (for example `0xD3C078FF` claiming a `12V` AVS floor, `0xC190786A`, or `0x00000000`) does **not** return `Reject`—the adapter goes completely silent on that request.
- Always copy the exact 32-bit word received from `EPR_Source_Capabilities` (`0x0A81912C`, `0x0008C24D`, `0x000B4247`, `0x000F0247`, `0xD3C096FF`).

---

## 4. EPR AVS Has a Hard `15.00V` Floor and `5.00A` Ceiling

- **Strict `15.00V` Floor**: Verified at `25mV` resolution both at initial EPR entry and during active AVS contracts:
  - `15.00V` (`0xB3C4B064` with `0xD3C096FF`) → `Accept` + `PS_RDY`.
  - `14.90V`, `14.75V`, and `12.00V` → **`Reject` (`0x04`)**, regardless of reserved RDO flag bits.
- **Strict `5.00A` Ceiling**: Even though Dell's unlocked Fixed EPR PDOs advertise `5.89A` (28V) and `5.83A` (36V/48V), USB-PD 3.1 EPR AVS limits operating current to:
  $$I_{\text{max}} \le \min\left(5.00\text{ A},\; \left\lfloor \frac{\text{PDP}_W}{V} \right\rfloor_{50\text{mA}}\right)$$
  Requesting `> 5.00A` in an AVS RDO is immediately rejected.
- **How to get `12.00–14.98V`**: Use **Branch A (Hidden SPR PPS `12.00–20.00V / 5.30A`, `0xC190786A`)**, triggered by replying to `Get_Sink_Cap` with `0xE004B12C` and `Get_Sink_Cap_Extended` with `Sink Modes = 0x20`.

---

## 5. Negative Findings from Full Dell SVDM Command Sweeps (`0x00–0x1F`)

Exhaustive sweeps across the 5-bit Structured VDM command space (`0x00–0x1F`, both bare and multi-object payloads, SVDM v1.0 and v2.0, in both 130W SPR and active EPR states) confirmed:

- **Only standard commands `0x03`, `0x04` (pos 1), and `0x05` (pos 1) are ACKed** for `SVID 0x413C`:
  - `0x03` (`Discover Modes`): ACK `0x413CA843, 0x00000002`
  - `0x04` (`Enter Mode`, `pos=1`): ACK `0x413CA944` — unlocks 130W SPR / 280W EPR
  - `0x04` (`Enter Mode`, `pos=2`): **NAK** `0x413CAA84` — `0x00000002` is not a second mode
  - `0x05` (`Exit Mode`, `pos=1`): ACK `0x413CA945` — **reverts `PDO4` from `6.50A` (`0x0006428A`) back to `4.80A` (`0x000641E0`)**
- **`0x06` (`Attention`)**: Always ignored (silence).
- **Vendor commands `0x07–0x1F` (including `0x12`)**: Always **NAK** (`0x413CA887..0x413CA89F`, e.g. `0x413CA892` for `cmd 0x12`). Real Dell workstations also send `0x413C8112` and receive `0x413CA892` NAK; replaying `0x12` is not required for unlocking.

---

## 6. Adapter PD Silence State & Clean Recovery

- If an initial `Source_Capabilities` exchange stalls, a `tSenderResponse` (`24ms`) window is missed, or repeated invalid frames occur, the `LA280PM240` latches into **PD silence** (it stops transmitting `Source_Capabilities` and stops responding to `Get_Source_Cap` or `Soft_Reset`).
- **Recovery**:
  1. Remove CC `Rd` pull-down completely for **`600ms–1500ms`** until `VBUS` drops to `0V`, then re-attach single `Rd`.
  2. After repeated consecutive EPR faults, the adapter's internal protection may require a longer **`10–25s` cooldown** while detached before PD negotiation resumes normally.

---

## 7. Do Not Mask Header `bit 15` When Decoding Packets

- Header `bit 15` is the standard USB-PD **Extended Message** flag, not a proprietary bit.
- When `Get_Source_Cap_Extended` (`0x11`) returns a 25-byte `SCEDB` prefixed by `0x8019`, clearing `bit 15` causes naive decoders to split those bytes into 6–7 fake PDOs (e.g., falsely decoding bytes containing `0x6A` and `0x78` as a `12–20V 5.30A` PPS entry from `SCEDB`, even when no `Source_Capabilities` PPS APDO was actually sent on the wire).
- Always check `(header >> 15) & 1` first and skip the 2-byte extended header before parsing extended payloads.

---

## 8. Passive Protocol Hygiene

- **Always respond promptly to `Get_Sink_Cap` (`0x08`) and `Get_Sink_Cap_Extended` (`0x16`)**: Failing to answer adapter queries can stall or silence the session.
- **Reject unsolicited role swaps**: Respond with `Reject` (`Control 0x04`) to unexpected `DR_Swap` (`0x09`), `PR_Swap` (`0x0A`), or `VCONN_Swap` (`0x0B`) requests from the source during active negotiation.
- **Handle `Soft_Reset` (`0x0D`) conservatively**: Do not blindly re-request high voltage after a `Soft_Reset`; invalidate the current contract state and perform a clean CC detach recovery.

---

## 9. High-Voltage Safety Guidelines

1. **Disconnect unprotected loads** before testing `> 20V` (`28V / 36V / 48V` or `AVS`).
2. **Enter EPR at `5V` first**: After `EPR_Source_Capabilities` reassembly, issue an initial safe `5V` `EPR_Request` (`[0x1344B12C, 0x0A81912C]`) before letting the user/application explicitly select `28V`, `36V`, `48V`, or `AVS`.
3. **Always step back to `5V` before stopping KeepAlive or disconnecting**, preventing hot-unplug arcing or leaving an unexpected contract active.
