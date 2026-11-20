# Proprietary USB-PD Protocol Research

[English](README.md) | [简体中文](README_CN.md)

[![License: CC BY 4.0](https://img.shields.io/badge/Docs-CC%20BY%204.0-lightgrey.svg)](LICENSE-CC-BY-4.0)
[![License: MIT](https://img.shields.io/badge/Code-MIT-blue.svg)](LICENSE-MIT)

Hardware-verified, wire-level specifications of proprietary **USB Power Delivery (USB-PD 3.0 / 3.1 / 3.2)** vendor extensions — covering Structured/Unstructured VDM (SVDM/UVDM) unlocks, Sink/Source capability handshakes (`Sink_Capabilities` / `SKEDB` / `SCEDB`), extended high-power profiles, and hidden voltage/current rails.

> [!WARNING]
> **High-Voltage Safety Notice**: Proprietary USB-PD protocols documented here can negotiate high voltages and powers up to **48V / 280W** over USB-C. Always disconnect unprotected downstream loads before testing `> 20V` transitions, always enter high-voltage modes safely (e.g., landing at `5V` first during EPR bring-up), and always negotiate back to `5V` before stopping keepalives or disconnecting.

---

## Protocol Index

| Vendor | Device / Topic | VID / SVID | Summary | Documentation |
|---|---|---|---|---|
| **Dell** | **`LA280PM240` (280W Adapter)** | `0x413C` | `DR_Swap` + SVDM Mode 1 Unlock, 130W SPR (`20V/6.5A`), Hidden `12–20V / 5.30A` PPS (`SPR AVS Sink APDO` + `SKEDB`), 280W EPR (`28/36/48V` + `255W AVS`) | [English](protocols/dell/la280pm240-280w/README.md) / [简体中文](protocols/dell/la280pm240-280w/README_CN.md) |

---

## Repository Structure & Conventions

Each vendor protocol lives under `protocols/<vendor>/<device-or-topic>/`:

```text
protocols/
└── <vendor>/
    └── <device-or-topic>/
        ├── README.md                 # Overview, capability table, and entry point (EN)
        ├── README_CN.md              # Overview, capability table, and entry point (ZH)
        ├── protocol-flow.md          # Step-by-step handshake & sequence diagram (EN)
        ├── protocol-flow_CN.md       # Step-by-step handshake & sequence diagram (ZH)
        ├── packet-reference.md       # Bit-level headers, PDOs, RDOs, VDMs, timings (EN)
        ├── packet-reference_CN.md    # Bit-level headers, PDOs, RDOs, VDMs, timings (ZH)
        ├── notes-and-pitfalls.md     # Empirical quirks, negative tests, safety rules (EN)
        └── notes-and-pitfalls_CN.md  # Empirical quirks, negative tests, safety rules (ZH)
```

### Wire Notation Conventions

- USB-PD transmits 16-bit headers and 32-bit data objects **little-endian** on the wire.
- Throughout this repository, 32-bit words (`PDO`, `RDO`, `VDM`, `Extended_Control`) are shown in standard 32-bit hexadecimal representation (`0xXXXXXXXX`), with bitfield indices following USB-PD convention (`bit 31` MSB down to `bit 0` LSB).
- Multi-byte extended blocks (`SCEDB`, `SKEDB`) also specify 0-based byte offsets in wire transmission order.

---

## License

This repository uses a split license model:

- **Documentation (`*.md`, `protocols/**`)**: Licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE-CC-BY-4.0). You are free to share and adapt this specification, including commercially, provided proper attribution is given.
- **Code Snippets & Formulas**: Licensed under the [MIT License](LICENSE-MIT).

See [LICENSE](LICENSE) for full details.
