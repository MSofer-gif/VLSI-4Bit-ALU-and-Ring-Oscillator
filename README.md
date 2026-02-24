# VLSI Design: 4-Bit ALU & Ring Oscillator in 45nm

## 📌 Project Overview
This repository contains the physical and logical design of a 4-bit Arithmetic Logic Unit (ALU) and a 2.5MHz Ring Oscillator, developed as the final project for the "Introduction to VLSI" course (Autumn 2025). The project covers the full custom IC design flow, from gate-level schematic design to layout placement, routing, and post-layout verification (DRC & LVS).

### 🛠️ Tools & Technologies
* **EDA Tool:** Cadence Virtuoso
* **Technology Node:** 45nm Standard Cell Library (`gsclib045`)
* **Verification:** Cadence DRC & LVS tools, Exhaustive Behavioral Simulation
* **Representation:** Two's Complement Signed Arithmetic

---

## 💻 Part 1: 4-Bit ALU Design

### 🎯 Requirements
The ALU is required to compute the function $Y = A + B - C$.
* **Inputs:** Three 4-bit signed integers ($A$, $B$, $C$) registered via D-Flip-Flops.
* **Internal Node:** A 5-bit register $X$ storing the intermediate sum $X = A + B$.
* **Outputs:** A registered output $Y$ representing the final result.
* **Constraints:** Nominal supply voltage of 1.2V.

### 🏗️ Architecture: The Kogge-Stone Adder
Based on our project specifications, the core arithmetic was implemented using a **Kogge-Stone Adder (KSA)**. We chose this parallel prefix architecture due to its $O(\log_2 N)$ latency and constant fan-out of 2. While KSA requires more wiring tracks and cells compared to Brent-Kung, it guarantees minimal delay and uniform signal driving capabilities.

![Block Diagram](./images/block_diagram.png) 
*(Note: Insert the block diagram from Page 3 showing the registers and the +/- blocks here)*

### 🧠 Data Path, Signed Arithmetic, and the Borrow Flag ($C_{out}$)
Handling Two's Complement subtraction optimally requires careful interpretation of the output flags. The subtraction $X - C$ is mathematically implemented as $Y = X + \sim C + 1$. 

Our hardware produces a 5-bit data bus $Y[4:0]$ and a Carry-Out flag ($C_{out}$) from the final prefix adder. We intentionally applied a logical Inverter (NOT gate) to the final carry-out bit, transforming $C_{out}$ into an **active-high Borrow indicator**:

* **Negative Result ($C_{out} = 1$):** Indicates $X < C$ (a borrow was required). The internal adder did *not* produce a carry, which we invert to '1'. The value on $Y[4:0]$ is the precise negative integer in Two's Complement.
    * *Simulation Proof:* For $(-5) + (-4) - (-1) = -8$, our ALU successfully asserts $C_{out} = 1$ and outputs $Y = 11000$ (which is -8 in 5-bit Two's complement).
* **Positive/Zero Result ($C_{out} = 0$):** Indicates $X \ge C$ (no borrow required). The result is positive, and the 5th bit (MSB) acts merely as a residual internal carry that can be discarded. The absolute magnitude is read directly from the lower 4 bits.
    * *Simulation Proof:* For $(6) + (-5) - (1) = 0$, our ALU outputs $C_{out} = 0$ and $Y = 10000$. Discarding the MSB leaves `0000`, correctly evaluating to 0.

### ⚙️ Hardware Optimization (Dead Logic Elimination)
To optimize area and delay, we did not treat the subtractor's $C_{in}$ as a variable input. Since $C_{in}$ is constantly tied to Logic '1' ($V_{DD}$), we optimized the first stage of the subtractor's prefix tree:
1. **Generate Logic:** $G_{new} = G_0 + (P_0 \cdot C_{in})$ simplified to $G_0 + P_0$, eliminating an AND gate.
2. **Sum Logic:** $Y_0 = P_0 \oplus C_{in}$ simplified to $NOT(P_0)$, replacing an XOR gate with a smaller Inverter.
3. **Black to Grey Cell:** We converted the first stage's Black Cell into a Grey Cell by removing dead Propagate calculation logic entirely.

![Optimized Subtractor](./images/optimized_subtractor.png)
*(Note: Insert the hand-drawn schematic of the optimized subtractor from Page 10 here)*

### 📐 Layout & Physical Verification
The final layout was structured hierarchically utilizing standard cells (`gsclib045`). The design passed all manufacturing rules and connectivity checks.
* **Estimated Area:** $345.394 \mu m^2$.
* **DRC:** 0 Errors.
* **LVS:** Clean match between schematic and layout.

![ALU Layout](./images/alu_layout.png)
*(Note: Insert the Virtuoso layout screenshot from Page 31 here)*

![DRC & LVS](./images/drc_lvs_clean.png)
*(Note: Insert the green tick DRC/LVS screenshots from Pages 31-32 here)*

---

## 🔄 Part 2: Ring Oscillator Design

### 🎯 Requirements
Design a clock generation circuit using a Ring Oscillator topology to achieve an oscillation frequency of $f_{osc} = 2.5MHz$. The design must account for added RC chains and a $100fF$ load capacitor representing a clock network.

### 🔬 Implementation & Tuning
The fundamental oscillator frequency is defined by $f_{osc} = \frac{1}{2 \cdot t_d \cdot n}$. 

1. **Initial Design:** We built a 5-stage inverter chain using `INVX1` cells. By tuning the RC chains, we achieved a frequency of $2.492MHz$.
2. **Load Testing:** Adding the $100fF$ load capacitor dropped the frequency to $2.44MHz$, attenuated the signal edges, and prevented the signal from reaching full rail-to-rail swings (0V to 1.2V). This indicated the `INVX1` drive strength was insufficient for the capacitive load.
3. **Optimization:** Upgrading the cells to `INVX2` significantly increased drive strength. This restored sharp edges, returned the signal to full rail-to-rail capacity, and successfully elevated the frequency back up to $2.73MHz$.

![Oscillator Output](./images/oscillator_waveform.png)
*(Note: Insert the transient response waveform from Page 42 showing the 2.736MHz frequency here)*
