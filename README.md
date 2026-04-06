# Two-Stage CMOS Operational Amplifier — gm/Id Design Methodology

Designed and simulated a Miller-compensated two-stage CMOS op-amp from scratch using the gm/Id methodology in Cadence Virtuoso. The entire design — transistor sizing, bias currents, and compensation network — was determined analytically from simulation-extracted lookup curves before running any transient or AC simulation.

---

## Specifications

| Parameter | Target | Achieved |
|---|---|---|
| DC Open-Loop Gain | ≥ 55 dB | 56.5 dB |
| Unity Gain Bandwidth | ~3.5 MHz | 3.507 MHz |
| Phase Margin | ≥ 60° | 67° |
| Slew Rate | ≥ 3 V/µs | ~3.3 V/µs |
| Input Offset Voltage | < 1 mV | ~180 µV |
| Load Capacitance | 5 pF | 5 pF |
| Supply Voltage | 1.2 V | 1.2 V |
| Technology | GPDK090 (90 nm CMOS) | GPDK090 |

---

## Circuit Overview

The topology is a classical two-stage Miller-compensated CMOS op-amp:

- **Stage 1** — Differential amplifier with PMOS current mirror load
- **Stage 2** — Common-source gain stage for additional voltage gain
- **Compensation** — Miller capacitor `Cc` with a series zero-cancellation resistor `Rz = 1/gm2`

