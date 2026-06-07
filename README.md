# CNN Accelerator Coprocessor — Graduation Project

> Reconfigurable CNN coprocessor integrated with E203 RISC-V via 
> custom instruction set extension.
> Grade: A | Team Project | Suez Canal University, 2024–2025

This project implements a reconfigurable CNN coprocessor based on the 
RISC-V instruction set architecture, targeting efficient edge AI 
inference on FPGA. The coprocessor supports convolution, pooling, ReLU, 
and matrix addition through four processing elements connected via a 
dynamic crossbar network.

The coprocessor interfaces with the E203 RISC-V core through the NICE 
(Nuclei Instruction Co-unit Extension) interface, using nine custom 
RISC-V instructions defined in the custom-0 opcode space.

---

## My Contributions

This was a team project. My personal contributions covered four 
subsystems:

### 1. Convolution Engine

- **Convolution buffer architecture** — sliding-window line buffer 
  that streams pixels one row at a time, eliminating redundant cache 
  refetching and reducing on-chip memory usage.
- **Sliding-window generator** — shift-register-based architecture 
  supporting 3×3 to 7×7 kernel sizes with window_valid handshaking.
- **Multiplier array** — 25 parallel multipliers with mul_en 
  enable-per-filter-size, performing 25 multiply operations in one 
  clock cycle.
- **Reduction adder tree** — 4-level combinational + 1-level 
  sequential hybrid adder tree, accumulating 25 products in one 
  clock cycle.
- **MAC datapath** — complete multiply-accumulate path completing 
  full convolution in 2 clock cycles per image window.

### 2. Accelerator Controller

- **Reconfigurable controller FSM** — 5-state architecture: IDLE → 
  CONFIG → EXECUTE → WAIT_COMPLETION → WRITE_BACK.
- **9 custom CNN instructions** — cnn.rst, cnn.store.ke, 
  cnn.store.img, cnn.store.res, cnn.load.res, cnn.store.old, 
  cnn.load.old, cnn.wm, cnn.sc — decoded and dispatched to 
  processing elements.
- **Timing control system** — 29 independent counters managing 
  cache timing, PE sequencing, memory access coordination, and 
  multi-layer pipeline transitions.
- **7 data flow paths** — image cache, kernel cache, old data cache, 
  PE data loading, crossbar RAM, result cache, and RISC-V result 
  transfer.
- **PE coordination** — dynamic enable sequencing, completion 
  monitoring (finish_conv, finish_pool, finish_relu, finish_adder), 
  and crossbar reconfiguration per instruction.

### 3. Memory System

- **8-way set-associative cache** — write-back policy with dirty-bit 
  tracking, scaled from the same architecture used in the 
  FloatCore-RV32IF processor. Supports 4 specialized cache banks: 
  image cache, kernel/coefficient cache, old data cache, and result 
  cache.
- **Address generation unit** — base+offset addressing for all four 
  CNN operation types (convolution, pooling, ReLU, addition) with 
  dedicated address counters per operation.
- **Cache interface timing** — 3-cycle hit detection windows, 
  ping-pong buffer coordination, and multi-cache enable/disable 
  sequencing.

### 4. System Integration

- Integrated convolution engine, controller, and cache subsystems 
  with team-developed pooling, ReLU, matrix addition, reconfigurable 
  crossbar, data fetcher, and decoder modules.
- Connected to E203 RISC-V core via NICE interface — handling 
  nice_req_valid/ready, nice_rsp_valid/rdat, and 
  nice_icb_cmd/rsp memory channels.
- Validated end-to-end with LeNet-5 (C1/S2 layers), Sobel edge 
  detection, and FIR filtering workloads.
- Python reference model used for 100% output match verification.

---

## Key Results

| Metric | Result |
|---|---:|
| Convolution cycles | 1,837 |
| Baseline RV32IM cycles | 12,982 |
| Convolution speedup | **7×** |
| Reference paper cycles | 2,070 |
| Improvement over reference paper | **+10%** |
| Pooling speedup | 1.44× |
| ReLU speedup | 1.13× |
| Matrix addition speedup | 1.37× |
| Python reference model match | 100% |
| Synthesis target | Artix-7 (xc7a35t) |
| LUTs (coprocessor only) | 12,872 |
| Registers | 8,959 |
| DSPs | 21 |

---

## System Architecture

```text
E203 RISC-V Core
       │
       │ NICE Interface
       ▼
┌──────────────────────────────────────────┐
│            EAI Controller                │
│  ┌─────────┐   ┌──────────────────────┐  │
│  │ Decoder │──▶│ Reconfigurable       │  │
│  │  (FSM)  │   │ Controller (FSM)     │  │
│  └─────────┘   └──────────────────────┘  │
│                        │                 │
│         ┌──────────────┼──────────┐      │
│         ▼              ▼          ▼      │
│  ┌────────────┐  ┌──────────┐  ┌──────┐ │
│  │ Data       │  │ Crossbar │  │Cache │ │
│  │ Fetcher    │  │ Network  │  │Banks │ │
│  └────────────┘  └──────────┘  └──────┘ │
│                        │                │
│         ┌──────────────┼──────────┐     │
│         ▼              ▼          ▼     │
│    ┌─────────┐  ┌──────────┐  ┌──────┐ │
│    │ Conv    │  │  Pool    │  │ ReLU │ │
│    │ Engine  │  │   PE     │  │  PE  │ │
│    └─────────┘  └──────────┘  └──────┘ │
└──────────────────────────────────────────┘
```

