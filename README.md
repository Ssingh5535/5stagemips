# MIPS 5-Stage Pipeline CPU

## Overview

This repository contains a fully pipelined 5-stage MIPS CPU design written in Verilog, along with a comprehensive testbench (`TB_MIPS_CPU.v`) for functional verification in ModelSim. It also includes scripts and instructions for synthesis and timing analysis in Intel Quartus.

Key features:

* Implements all five classic pipeline stages: IF, ID, EX, MEM, WB
* Forwarding and hazard detection to resolve data hazards
* Branch and jump control with flush and stall logic
* Parameterizable instruction and data memories
* 7‑segment display output support for register/debug values

## Repository Structure

```
├── src/
│   ├── M_7SEG_DECODER.v      # 7-segment display driver
│   ├── M_ADDER.v             # 32-bit adder (used for PC+4 and branch target)
│   ├── M_ALU.v               # Arithmetic Logic Unit
│   ├── M_ALU_CONTROLLER.v    # Generates ALU control signals
│   ├── M_CONTROLLER.v        # Main control unit (opcode → control signals)
│   ├── M_EQUAL.v             # Comparator used for branch decisions
│   ├── M_EX_MEM_REG.v        # EX/MEM pipeline register
│   ├── M_HAZARD_UNIT.v       # Hazard detection & forwarding unit
│   ├── M_ID_EX_REG.v         # ID/EX pipeline register
│   ├── M_IF_ID_REG.v         # IF/ID pipeline register
│   ├── M_MEM_ASYNC.v         # Asynchronous instruction memory (ROM)
│   ├── M_MEM_SYNC.v          # Synchronous data memory (RAM)
│   ├── M_MEM_WB_REG.v        # MEM/WB pipeline register
│   ├── M_MIPS_CPU.v          # Top-level CPU integrating all modules
│   ├── M_MUX_2.v             # 2-to-1 multiplexer
│   ├── M_MUX_2_DONTCARE.v    # 2-to-1 mux ("don't-care" timing)
│   ├── M_MUX_3_DONTCARE.v    # 3-to-1 mux for forwarding paths
│   ├── M_PC_ADDER.v          # Adds immediate offsets (branch target)
│   ├── M_PC_REG.v            # Program counter register with stall/flush
│   ├── M_REG_FILE.v          # 32×32 register file
│   ├── M_SEXT_16.v           # Sign-extension of 16-bit immediate
│   ├── M_SLL2.v              # Shift-left-by-2 (for branch offset)
│   └── M_DISPLAY_RESULTS_S1.v# (Optional) Drive 7‑segment displays
├── tb/
│   └── TB_MIPS_CPU.v         # Testbench for the MIPS CPU
└── images/                   # Pipeline diagrams & simulation waveforms
    ├── pipeline_diagram.png
    ├── if_stage.png
    ├── id_stage.png
    └── waveform.png
```

## Pipeline Overview

The CPU implements the following sequence of stages, each separated by pipeline registers:

1. **Instruction Fetch (IF)**

   * **Modules:** `M_PC_REG`, `M_PC_ADDER`, `M_MEM_ASYNC` (IMEM)
   * **Function:** Fetch instruction from instruction memory based on `PC`. Compute `PC + 4` for the next sequential fetch.
   * **Artifacts:** Output `instr` and `pc + 4` are stored in the IF/ID register.

   ![IF Stage](images/if_stage.png)

2. **Instruction Decode (ID)**

   * **Modules:** `M_REG_FILE`, `M_SEXT_16`, `M_CONTROLLER`, `M_ALU_CONTROLLER`, `M_HAZARD_UNIT`
   * **Function:** Decode opcode & funct fields to generate control signals. Read source registers from the register file. Sign-extend immediate values. Detect hazards.
   * **Artifacts:** Signals and operands forwarded into ID/EX register.

   ![ID Stage](images/id_stage.png)

3. **Execute (EX)**

   * **Modules:** `M_ALU`, `M_SLL2`, `M_PC_ADDER`, forwarding multiplexers `M_MUX_3_DONTCARE`
   * **Function:** Perform ALU operations (add, sub, logic, shift). Calculate branch target by shifting the sign-extended immediate and adding to `pc + 4`. Resolve data hazards via forwarding.

4. **Memory Access (MEM)**

   * **Modules:** `M_MEM_SYNC` (DMEM), `M_EX_MEM_REG`
   * **Function:** If the instruction is a load/store, access data memory. The branch decision (zero flag & control) is finalized here.

5. **Write Back (WB)**

   * **Modules:** `M_MUX_2`, `M_MEM_WB_REG`, `M_REG_FILE`
   * **Function:** Select between ALU result and data memory output, then write back to the register file.

The complete datapath and control flow is summarized below:

![Pipeline Diagram](images/pipeline_diagram.png)

## Detailed Module Descriptions

* **`M_CONTROLLER.v`**: Main finite‑state control that translates the 6‑bit opcode into control signals (`RegDst`, `ALUSrc`, `MemtoReg`, `RegWrite`, `MemRead`, `MemWrite`, `Branch`, `Jump`, `ALUOp`).
* **`M_ALU_CONTROLLER.v`**: Combines `Funct` bits with the 2‑bit `ALUOp` to produce a 3‑bit `ALUControl` code.
* **`M_ALU.v`**: Performs arithmetic and logic operations based on `ALUControl`.
* **`M_HAZARD_UNIT.v`**: Detects load-use hazards and generates stall/flush signals. Also determines forwarding paths for EX stage.
* **Pipeline Registers:**

  * `M_IF_ID_REG.v`, `M_ID_EX_REG.v`, `M_EX_MEM_REG.v`, `M_MEM_WB_REG.v` hold control & data signals between stages.
* **`M_MEM_ASYNC.v` / `M_MEM_SYNC.v`**: Parameterizable memories; instruction memory is asynchronous, data memory is synchronous.
* **`M_REG_FILE.v`**: Implements a 32×32 register file with two read ports and one write port. Also drives six 7‑segment outputs for monitoring.
* **Utility Modules:** Multiplexers (`M_MUX_2.v`, `M_MUX_3_DONTCARE.v`), sign‑extension (`M_SEXT_16.v`), shift‑left-by-2 (`M_SLL2.v`), comparators (`M_EQUAL.v`).

## Simulation Setup (ModelSim)

1. **Compile all sources:**

   ```bash
   vlib work
   vlog src/*.v tb/TB_MIPS_CPU.v
   ```
2. **Launch simulation:**

   ```bash
   vsim -c TB_MIPS_CPU -do "add wave -r /*; run -all; quit"
   ```
3. **Viewing waveforms:**

   * Use `add wave` commands to insert signals (e.g., `add wave -position end sim:/TB_MIPS_CPU/*`).
   * Run for the desired number of cycles (e.g., `run 500ns`).

Example waveform of a simple arithmetic instruction:

![Waveform](images/waveform.png)

## Synthesis & Implementation (Quartus)

1. Open Quartus and create a new project.
2. Add all `.v` files under `src/` to the project.
3. Set `M_MIPS_CPU` as the Top-Level Entity.
4. Define the clock constraint (e.g., 100 MHz) in the Timing Analyzer (TimeQuest).
5. Assign FPGA pins for `clk`, `reset`, and 7‑segment outputs as per your board.
6. Compile the design and review resource utilization and timing reports.

## License & Attribution

This project is provided for educational use. Feel free to modify and extend for your own research.

---

**Author:** Stephen Singh

**Contact:** [stephen.singh@example.com](mailto:stephen.singh@example.com)
