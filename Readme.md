# RISC-V Processor Design and FPGA Implementation

## Table of Contents

* [Overview](#overview)
* [Repository Structure](#repository-structure)
* [Part 1 – Single-Cycle RV32I Processor](#part-1--single-cycle-rv32i-processor)

  * [Architecture](#architecture--single-cycle)
  * [Core Modules](#core-modules--single-cycle)
  * [Datapath Diagram](#datapath-diagram--single-cycle)
* [Part 2 – 5-Stage Pipelined RV32I Processor](#part-2--5-stage-pipelined-rv32i-processor)

  * [Architecture](#architecture--pipeline-rv32i)
  * [Core Modules](#core-modules--pipeline-rv32i)
  * [Datapath Diagram](#datapath-diagram--pipeline-rv32i)
* [Part 3 – Pipelined RV32IMC Processor](#part-3--pipelined-rv32imc-processor)
* [Supported Instruction Set](#supported-instruction-set)
* [Hazard Handling](#hazard-handling)
* [FPGA Platforms](#fpga-platforms)
* [Tools & Prerequisites](#tools--prerequisites)
* [Getting Started](#getting-started)
* [Simulation](#simulation)
* [FPGA Implementation](#fpga-implementation)
* [Performance Counters](#performance-counters)
* [Project Highlights](#project-highlights)

---

## Overview

This project presents a ground-up implementation of **32-bit RISC-V processors** in **Verilog HDL**, developed in stages from a basic single-cycle design to a fully pipelined processor with hazard management, and finally expanded into the **RV32IMC** instruction set.

All three designs were simulated, synthesized, and tested on real **FPGA hardware** using **Xilinx Vivado**. The project reflects a systematic approach to processor design, covering datapath construction, control logic, pipeline integration, verification, and hardware deployment.

| Design       | ISA     | Architecture     | Target FPGA                      |
| ------------ | ------- | ---------------- | -------------------------------- |
| Single-Cycle | RV32I   | Single-cycle     | ZedBoard (Zynq-7000)             |
| Pipeline     | RV32I   | 5-stage pipeline | ZedBoard (Zynq-7000)             |
| Pipeline     | RV32IMC | 5-stage pipeline | ZedBoard (Zynq-7000) + Genesys 2 |

---

## Repository Structure

```
RISCV-Processor-Design-and-FPGA-Implementation/
│
├── Single Cycle RV32I/
│   ├── Sources/              # Verilog source files
│   ├── Testbench/            # Simulation testbench
│   ├── Constraints/          # XDC constraint files
│   └── Images/               # Architecture diagrams
│
├── Pipeline RV32I/
│   ├── Sources/              # Verilog source files
│   ├── Testbench/            # Simulation testbench
│   ├── Constraints/          # XDC constraint files
│   └── Images/               # Architecture diagrams
│
├── Pipeline RV32IMC/
│   ├── Sources/              # Verilog source files
│   ├── Testbench/            # Simulation testbench
│   ├── Constraints/          # XDC constraint files
│   └── Images/               # Architecture diagrams
│
└── README.md
```

---

## Part 1 – Single-Cycle RV32I Processor

### Architecture – Single-Cycle

The single-cycle processor completes each instruction in a single clock cycle. For every instruction, the datapath elements operate together in one pass, with the control unit selecting the appropriate execution path based on the opcode.

```
         ┌──────────┐    ┌──────────────────┐    ┌──────────┐
  CLK ──►│    PC    │───►│ Instruction Mem  │───►│ Control  │
         └──────────┘    └──────────────────┘    │  Unit    │
               ▲                  │               └──────────┘
               │            Instruction                │
               │                  ▼               Control Signals
               │         ┌──────────────┐              │
               │         │  Reg File    │◄─────────────┤
               │         └──────────────┘              │
               │                  │                    │
               │                  ▼                    │
               │             ┌─────────┐               │
               │             │   ALU   │◄──────────────┤
               │             └─────────┘               │
               │                  │                    │
               │                  ▼                    │
               │          ┌─────────────┐              │
               └──────────│  Data Mem   │              │
                          └─────────────┘              │
```

### Core Modules – Single-Cycle

| Module               | File                   | Description                                                                                     |
| -------------------- | ---------------------- | ----------------------------------------------------------------------------------------------- |
| `PC`                 | `PC.v`                 | Program counter that stores and updates the current instruction address                         |
| `Inst_Memory`        | `Inst_Memory.v`        | Read-only instruction memory initialized with the program                                       |
| `Reg_File`           | `Reg_File.v`           | 32 × 32-bit general-purpose registers, with `x0` hardwired to zero                              |
| `ALU`                | `ALU.v`                | Executes arithmetic and logic operations such as ADD, SUB, AND, OR, XOR, SLL, SRL, SRA, and SLT |
| `Control_Unit`       | `Control_Unit.v`       | Decodes the opcode and generates control signals for the datapath                               |
| `Imm_Gen`            | `Imm_Gen.v`            | Extracts and sign-extends immediate values for the supported instruction formats                |
| `Data_Memory`        | `Data_Memory.v`        | Read/write memory used for load and store instructions                                          |
| `PerformanceCounter` | `PerformanceCounter.v` | Tracks total cycles and retired instructions                                                    |
| `Top`                | `Top.v`                | Top-level module that connects all processor blocks                                             |
| `Testbench`          | `Testbench.v`          | Simulation testbench used for verification                                                      |

### Datapath Diagram – Single-Cycle

![Single Cycle RISCV Datapath](Single%20Cycle%20RV32I/Images/Single_Cycle_RISCV_Datapath.png)

---

## Part 2 – 5-Stage Pipelined RV32I Processor

### Architecture – Pipeline RV32I

The pipelined processor follows the standard **5-stage RISC pipeline**: Fetch, Decode, Execute, Memory, and Writeback. Pipeline registers are placed between stages so that multiple instructions can be processed at the same time.

```
  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
  │  FETCH  │──►│ DECODE  │──►│ EXECUTE │──►│ MEMORY  │──►│  WRITE  │
  │         │   │         │   │         │   │         │   │  BACK   │
  │  IF/ID  │   │  ID/EX  │   │ EX/MEM  │   │ MEM/WB  │   │         │
  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
       ▲               ▲            ▲
       │               │            │
  ┌────┴───────────────┴────────────┴──────┐
  │         Hazard & Forwarding Unit       │
  └────────────────────────────────────────┘
```

**Key features:**

* **Data forwarding** (EX-EX and MEM-EX) to reduce pipeline stalls
* **Hazard detection unit** for load-use cases, with a one-cycle bubble when needed
* **Branch flushing** to handle control hazards when a branch is taken

### Core Modules – Pipeline RV32I

| Module               | File                   | Description                                             |
| -------------------- | ---------------------- | ------------------------------------------------------- |
| `RISCV_Top`          | `RISCV_Top.v`          | Top-level module integrating all pipeline stages        |
| `fetch_cycle`        | `fetch_cycle.v`        | IF stage: instruction fetch and PC update               |
| `Inst_memory`        | `Inst_memory.v`        | Instruction memory                                      |
| `decode_cycle`       | `decode_cycle.v`       | ID stage: decode and register read                      |
| `Reg_file`           | `Reg_file.v`           | 32 × 32-bit register file, with `x0 = 0`                |
| `control_unit`       | `control_unit.v`       | Top-level control unit                                  |
| `main_decoder`       | `main_decoder.v`       | Generates high-level control signals from the opcode    |
| `ALU_decoder`        | `ALU_decoder.v`        | Selects the specific ALU operation                      |
| `Sign_extend`        | `Sign_extend.v`        | Immediate extraction and sign-extension                 |
| `execute_cycle`      | `execute_cycle.v`      | EX stage: ALU operations and branch evaluation          |
| `ALU`                | `ALU.v`                | Arithmetic Logic Unit                                   |
| `forwarding_unit`    | `forwarding_unit.v`    | Resolves data hazards using forwarding paths            |
| `hazard_unit`        | `hazard_unit.v`        | Manages stalls and flushes for hazard control           |
| `memory_cycle`       | `memory_cycle.v`       | MEM stage: data memory access                           |
| `Data_memory`        | `Data_memory.v`        | Data memory used for loads and stores                   |
| `writeback_cycle`    | `writeback_cycle.v`    | WB stage: writes results back to the register file      |
| `Mux`                | `Mux.v`                | Parameterized multiplexers used throughout the datapath |
| `PerformanceCounter` | `PerformanceCounter.v` | Counts cycles and retired instructions                  |
| `RISCV_Testbench`    | `RISCV_Testbench.v`    | Complete simulation testbench                           |

### Datapath Diagram – Pipeline RV32I

![Pipeline RISCV Datapath](Pipeline%20RV32IMC/Images/5%20stage%20pipeline%20datapath.png)

---

## Part 3 – Pipelined RV32IMC Processor

The **RV32IMC** processor extends the pipelined RV32I design with two standard RISC-V extensions:

| Extension                       | Description                                                                          |
| ------------------------------- | ------------------------------------------------------------------------------------ |
| **M** – Integer Multiply/Divide | Adds `MUL`, `MULH`, `MULHSU`, `MULHU`, `DIV`, `DIVU`, `REM`, and `REMU` instructions |
| **C** – Compressed Instructions | Adds 16-bit compressed instruction encodings to reduce code size                     |

This is the most feature-complete version in the repository and is targeted for both the **ZedBoard** and the **Genesys 2** FPGA platform.

---

## Supported Instruction Set

### RV32I Base Integer Instructions

| Category                | Instructions                               |
| ----------------------- | ------------------------------------------ |
| **Arithmetic (R-type)** | `ADD`, `SUB`, `SLT`, `SLTU`                |
| **Arithmetic (I-type)** | `ADDI`, `SLTI`, `SLTIU`                    |
| **Logical (R-type)**    | `AND`, `OR`, `XOR`                         |
| **Logical (I-type)**    | `ANDI`, `ORI`, `XORI`                      |
| **Shift (R-type)**      | `SLL`, `SRL`, `SRA`                        |
| **Shift (I-type)**      | `SLLI`, `SRLI`, `SRAI`                     |
| **Load**                | `LW`, `LH`, `LB`, `LHU`, `LBU`             |
| **Store**               | `SW`, `SH`, `SB`                           |
| **Branch**              | `BEQ`, `BNE`, `BLT`, `BGE`, `BLTU`, `BGEU` |
| **Jump**                | `JAL`, `JALR`                              |
| **Upper Immediate**     | `LUI`, `AUIPC`                             |

### RV32M Extension (Pipeline RV32IMC only)

| Category             | Instructions                     |
| -------------------- | -------------------------------- |
| **Multiply**         | `MUL`, `MULH`, `MULHSU`, `MULHU` |
| **Divide/Remainder** | `DIV`, `DIVU`, `REM`, `REMU`     |

### RV32C Extension (Pipeline RV32IMC only)

16-bit compressed encodings for common integer instructions, reducing instruction memory usage.

---

## Hazard Handling

The pipelined designs include hazard handling logic to keep execution correct and efficient.

### Data Hazards

| Hazard Type           | Resolution                                                                         |
| --------------------- | ---------------------------------------------------------------------------------- |
| **EX-EX Forwarding**  | The ALU result from the EX/MEM register is forwarded directly to the execute stage |
| **MEM-EX Forwarding** | Results from the MEM/WB register are forwarded to the execute stage when needed    |
| **Load-Use Hazard**   | A one-cycle stall is inserted by the hazard detection unit                         |

### Control Hazards

| Hazard Type      | Resolution                                                                          |
| ---------------- | ----------------------------------------------------------------------------------- |
| **Branch Taken** | Instructions in IF and ID are flushed, adding a two-cycle penalty                   |
| **JAL / JALR**   | The target PC is computed in the execute stage, and the IF/ID registers are flushed |

---

## FPGA Platforms

| Feature           | ZedBoard                 | Genesys 2                |
| ----------------- | ------------------------ | ------------------------ |
| **Device**        | Xilinx Zynq-7000 XC7Z020 | Xilinx Kintex-7 XC7K325T |
| **Logic Cells**   | 85,000                   | 326,080                  |
| **Block RAM**     | 560 KB                   | 16 MB                    |
| **Onboard Clock** | 100 MHz                  | 200 MHz                  |
| **Used by**       | All three designs        | Pipeline RV32IMC         |

### Board Resources Used

* Onboard system clock
* User push-button for synchronous reset
* User LEDs for live debug output

---

## Tools & Prerequisites

| Tool              | Version / Notes                       |
| ----------------- | ------------------------------------- |
| **Xilinx Vivado** | 2016.x or later (Design Suite)        |
| **FPGA Boards**   | Zedboard Zynq 7000, Kintex-7 Genesys2 |

---

## Getting Started

### 1. Open Vivado and Create a Project

1. Launch **Xilinx Vivado**.
2. Click **Create Project** and choose a project name and directory.
3. Select **RTL Project** and check *Do not specify sources at this time*.
4. Choose your target FPGA part:

   * ZedBoard: `xc7z020clg484-1`
   * Genesys 2: `xc7k325tffg900-2`

### 2. Add Sources

1. Go to **Add Sources** → **Add or Create Design Sources**.
2. Add all `.v` files from the relevant folder, such as `Single Cycle RV32I/Sources/`.
3. Go to **Add Sources** → **Add or Create Simulation Sources**.
4. Add the testbench `.v` file.
5. Go to **Add Sources** → **Add or Create Constraints**.
6. Add the `.xdc` constraint file for the target board.

---

## Simulation

### Running Behavioral Simulation

1. In Vivado, go to **Flow Navigator** → **Simulation** → **Run Behavioral Simulation**.
2. The waveform viewer opens automatically.
3. Add the signals you want to inspect, such as PC, register values, ALU output, and memory data.
4. Click **Run All** to execute the testbench.

### What to Observe

| Signal                | Description                                  |
| --------------------- | -------------------------------------------- |
| `PC_Current`          | Instruction address being executed           |
| `RegFile[x1]...[x31]` | Register values after each instruction       |
| `ALU_Result`          | ALU output for the current cycle             |
| `DataMem_Out`         | Data read from memory during load operations |
| `cycle_count`         | Total number of clock cycles                 |
| `instr_count`         | Number of retired instructions               |

### Example Simulation Output

```
=== Checkpoint 1 ===
x1 = 5
x2 = 3

=== Checkpoint 2 ===
x3 = 8       // x3 = x1 + x2

=== Checkpoint 3 ===
x4 = 2       // x4 = x2 - x1 ... (expected: 3 - 5 = -2 in 2's complement)
Memory[0] = 8
```

---

## FPGA Implementation

### Synthesis and Implementation Steps

1. In Vivado, run **Synthesis** (`Flow Navigator` → `Run Synthesis`).
2. Run **Implementation** (`Flow Navigator` → `Run Implementation`).
3. Review the **Timing Summary** and make sure there are no setup or hold violations (WNS ≥ 0).
4. Review the **Utilization Report** for LUT, FF, and BRAM usage.
5. Generate the **Bitstream** (`Flow Navigator` → `Generate Bitstream`).
6. Connect the FPGA board via USB-JTAG.
7. Open **Hardware Manager** → **Program Device**.

### Clock Divider

The processor runs at the full board clock frequency internally, but to make execution visible on LEDs, a **clock divider** is used to generate a slower heartbeat signal.

```verilog
// Example: divide 100 MHz down to ~1 Hz for LED visualization
reg [26:0] clk_div_counter;
reg        clk_slow;

always @(posedge clk) begin
    if (clk_div_counter == 27'd99_999_999) begin
        clk_div_counter <= 0;
        clk_slow <= ~clk_slow;
    end else begin
        clk_div_counter <= clk_div_counter + 1;
    end
end
```

---

## Performance Counters

The design includes counters that help evaluate runtime behavior and architectural efficiency.

* **Cycle counter**: records the total number of clock cycles consumed
* **Instruction counter**: records the number of retired instructions
* **CPI analysis**: enables comparison between the single-cycle and pipelined designs
* **Debug support**: helps verify execution behavior during simulation and FPGA testing

---

## Project Highlights

* Complete RV32I implementation developed in Verilog HDL
* Stepwise design progression from single-cycle to 5-stage pipelined architecture
* RV32IMC extension with multiply/divide and compressed instruction support
* Hazard handling through forwarding, stalling, and branch flushing
* Performance counters for cycle and instruction analysis
* Validated on real FPGA hardware, including ZedBoard and Genesys 2
* Modular and readable Verilog organization across processor stages
* Simulation testbenches and constraint files provided for each design
