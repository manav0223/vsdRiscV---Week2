<div align="center">
 
# Week 2 : Part 2
# BabySoC Fundamentals & Functional Modelling

</div>

<div align="center">
 
[![RISC-V](https://img.shields.io/badge/RISC--V-SoC%20Tapeout-blue?style=for-the-badge&logo=riscv)](https://riscv.org/)
[![VSD](https://img.shields.io/badge/VSD-Program-orange?style=for-the-badge)](https://vsdiat.vlsisystemdesign.com/)
![Week](https://img.shields.io/badge/Week-2-pink?style=for-the-badge)

</div>

## 📂 Project Structure

```txt
VSDBabySoC/
├── src/
│   ├── include/      # Header files (*.vh)
│   ├── module/       # Verilog + TLV modules
│   │   ├── vsdbabysoc.v   # Top-level module
│   │   ├── rvmyth.v       # CPU
│   │   ├── avsdpll.v      # PLL
│   │   ├── avsddac_stub.v      # DAC
│   │   └── testbench.v    # Testbench
└── output/           # Simulation outputs
```

## 🛠️ Setup

### 📥 Cloning the Project
```
cd VLSI
git clone https://github.com/manili/VSDBabySoC.git
cd VSDBabySoC
```

## 🔧 TLV → Verilog Conversion

Since **RVMYTH** is written in **TL-Verilog (.tlv)**, we need to convert it to Verilog before simulating.

```
# Install tools
sudo apt update
sudo apt install python3-venv python3-pip

# Create virtual env
python3 -m venv sp_env
source sp_env/bin/activate

# Install SandPiper-SaaS
pip install pyyaml click sandpiper-saas

# Convert TLV → Verilog
sandpiper-saas -i ./src/module/*.tlv -o rvmyth.v --bestsv --noline -p verilog --outdir ./src/module/
```

Now you’ll have `rvmyth.v` alongside your other Verilog files.


## 🧪 Simulation Flow
## 🔹 Pre-Synthesis Simulation
As we have already installed Icarus Verilog & GTKWave, we can go ahead with the synthesis
```
mkdir -p output/pre_synth_sim

iverilog -o output/pre_synth_sim/pre_synth_sim.out \
  -DPRE_SYNTH_SIM \
  -I src/include -I src/module \
  src/module/testbench.v
cd output/pre_synth_sim
./pre_synth_sim.out
```
To view the simulation waveforms, we will open the vcd file in GTKWave
```
gtkwave pre_synth_sim.vcd
```
<img width="1851" height="796" alt="Pre_Synth GTK Wave" src="https://github.com/user-attachments/assets/7a3f5168-3d38-4dae-a22c-f3f02c62616e" />
<img width="967" height="189" alt="Pre_Synth GTK Log" src="https://github.com/user-attachments/assets/9d5daf4e-f60a-4e15-b3fd-146c302c6ac3" />
### 🧠 Pre-Synthesis Simulation Waveform Analysis  


---

### 📘 Overview
This document provides a detailed explanation of the waveform signals captured from the **pre-synthesis simulation** of the `vsdbabysoc` design.  
The simulation includes CPU, DAC, and PLL modules integrated within the SoC.  
All signals were analyzed from the GTKWave output to verify correct system behavior.

---

### 🧩 Module Hierarchy
```
vsdbabysoc_tb
└── uut
    ├── core    → CPU core and memory interface  
    ├── dac     → Digital-to-Analog Converter  
    └── pll     → Phase-Locked Loop subsystem  
```

---

### ⚙️ Observations from Simulation Waveform

### Signal-by-Signal Analysis

#### 1. **CLK (System Clock)**
- **Type:** `reg`  
- **Description:** Primary reference clock driving all synchronous logic.  
- **Waveform:** Clean, periodic square wave (~50 ns period).  

---

#### 2. **CPU_dmem_addr_a4[3:0]**
- **Type:** 4-bit bus  
- **Description:** CPU data memory address lines.  
- **Behavior:** Rapid toggling during memory access cycles, showing active program execution.  

---

#### 3. **OUT[9:0]**
- **Type:** 10-bit bus  
- **Description:** Digital output from DAC input register.  
- **Behavior:**  
  - Undefined (`x`) during reset.  
  - Periodic sawtooth-like pattern after reset — indicates waveform generation.  

---

#### 4. **reset**
- **Type:** `wire`  
- **Description:** System-wide asynchronous reset (active HIGH).  
- **Behavior:**  
  - Asserted during initialization (~0–200 ns).  
  - Deasserted afterward to start normal operation.  

---

#### 5. **Dext[10:0]**
- **Type:** 11-bit bus  
- **Description:** External or intermediate data bus feeding DAC.  
- **Behavior:**  
  - Unknown at startup.  
  - Transitions to valid sample values post-reset.  

---

#### 6. **EN (Enable Signal)**
- **Type:** `wire`  
- **Description:** General enable control for active modules.  
- **Behavior:** Stays HIGH during operation.  

---

#### 7. **NaN (Invalid Flag)**
- **Type:** `wire`  
- **Description:** Indicator for invalid or undefined analog computations.  
- **Behavior:** Constant `nan`.  

---

#### 8. **VREFH / VREFL**
- **Type:** `wire`  
- **Description:** DAC reference voltages (high/low).  
- **Behavior:**  
  - `VREFH` = Constant `1`  
  - `VREFL` = Constant `0`  

---

#### 9. **OUT (Analog Output)**
- **Type:** `real`  
- **Description:** DAC analog output.  
- **Behavior:** Smooth periodic waveform, confirming digital-to-analog conversion.  

---

#### 10. **PLL Internal Signals**

| Signal | Type | Description | Behavior |
|:-------|:------|:-------------|:-----------|
| **CLK** | `reg` | Internal PLL clock output | High-frequency multiplied version of input clock |
| **ENb_CP** | `wire` | Charge pump enable (active low) | Low → Active; shows PLL feedback loop operation |
| **ENb_VCO** | `wire` | VCO enable (active low) | Constant low, enabling continuous operation |
| **REF** | `wire` | PLL reference clock | Periodic slower pulse used for phase comparison |
| **VCO_IN** | `wire` | Input to Voltage Controlled Oscillator | Fast periodic waveform controlled by loop feedback |
| **lastedge** | `real` | Time of last clock edge | Increments by ~283 ns between edges |
| **period** | `real` | Computed signal period | Stabilizes around 35.4 ns after lock |
| **refpd** | `real` | Reference phase detector output | Locks at ~283.33 ns — indicates phase alignment |

---

#### 🔁 Operational Phases

##### 1. **Reset Phase (0 – ~200 ns)**
- All digital outputs = `x` (unknown)
- PLL and DAC idle
- No valid waveform generation

##### 2. **Startup Phase (~200 ns – 40 µs)**
- Reset released
- CPU begins address cycles (`CPU_dmem_addr_a4`)
- PLL starts phase alignment (`refpd` and `period` adjusting)
- DAC begins digital-to-analog conversions

##### 3. **Steady-State Phase (> 40 µs)**
- PLL achieves lock (stable `period` and `refpd`)
- DAC outputs consistent analog waveform
- CPU, PLL, and DAC fully synchronized

---

#### 📈 Functional Flow Summary

```
CPU → Dext[10:0] → DAC → OUT[9:0] → OUT(real)
          ↑
          └──── PLL synchronizes via REF & VCO_IN
```

**System Behavior:**
- CPU generates digital data samples.
- DAC converts them to analog waveform.
- PLL ensures stable timing and frequency control.
- The overall output is a stable, periodic analog signal.

---

#### ✅ Key Takeaways

- **PLL Lock Verified:** `refpd` and `period` stabilize at steady-state.  
- **DAC Output Verified:** Digital values successfully generate analog waveform.  
- **CPU Activity Confirmed:** Active memory addressing and data flow visible.  
- **System Integration Successful:** All modules operate coherently post-reset.

---

#### 📂 Files
| File | Description |
|:------|:-------------|
| `pre_synth_sim.vcd` | Value Change Dump (VCD) file for GTKWave |
| `Pre_Synth GTK Wave.png` | Screenshot of waveform viewer |
| `vsdbabysoc_tb.v` | Top-level testbench |
| `uut/core.v`, `uut/dac.v`, `uut/pll.v` | Design modules under test |

---

#### 🛠️ Tools & Environment
- **Simulator:** Icarus Verilog  
- **Waveform Viewer:** GTKWave  
- **Operating System:** Ubuntu Linux  
- **Date of Simulation:** October 5, 2025  

---

#### 🧾 Summary
This pre-synthesis waveform confirms correct functional interaction between the CPU, DAC, and PLL subsystems of the `vsdbabysoc` SoC.  
The design successfully transitions from reset to normal operation, achieves PLL lock, and produces a valid analog output waveform.


## 🔹 Post-Synthesis Simulation

For Post Synthesis, we need a vsdbabysoc.synth.v file which we will get by doing synthesis using Yosys

#### 1.
```
source sp_env/bin/activate   
```
##### 🧠 What it does
This command activates a Python virtual environment called sp_env.
A virtual environment is an isolated Python workspace — it contains its own version of Python, along with specific packages (like yosys, cocotb, matplotlib, etc.) that won’t interfere with your system-wide Python setup.

#### 2. Open Yosys
```
yosys
```
#### 3. First we will read all verilog design files into Yosys so it can analyze and synthesize them
```
read_verilog -I ./src/include ./src/module/clk_gate.v
read_verilog -I ./src/include ./src/module/rvmyth.v
read_verilog -I ./src/include ./src/module/avsdpll_stub.v
read_verilog -I ./src/include ./src/module/avsddac_stub.v
read_verilog -I ./src/include ./src/module/vsdbabysoc.v
```
<img width="1132" height="1080" alt="Reading all Verilog files" src="https://github.com/user-attachments/assets/f3183671-6390-45b6-890c-dfa506d8220f" />

#### 4. Now we run the Synthesis process in Yosys using the module vsdbabysoc as the top-level design.
```
synth -top vsdbabysoc
```
<img width="1850" height="1080" alt="Synthesis of vsdbabysoc" src="https://github.com/user-attachments/assets/07a609f4-611b-4b38-8e8a-f908ab61a973" />

#### 5. Now exit Yosys and make a directory
```
mkdir -p output/synthesized
```
#### 6. Now we will go back to Yosys and run
```
yosys
write_verilog output/synthesized/vsdbabysoc.synth.v
```
<img width="1850" height="361" alt="Writing netlist for vsdbabysoc" src="https://github.com/user-attachments/assets/92103a26-a381-4b99-8145-a58fe1dc56a9" />

##### Yosys does the following:
- i). Generates a Verilog netlist — consisting of logic gates, flip-flops, and other primitives.
- ii). Flattens the hierarchy (optional) — depending on synthesis steps, submodules may be merged.
- iii). Writes that netlist into a .v file.

##### Directory Structure after running
```
output/
 └── synthesized/
      └── vsdbabysoc.synth.v
```
#### 7. Now we will exit the Yosys and will perform Post-Synthesis Simulation
```
iverilog -o output/post_synth_sim/post_synth_sim.out -DPOST_SYNTH_SIM \
    -I src/include -I src/module -I src/gls_model \
    src/module/testbench.v \
    output/synthesized/vsdbabysoc.synth.v \
    src/module/avsddac_stub.v
```

##### On running this we got a error: 
```
src/gls_model/sky130_fd_sc_hd.v:67667: syntax error
src/gls_model/sky130_fd_sc_hd.v:67667: error: Invalid module item.
```
<img width="1811" height="334" alt="Error at line 67667 " src="https://github.com/user-attachments/assets/ee7b46c6-e93d-451a-94a7-38d168f3dc25" />

To rectify the error, first located the line inside the sky130_fd_sc_hd.v file using the command:
```
nl -ba src/gls_model/sky130_fd_sc_hd.v | sed -n '67650,67680p'
```
Then opened the file in the vi editor using the command
```
vi sky130_fd_sc_hd.v
```
###### Make a directory
```
mkdir -p output/post_synth_sim
```

##### After making the changes, saved the file and re-ran the command

```
iverilog -o output/post_synth_sim/post_synth_sim.out -DPOST_SYNTH_SIM \
  -I src/include -I src/module -I src/gls_model \
  src/module/testbench.v \
  output/synthesized/vsdbabysoc.synth.v \
  src/module/avsddac_stub.v
```

##### Now we will get the vcd file name using the following command:
```
vvp output/post_synth_sim/post_synth_sim.out
```
<img width="913" height="249" alt="VCD File for Post_Synth" src="https://github.com/user-attachments/assets/d1cd059c-7b55-456a-95a7-8f80c3d65f7a" />
    
#### 8. Now we will open GTKWave for post synthesis file using the following command:
```
gtkwave post_synth_sim.vcd
```
<img width="1855" height="678" alt="Post_Synth GTK Wave" src="https://github.com/user-attachments/assets/0a41ce48-f8df-41ea-9592-e5424784ce99" />
<img width="767" height="185" alt="Post_Synth GTK Log" src="https://github.com/user-attachments/assets/7e1899eb-652c-452c-87dd-1ffef5b7186e" />

### 🧩 Post-Synthesis Waveform Analysis — VSDBabySoC


---

#### 🧠 Overview

The waveform represents the interaction between the **RISC-V core (rvmyth)**, **PLL**, and **DAC** after synthesis.

**Hierarchy in GTKWave:**
```
vsdbabysoc_tb
└── uut
    ├── core
    ├── dac
    └── pll
```

The signals confirm that:
- The PLL is locked and generating a stable output clock.
- The CPU core is active, producing addresses and data.
- The DAC outputs digital data patterns consistent with analog waveform generation.

---

#### 🧩 Signals Observed

##### **1. CLK (System Clock)**
**Type**: `reg`  
**Waveform**: Clean periodic square wave  
**Observation**:  
- Runs continuously with stable period.
- Drives the entire SoC.

---

##### **2. reset**
**Type**: `wire`  
**Value**: `0` (inactive)  
**Observation**:  
- System is running normally (reset deasserted).

---

##### **3. CPU_dmem_addr_a4[3:0]**
**Type**: 4-bit bus  
**Values shown**: `8`, `F`, `B`, `A`, `6`, `2`, `E`, `3`, ...  
**Observation**:  
- CPU is actively generating memory addresses.
- Confirms correct core functionality and memory activity.

---

##### **4. D[9:0]**
**Type**: 10-bit bus  
**Values shown**: `0AB`, `0BE`, `099`, `088`, `078`  
**Observation**:  
- Represents digital data being sent to the DAC.
- Gradual decrement pattern shows waveform shaping behavior.

---

##### **5. OUT**
**Type**: wire  
**Value**: `z` (high-impedance / undefined at this instant)  
**Observation**:  
- May represent the analog DAC output before settling.

---

##### **6. VREFH**
**Type**: wire  
**Value**: `1`  
**Observation**:  
- High reference voltage for DAC.
- Indicates proper biasing.

---

##### **7. PLL Section Signals**

###### **CLK (PLL Output Clock)**
- Stable, high-frequency square wave.
- Drives synchronized logic blocks.

###### **ENb_CP (Enable Charge Pump)**
- Value: `x` (unknown or inactive)
- May indicate inactive control path during steady state.

###### **ENb_VCO (Enable VCO)**
- Value: `1`
- VCO enabled, PLL operating in locked condition.

###### **REF**
- Value: `1`
- Reference input to the PLL for phase detection.

###### **VCO_IN**
- Value: `1`
- Represents active internal oscillation from the VCO.

###### **lastedge**
- Value: `42032.84 ns`
- Tracks timing of the last detected edge of the VCO.

###### **period**
- Value: `35.41625 ns`
- Shows PLL has stabilized with consistent clock period.

###### **refpd**
- Value: `283.33 ns`
- Indicates a constant phase difference, confirming PLL lock.

---

#### 🧭 Key Observations

| Signal | Description | Value/Behavior |
|---------|--------------|----------------|
| **CLK** | System clock | Stable, periodic |
| **reset** | Reset input | Inactive (`0`) |
| **CPU_dmem_addr_a4[3:0]** | CPU address bus | Active transitions |
| **D[9:0]** | DAC digital input | Sequential digital pattern |
| **OUT** | DAC output | High-impedance (`z`) |
| **VREFH** | DAC ref high | Constant high |
| **ENb_VCO** | VCO enable | Active (`1`) |
| **period** | PLL clock period | `35.416 ns` |
| **refpd** | PLL phase detector output | `283.33 ns` |
| **PLL lock** | Status | Achieved (stable period + refpd) |

---

#### 🧩 Functional Flow

```
CPU (rvmyth)
   ↓
CPU_dmem_addr_a4[3:0] → Data Memory
   ↓
D[9:0] → DAC Input
   ↓
OUT → Analog Output
```

Simultaneously, the PLL generates a stable, high-frequency system clock:

```
REF → Phase Detector → Charge Pump → VCO → CLK_out
```

---

#### ✅ Summary

| Feature | Status | Notes |
|----------|---------|-------|
| Reset handling | ✅ | Reset is inactive; SoC operational |
| CPU activity | ✅ | Memory addresses and data active |
| PLL locking | ✅ | Stable period = 35.416 ns |
| DAC signals | ✅ | Proper digital waveform |
| VREF signals | ✅ | Bias levels correct |
| Simulation correctness | ✅ | Matches expected post-synthesis behavior |



#### ⚠️ Troubleshooting Tips

| Issue | Possible Cause | Fix |
|-------|----------------|-----|
| `Unknown module avsddac_stub` | Missing DAC stub file | Include `src/module/avsddac_stub.v` |
| `No waveform activity` | Clock not running or reset stuck | Check PLL inputs, reset polarity |
| `X or Z values persist` | Improper connections or floating nets | Verify top-level port mapping |
| `CLK unstable` | PLL not configured properly | Ensure `ENb_VCO` and `ENb_CP` are set correctly |

---

#### 🧾 Summary

- The **post-synthesis waveform** should closely match RTL-level behavior, with minor propagation delays.
- The **key verification goal** is that synthesized logic retains functionality after synthesis.
- Always confirm:
  - ✅ Clock integrity
  - ✅ Reset release
  - ✅ Signal activity through `RV_TO_DAC` and `OUT`

---

## 🧰 Toolchain Information

| Tool | Version (Typical) |
|------|-------------------|
| Yosys | `0.32+` |
| Icarus Verilog | `11.0+` |
| GTKWave | `3.3.100+` |


