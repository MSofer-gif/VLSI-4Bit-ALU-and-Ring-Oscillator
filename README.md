# VLSI Design: 4-Bit ALU & Ring Oscillator in 45nm

## 📌 Project Overview
[cite_start]This repository contains the physical and logical design of a 4-bit Arithmetic Logic Unit (ALU) and a 2.5MHz Ring Oscillator, developed as the final project for the "Introduction to VLSI" course (Autumn 2025)[cite: 688, 691]. [cite_start]The project covers the full custom IC design flow, from gate-level schematic design to layout placement, routing, and post-layout verification (DRC & LVS)[cite: 701, 783].

### 🛠️ Tools & Technologies
* [cite_start]**EDA Tool:** Cadence Virtuoso [cite: 701]
* [cite_start]**Technology Node:** 45nm Standard Cell Library (`gsclib045`) [cite: 701]
* [cite_start]**Verification:** Cadence DRC & LVS tools, Exhaustive Behavioral Simulation [cite: 348, 524, 525]
* [cite_start]**Representation:** Two's Complement Signed Arithmetic [cite: 709]

---

## 💻 Part 1: 4-Bit ALU Design

### 🎯 Requirements
[cite_start]The ALU is required to compute the function $Y = A + B - C$[cite: 48, 711, 712].
* [cite_start]**Inputs:** Three 4-bit signed integers ($A$, $B$, $C$) registered via D-Flip-Flops[cite: 49, 706, 707].
* [cite_start]**Internal Node:** A 5-bit register $X$ storing the intermediate sum $X = A + B$[cite: 52, 713].
* [cite_start]**Outputs:** A registered output $Y$ representing the final result[cite: 50, 708].
* [cite_start]**Constraints:** Nominal supply voltage of 1.2V[cite: 714].

### 🏗️ Architecture: The Kogge-Stone Adder
[cite_start]Based on our project specifications, the core arithmetic was implemented using a **Kogge-Stone Adder (KSA)**[cite: 64, 753]. [cite_start]We chose this parallel prefix architecture due to its $O(\log_2 N)$ latency and constant fan-out of 2[cite: 66, 69]. [cite_start]While KSA requires more wiring tracks and cells compared to Brent-Kung, it guarantees minimal delay and uniform signal driving capabilities[cite: 71, 72, 73].

![Block Diagram](./images/block_diagram.png) 
*(Note to Shachar: Insert the block diagram from Page 3 showing the registers and the +/- blocks here)*

### 🧠 Data Path, Signed Arithmetic, and the Borrow Flag ($C_{out}$)
Handling Two's Complement subtraction optimally requires careful interpretation of the output flags. [cite_start]The subtraction $X - C$ is mathematically implemented as $Y = X + \sim C + 1$[cite: 125, 126]. 

[cite_start]Our hardware produces a 5-bit data bus $Y[4:0]$ and a Carry-Out flag ($C_{out}$) from the final prefix adder[cite: 291]. [cite_start]We intentionally applied a logical Inverter (NOT gate) to the final carry-out bit, transforming $C_{out}$ into an **active-high Borrow indicator**[cite: 293, 294]:

* [cite_start]**Negative Result ($C_{out} = 1$):** Indicates $X < C$ (a borrow was required)[cite: 297, 298]. [cite_start]The internal adder did *not* produce a carry, which we invert to '1'[cite: 297]. [cite_start]The value on $Y[4:0]$ is the precise negative integer in Two's Complement[cite: 300, 303].
    * *Simulation Proof:* For $(-5) + (-4) - (-1) = -8$, our ALU successfully asserts $C_{out} = 1$ and outputs $Y = 11000$ (which is -8 in 5-bit Two's complement)[cite: 329, 331, 332].
* [cite_start]**Positive/Zero Result ($C_{out} = 0$):** Indicates $X \ge C$ (no borrow required)[cite: 295, 296]. [cite_start]The result is positive, and the 5th bit (MSB) acts merely as a residual internal carry that can be discarded[cite: 343, 344]. [cite_start]The absolute magnitude is read directly from the lower 4 bits[cite: 344].
    * *Simulation Proof:* For $(6) + (-5) - (1) = 0$, our ALU outputs $C_{out} = 0$ and $Y = 10000$. Discarding the MSB leaves `0000`, correctly evaluating to 0[cite: 341, 342, 343, 344].

### ⚙️ Hardware Optimization (Dead Logic Elimination)
To optimize area and delay, we did not treat the subtractor's $C_{in}$ as a variable input. Since $C_{in}$ is constantly tied to Logic '1' ($V_{DD}$), we optimized the first stage of the subtractor's prefix tree[cite: 166, 167]:
1.  **Generate Logic:** $G_{new} = G_0 + (P_0 \cdot C_{in})$ simplified to $G_0 + P_0$, eliminating an AND gate[cite: 171, 172, 173].
2.  **Sum Logic:** $Y_0 = P_0 \oplus C_{in}$ simplified to $NOT(P_0)$, replacing an XOR gate with a smaller Inverter[cite: 175, 176].
3.  **Black to Grey Cell:** We converted the first stage's Black Cell into a Grey Cell by removing dead Propagate calculation logic entirely[cite: 178].

![Optimized Subtractor](./images/optimized_subtractor.png)
*(Note to Shachar: Insert the hand-drawn schematic of the optimized subtractor from Page 10 here)*

### 📐 Layout & Physical Verification
The final layout was structured hierarchically utilizing standard cells (`gsclib045`). The design passed all manufacturing rules and connectivity checks.
* [cite_start]**Estimated Area:** $345.394 \mu m^2$[cite: 237].
* [cite_start]**DRC:** 0 Errors [cite: 783]
* **LVS:** Clean match between schematic and layout[cite: 526, 783].

![ALU Layout](./images/alu_layout.png)
*(Note to Shachar: Insert the Virtuoso layout screenshot from Page 31 here)*
![DRC & LVS](./images/drc_lvs_clean.png)
*(Note to Shachar: Insert the green tick DRC/LVS screenshots from Pages 31-32 here)*

---

## 🔄 Part 2: Ring Oscillator Design

### 🎯 Requirements
Design a clock generation circuit using a Ring Oscillator topology to achieve an oscillation frequency of $f_{osc} = 2.5MHz$[cite: 876, 887]. The design must account for added RC chains and a $100fF$ load capacitor representing a clock network[cite: 899, 903].

### 🔬 Implementation & Tuning
The fundamental oscillator frequency is defined by $f_{osc} = \frac{1}{2 \cdot t_d \cdot n}$[cite: 890]. 
1.  **Initial Design:** We built a 5-stage inverter chain using `INVX1` cells[cite: 612]. By tuning the RC chains, we achieved a frequency of $2.492MHz$[cite: 622, 632].
2.  **Load Testing:** Adding the $100fF$ load capacitor dropped the frequency to $2.44MHz$, attenuated the signal edges, and prevented the signal from reaching full rail-to-rail swings (0V to 1.2V)[cite: 647, 648, 649, 650]. This indicated the `INVX1` drive strength was insufficient for the capacitive load[cite: 631, 632].
3.  **Optimization:** Upgrading the cells to `INVX2` significantly increased drive strength[cite: 670]. This restored sharp edges, returned the signal to full rail-to-rail capacity, and successfully elevated the frequency back up to $2.73MHz$[cite: 671, 679].

![Oscillator Output](./images/oscillator_waveform.png)
*(Note to Shachar: Insert the transient response waveform from Page 42 showing the 2.736MHz frequency here)*
