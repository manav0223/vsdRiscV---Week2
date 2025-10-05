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

### 🔹 Post-Synthesis Simulation

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

#### 4. Now we run the Synthesis process in Yosys using the module vsdbabysoc as the top-level design.
```
synth -top vsdbabysoc
```
#### 5. Now exit Yosys and make a directory
```
mkdir -p output/synthesized
```
#### 6. Now we will go back to Yosys and run
```
yosys
write_verilog output/synthesized/vsdbabysoc.synth.v
```
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

To rectify the error, first located the line inside the sky130_fd_sc_hd.v file using the command:
```
nl -ba src/gls_model/sky130_fd_sc_hd.v | sed -n '67650,67680p'
```
Then opened the file in the vi editor using the command
```
vi sky130_fd_sc_hd.v
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
    
#### 8. Now we will open GTKWave for post synthesis file using the following command:
```
gtkwave post_synth_sim.vcd
```

## 🔍 Signals to Observe in GTKWave

When you open `post_synth_sim.vcd` in **GTKWave**, add the following key signals:

| Signal | Description | Expected Behavior |
|---------|--------------|-------------------|
| `reset` | System reset input | High (active) at start, then deasserted to start the system |
| `REF` | Reference clock input to PLL | Fixed-frequency square wave |
| `CLK` | Generated clock from `avsdpll` | Clean and stable oscillation after PLL lock |
| `RV_TO_DAC[9:0]` | 10-bit digital output from `rvmyth` core | Toggles dynamically, showing data activity |
| `OUT` | Output of DAC stub | Follows the pattern of `RV_TO_DAC` |
| `ENb_VCO`, `ENb_CP` | Control inputs to PLL | Static signals used to enable/disable the PLL |
| `VCO_IN` | Input clock to PLL | Used for frequency multiplication |

---

## 🧠 Expected Waveform Behavior

The waveform should clearly show these **four operational phases**:

### 🔹 Phase 1 — Reset Phase
- `reset` is asserted.
- `CLK` may be idle or undefined.
- No activity on `RV_TO_DAC` or `OUT`.

### 🔹 Phase 2 — PLL Lock Phase
- After deasserting `reset`, the PLL (`avsdpll`) begins stabilizing.
- `REF` and `VCO_IN` synchronize.
- `CLK` becomes a clean square wave.

### 🔹 Phase 3 — Core Execution Phase
- `rvmyth` CPU starts executing instructions.
- `RV_TO_DAC` bus shows toggling activity.
- `OUT` follows the pattern of `RV_TO_DAC`.

### 🔹 Phase 4 — Stable Operation
- `CLK` remains stable.
- Data signals transition periodically.
- No persistent `X` (unknown) or `Z` (high-impedance) values.

---

## 📊 Example Waveform Visualization

A simplified conceptual example of what you should observe in **GTKWave**:

```
| Signal     |   Time → → →
|-------------|------------------------------------------
| reset       | ────────┐__________
| REF         | ──▁──▁──▁──▁──▁──▁──▁──▁──
| CLK         | ______▁______▁______▁______▁
| RV_TO_DAC   | 000000 → toggling → various patterns
| OUT         | follows RV_TO_DAC transitions
```

---

## 🧩 Recommended GTKWave Signal Grouping

Group signals logically for easy observation:

```
vsdbabysoc/
 ├── reset
 ├── REF
 ├── CLK
 ├── RV_TO_DAC[9:0]
 ├── OUT
 ├── ENb_VCO
 ├── ENb_CP
 └── VCO_IN
```

To add them:
1. Open GTKWave.
2. Click **SST → Search Signals**.
3. Drag these signals into the waveform view.
4. Use **“Zoom Fit”** or “Zoom Out” to see the full simulation window.

---

## ✅ Successful Simulation Indicators

If everything is correct, you should see:

- ✔️ No compilation or elaboration errors in `iverilog` or `vvp`
- ✔️ `CLK` starts oscillating after `reset` deasserts
- ✔️ `RV_TO_DAC` bus toggles dynamically after reset
- ✔️ `OUT` reflects DAC activity
- ✔️ No continuous `X` or `Z` signals in GTKWave

---

## ⚠️ Troubleshooting Tips

| Issue | Possible Cause | Fix |
|-------|----------------|-----|
| `Unknown module avsddac_stub` | Missing DAC stub file | Include `src/module/avsddac_stub.v` |
| `No waveform activity` | Clock not running or reset stuck | Check PLL inputs, reset polarity |
| `X or Z values persist` | Improper connections or floating nets | Verify top-level port mapping |
| `CLK unstable` | PLL not configured properly | Ensure `ENb_VCO` and `ENb_CP` are set correctly |

---

## 🧾 Summary

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