---

## Custom Instruction Set

| Instruction | Function Code | Operation |
|---|---|---|
| cnn.rst | 1 | Local reset control |
| cnn.store.ke | 2 | Store kernel/filter to COE cache |
| cnn.store.img | 3 | Store input image to image cache |
| cnn.store.res | 4 | Load result from crossbar to result cache |
| cnn.load.res | 5 | Transfer result from cache to RISC-V |
| cnn.store.old | 6 | Store old data for multi-stage operations |
| cnn.load.old | 7 | Load old data to RISC-V |
| cnn.wm | 8 | Set PE working mode and crossbar config |
| cnn.sc | 9 | Start CNN computation |

---

## Convolution Engine Architecture

```text
pixel_in (8-bit stream)
       │
       ▼
┌─────────────────────────────────┐
│     Sliding-Window Line Buffer  │
│  row_buffer0, row_buffer1,      │
│  current_row → 3×3 or 5×5      │
│  window assembly                │
│  window_valid handshake         │
└────────────────┬────────────────┘
                 │ image_window [N×N]
                 ▼
┌─────────────────────────────────┐
│     25 Parallel Multipliers     │
│  mul_en enables per filter size │
│  1 clock cycle                  │
└────────────────┬────────────────┘
                 │ 25 partial products
                 ▼
┌─────────────────────────────────┐
│  4-level combinational adders   │
│  + 1 sequential accumulator     │
│  1 clock cycle                  │
└────────────────┬────────────────┘
                 │
                 ▼
         conv_result (32-bit)
         finish signal
         Total: 2 clock cycles
```

---

## Controller FSM

```text
         ┌─────────────────┐
    ┌───▶│      IDLE       │◀───┐
    │    │ Await cfg_valid │    │
    │    └────────┬────────┘    │
    │             │             │
    │             ▼             │
    │    ┌─────────────────┐    │
    │    │     CONFIG      │    │
    │    │ Decode 9 CNN    │    │
    │    │ instructions    │    │
    │    │ Configure PEs,  │    │
    │    │ crossbar, cache │    │
    │    └────────┬────────┘    │
    │             │ cnn.sc only │
    │             ▼             │
    │    ┌─────────────────┐    │
    │    │    EXECUTE      │    │
    │    │ Enable PEs      │    │
    │    │ Configure paths │    │
    │    │ Initiate memory │    │
    │    └────────┬────────┘    │
    │             │             │
    │             ▼             │
    │    ┌─────────────────┐    │
    │    │ WAIT_COMPLETION │    │
    │    │ Monitor finish  │    │
    │    │ signals from    │    │
    │    │ all active PEs  │    │
    │    └────────┬────────┘    │
    │             │             │
    │             ▼             │
    │    ┌─────────────────┐    │
    └────│  WRITE_BACK     │────┘
         │ Collect results │
         │ Write to cache  │
         │ Cleanup         │
         └─────────────────┘
```

---

## How to Run

**Tools required:** Xilinx Vivado, GCC RISC-V toolchain

**Simulate convolution unit:**
```tcl
vlib work
vlog conv_unit.v tb_conv.v
vsim -do "run -all" work.tb_conv
```

**Synthesize (coprocessor only):**
Open Vivado, target Artix-7 (xc7a35t), add RTL sources, run 
implementation.

**Software integration:**
```c
#include "cnn_lib.h"

// Store kernel
cnn_store_ke(kernel_addr, kernel_size);

// Store input image
cnn_store_img(image_addr, image_size);

// Start convolution
cnn_start(CONV_MODE, filter_size, num_pe);

// Load result
cnn_load_res(result_addr);
```

---

## Validation

- End-to-end validation with LeNet-5 — C1 and S2 layers implemented 
  and verified.
- Python reference model comparison — 100% output match confirmed.
- Sobel edge detection — validated reconfigurability beyond CNN 
  workloads.
- FIR filtering — validated 1D convolution mode for signal 
  processing.

---

## Known Limitations and Future Work

- Full-system synthesis (RISC-V + coprocessor together) was not 
  completed — only separate synthesis results available.
- LeNet-5 implementation covers C1 and S2 layers; remaining layers 
  are future work.
- Full ASIC flow is planned but not yet completed.
- Parallel MAC arrays across multiple images (further convolution 
  speedup) are planned.
- Formal verification of the controller FSM is not yet done.

---

## Team Contributions

This project was developed as a team. My personal contributions 
covered the convolution engine, accelerator controller, 8-way 
set-associative cache, address generation, and system integration. 
Team members developed the pooling, ReLU, matrix addition processing 
elements, reconfigurable crossbar, data fetcher, decoder, EAI 
interface, and software/toolchain integration.

---

## Reference Paper

This implementation is based on and extends:

Wu, N. et al. "A Reconfigurable Convolutional Neural Network-
Accelerated Coprocessor Based on RISC-V Instruction Set." 
*Electronics*, vol. 11, no. 6, article 1005, 2022.

Our convolution implementation achieved **1,837 cycles** versus the 
paper's **2,070 cycles** — a 10% improvement over the reference 
baseline.

---

## Keywords

`CNN` `RISC-V` `Hardware Accelerator` `E203` `NICE Interface` 
`Custom Instructions` `Convolution Engine` `MAC Datapath` 
`Set-Associative Cache` `FSM Controller` `Edge AI` `FPGA` `Vivado` 
`Artix-7` `Digital IC Design`
