# Apple Neural Engine (ANE) Architecture & Driver Specification (H18)

Open clean-room architectural specification and hardware execution model for the Apple Neural Engine (Generation H18 / Apollo v5) on modern Apple Silicon.

---

## Overview

The Apple Neural Engine (ANE) is a dedicated hardware accelerator designed for tensor operations and deep learning inference. While traditionally accessed exclusively through CoreML, this repository provides clean-room documentation of the underlying hardware architecture, memory tile geometry, task container format, and hardware command interface to facilitate independent, bare-metal runtime development (e.g. for `llama.cpp` or custom ML engines).

```
+-----------------------------------------------------------------------------+
|                      ANE HIGH-LEVEL SYSTEM TOPOLOGY                         |
+-----------------------------------------------------------------------------+
|                                                                             |
|  [Userland Application / ML Runtime (e.g. llama.cpp / custom C runtime)]    |
|         │                                                                   |
|         ▼ Direct IOKit / UserClient Dispatch                                |
|  [Kernel Driver (ANE Interface & HAL)]                                      |
|         │                                                                   |
|         ▼ FIFO Ring Submission (Node ID 0x40, 16-byte Granularity)          |
|  ============================== HARDWARE BOUNDARY ========================  |
|         │                                                                   |
|         ▼                                                                   |
|  [Hardware Coprocessor Scheduler (Apollo v5)]                               |
|         │                                                                   |
|         ├───► [Cluster 0: fANE0] ── 16 Cores (Matrix Array + Vector ALU)    |
|         ├───► [Cluster 1: fANE1] ── 16 Cores (Matrix Array + Vector ALU)    |
|         └───► [Cluster 2: fANE2] ── 16 Cores (Matrix Array + Vector ALU)    |
|                     ▲                     ▲                     ▲           |
|                     └─────────────────────┼─────────────────────┘           |
|                                           ▼                                 |
|             [16 MB Shared SRAM Scratchpad (Cross-Cluster DMA)]              |
|                                           ▲                                 |
|                                           ▼                                 |
|             [DART IOMMU (16 KB Pages, Direct Physical Addressing)]          |
|                                                                             |
+-----------------------------------------------------------------------------+
```

---

## Key Hardware Invariants

| Parameter | Value | Description |
|:---|:---:|:---|
| **SoC Architecture** | `t8150` | Apple Silicon A19 Pro |
| **ANE Generation** | `H18` | Apollo v5 microcode scheduler |
| **Physical Clusters** | `3` | Tri-cluster array (`fANE0`, `fANE1`, `fANE2`) |
| **Total Neural Cores** | `48` | 16 cores per physical cluster |
| **Memory Tile Size** | `16 KB` (`0x4000`) | Native hardware memory allocation unit |
| **Tile Shift** | `14` | $\log_2(16384)$ |
| **Vector Processing Width** | `32` channels | Systolic Processing Element channel slice |
| **Row Alignment** | `64` bytes | Hardware DMA burst stride constraint |
| **Command Mailbox** | `Node ID 0x40` | Hardware FIFO submission node |
| **Command Granularity**| `16` bytes (`0x10`) | Command packet descriptor alignment |
| **Hardware Ring Buffer**| `32` slots | 64 bytes per descriptor slot (2,048 bytes total) |

---

## 16 KB Memory Tile Geometry

The ANE does not stream linear memory arrays; all tensor operations are addressed in **16 KB memory tiles** arranged along 32-channel raster planes:

```
+-----------------------------------------------------------------------------+
|                      16 KB TILE SPATIAL GEOMETRY                            |
+-----------------------------------------------------------------------------+
|                                                                             |
|  For tensor [N, C, H, W] with data type size B = sizeof(dtype):             |
|                                                                             |
|  1. Aligned Row Stride (R):                                                 |
|     R = ceil((W * B) / 64) * 64                                             |
|                                                                             |
|  2. Channel Plane Pitch (P):                                                |
|     PlaneSize = H * R                                                       |
|     P = ceil(PlaneSize / 16384) * 16384                                     |
|                                                                             |
|  3. Channel Slicing:                                                        |
|     ChannelGroups = ceil(C / 32)                                            |
|                                                                             |
|  4. Total Buffer Size:                                                      |
|     TotalBytes = N * ChannelGroups * 32 * P                                 |
|                                                                             |
+-----------------------------------------------------------------------------+
```

---

## Repository Structure

```
apple-neural-engine-h18-re/
├── README.md                      # Architectural summary & system overview
├── LICENSE                        # MIT License
├── specs/                         # Machine-readable specifications (JSON)
│   ├── hardware_dispatch_spec.json  # FIFO mailbox & DMA channel allocations
│   ├── ane_isa_opcode_table.json    # Hardware command opcodes & tile geometry
│   ├── hwx_container_spec.json      # ANEC container header & task descriptors
│   ├── symbol_inventory.json        # Class definitions & jump table selectors
│   ├── client_server_topology.json  # Client-driver communication flow
│   └── cross_validation_matrix.json # Numerical constants verification matrix
├── runtime/                       # Clean-room driver implementation
│   └── ane_direct_runtime.c       # Independent C driver (tile packing, dispatch)
├── tools/                         # Automated test & validation tooling
│   └── verify_ane_constants.py    # Verification script for hardware constants
└── docs/                          # Detailed engineering reports
    ├── 02_HWX_COMPILER.md         # Lowering pipeline & task descriptor layout
    ├── 03_ISA_BYTECODE.md         # Hardware opcodes & memory transformations
    └── 04_DRIVER_DISPATCH.md      # Kernel queue interface & DART integration
```

---

## Running Verification

To validate all hardware constants and structural definitions against the schema:

```bash
python3 tools/verify_ane_constants.py
```

---

## License

This clean-room reverse engineering specification and runtime code are distributed under the **MIT License**. See [LICENSE](LICENSE) for details.