![results/Stage 1 Differential Amplifier (1).png

---

## Why gm/Id?

The square-law model (`Id = 0.5 * µCox * W/L * Vov²`) breaks down below 180 nm due to velocity saturation and short-channel effects. The gm/Id methodology replaces it with simulation-extracted lookup curves that implicitly capture all of these effects. Every transistor operating point is characterized by the dimensionless transconductance efficiency `gm/Id` (V⁻¹), which encodes the inversion level:

- `gm/Id > 20 V⁻¹` → Weak inversion (maximum efficiency, very low speed)
- `gm/Id ≈ 10–20 V⁻¹` → Moderate inversion (balanced trade-off)
- `gm/Id < 10 V⁻¹` → Strong inversion (fastest, least efficient)

---

## Design Flow

### Step 1 — Extract Lookup Curves

Four curves are extracted from DC simulation by sweeping gate voltage at fixed drain voltage:

**gm/Id vs Overdrive Voltage**
Used to verify the transistor's inversion level and check headroom on a 1.2 V supply.

![gm/id vs gm/gds ]gm_id/gm_over_id_vs_voverdrive.jpg

**gm/gds vs gm/Id (Intrinsic Gain)**
Used to select channel length. Longer L increases gm/gds (more gain) but reduces fT (slower). L = 400–500 nm was chosen as the balance point.

![gm/gds vs gm/Id](gmovergdsvsgmid.jpg)

At gm/Id = 13 V⁻¹ and L = 400 nm → gm/gds ≈ 52, sufficient for the gain target.

**fT vs gm/Id (Speed)**
Used to confirm the second pole is not a limiting factor. At L = 400 nm, fT is in the GHz range — well above the 3.5 MHz UGB.

![fT vs gm/Id](gmidvsfug.jpg)

**Id/W vs gm/Id (Sizing)**
The transistor sizing tool. Once Id and gm/Id are known, width is simply `W = Id / (Id/W)`.

![Id/W vs gm/Id](gmidvsidbyw.jpg)

---

### Step 2 — Phase Margin → Compensation Capacitor Cc

For a two-pole system, phase margin is:

```
PM = 90° - arctan(ωu / p2)
```

For PM = 67°:
```
arctan(ωu / p2) = 23°  →  ωu / p2 = 0.424  →  p2 = 2.36 * ωu
```

Substituting `ωu = gm1 / Cc` and `p2 = gm2 / CL`, with `gm2 = 6 * gm1`:

```
Cc ≥ CL / (0.424 × 6) = 5 pF / 2.55 ≈ 1.97 pF
```

Initial value chosen: `Cc = 2.3 pF` (with margin). Reduced to `900 fF` post-optimization.

**Why gm2 = 6 × gm1?**
Not an assumption — derived from the stability constraint. Placing p2 at 2.4 × ωu requires `gm2/gm1 ≥ 2.4 × CL/Cc ≈ 5.3 ≈ 6`.

---

### Step 3 — Slew Rate → Tail Current

```
SR = I_tail / Cc  →  I_tail = SR × Cc = 3.3 V/µs × 0.9 pF ≈ 3 µA (minimum)
```

Post-optimization: `I_tail = 20 µA` to satisfy slew rate after Cc was reduced and gain was recovered.

---

### Step 4 — Deriving gm/Id (Not Assuming It)

This is the key step. `gm1` is set by the UGB requirement:

```
gm1 = 2π × UGB × Cc = 2π × 3.5 MHz × 2.3 pF ≈ 50 µS
```

With `ID1 = I_tail / 2 = 2.3 µA`:

```
gm/Id = gm1 / ID1 = 50 µS / 2.3 µA ≈ 21.7 V⁻¹
```

`gm/Id` is the **result** of the UGB and slew-rate constraints — not a starting assumption. Post-optimization this converged to ≈ 13 V⁻¹ in moderate inversion.

---

### Step 5 — Transistor Sizing from Id/W Curve

At `gm/Id = 13 V⁻¹`:
```
Id/W ≈ 5.88 µA/µm  →  W1 = 2.3 µA / 5.88 µA/µm ≈ 0.39 µm
```

Final widths after optimization are listed in the table below.

---

## Final Device Sizes

| Device | Type | W (nm) | L (nm) | Role |
|---|---|---|---|---|
| NM8 | NMOS | 9560 | 400 | Differential pair input |
| NM6 | NMOS | 300 | 550 | Diff pair load |
| NM1 | NMOS | 300 | 550 | Diff pair load |
| PM0 | PMOS | 600 | 400 | Current mirror load |
| PM1 | PMOS | 600 | 400 | Current mirror load |
| NM9 | NMOS | 2500 | 400 | Tail current source |
| PM2 | PMOS | 1800 | 600 | Stage 2 pull-up (m=4) |
| NM3 | NMOS | 12500 | 400 | Stage 2 pull-down |
| R1 | Poly | — | — | Rz = 4.1 kΩ (zero cancel) |
| Cc | MOM cap | — | — | ~900 fF |
| CL | Load | — | — | 5 pF |

---

## Simulation Results

### AC Response — Bode Plot

![AC Response](image__6_.png)

| Marker | Frequency | Value |
|---|---|---|
| M3 | 63.1 Hz | 56.57 dB (DC gain) |
| M4 (gain) | 3.507 MHz | ~0 dB (UGB) |
| M4 (phase) | 3.507 MHz | −112.5° → PM = **67.5°** |

### Transient Response — Unity Gain Buffer, 1 V Step

![Step Response](SLEWw.jpg)

- Settles to 999.8 mV at t = 239.7 ns
- No overshoot — consistent with PM > 60°
- Slew rate ≈ 3.3 V/µs

---

## Post-Simulation Optimization

The first iteration had two problems: slew rate ~0.6 V/µs (target: 3 V/µs) and gain ~54 dB (target: 55 dB). Three changes were made:

| Change | From | To | Why |
|---|---|---|---|
| Compensation cap Cc | 2.3 pF | 900 fF | Improves SR and UGB; PM stays > 60° |
| Tail current | 4.6 µA | 20 µA | Directly improves slew rate |
| Channel length | 400 nm | 500 nm | Recovers gain lost from higher current (larger ro) |

**Trade-offs consciously accepted:**
- Smaller Cc → better SR and BW, but PM dropped from ~75° to 67° (still passes spec)
- Higher current → better SR, but increases power and slightly lowers ro
- Longer L → better gain, but reduces fT (acceptable since fT >> UGB even at 500 nm)

---

## Functional Verification

The op-amp was tested in multiple closed-loop configurations:

- Unity gain buffer — step response, slew rate measurement
- Inverting amplifier — gain tracking up to ×10 ratio
- Non-inverting amplifier — correct closed-loop behavior verified
- Voltage comparator — switching verified for differential input > 1 mV
- Summing amplifier — superposition of two inputs verified

---

## Repository Structure

```
.
├── README.md
├── report/
│   └── two_stage_opamp_gmid_report.pdf       # Full design report (22 pages)
│   └── two_stage_opamp_gmid_report.tex       # LaTeX source
├── schematics/
│   └── Stage_1_Differential_Amplifier__1_.png
├── simulation_results/
│   ├── image__6_.png                         # AC Bode plot
│   ├── SLEWw.jpg                             # Transient step response
├── gmid_curves/
│   ├── gm_over_id_vs_voverdrive.jpg
│   ├── gmovergdsvsgmid.jpg
│   ├── gmidvsfug.jpg
│   └── gmidvsidbyw.jpg
```

---

## References

1. B. Razavi, *Design of Analog CMOS Integrated Circuits*, 2nd ed., McGraw-Hill, 2017.
2. N. Krishnapura, "Analog IC Design," NPTEL Video Lecture Series, IIT Madras. [nptel.ac.in/courses/117106030](https://nptel.ac.in/courses/117106030)
3. Algo-Aura, "gm/Id Methodology in Cadence Virtuoso," YouTube. [youtu.be/n9gD-A6FO3c](https://youtu.be/n9gD-A6FO3c?si=WzirX0JccUkEfswW)
4. F. Silveira, D. Flandre, P. G. A. Jespers, "A gm/ID based methodology for the design of CMOS analog circuits," *IEEE JSSC*, vol. 31, no. 9, pp. 1314–1319, Sep. 1996.

---

## Tools

- Cadence Virtuoso (Schematic Entry, ADE L/XL)
- GPDK090 — 90 nm CMOS Process Design Kit
- pdflatex — Report compilation

---

*Indian Institute of Information Technology, Design and Manufacturing Kurnool
Electronics and Communication Engineering*
