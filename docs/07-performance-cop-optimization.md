# COP, Optimal Intermediate Pressure, and Design Optimization

*This document derives the coefficient of performance of the two-stage R718 cycle, benchmarks it against Carnot, explains quantitatively why two-stage + economizer beats single-stage at a 115 K lift, analyzes the optimal intermediate pressure, tabulates the sensitivities that the interactive calculator ([`html/index.html`](../html/index.html)) exposes as sliders, and states the formal design-optimization problem over the task's class-2 variables. It feeds the supervisory control layer of [doc 09](09-control-design.md) and the sizing of [doc 10](10-prototype-design.md).*

---

## 7.1 COP derivation

Heating COP is delivered heat over compression power. Per unit main-stream flow $\dot m_e$, with $r = \dot m_{inj}/\dot m_e$, $w_s = \dot m_{spray}/\dot m_e$, $F = 1 + r + w_s$ and $\dot m_c = F \dot m_e$:

$$ \mathrm{COP}_{gas} = \frac{\dot Q_{cond}}{\dot W_1 + \dot W_2}
= \frac{F\,(h_5 - h_6)}{(h_3 - h_2) + F\,(h_5 - h_4)} \tag{7.1} $$

The *electrical* COP divides by the drive-train efficiency (motor × mechanical × VFD):

$$ \mathrm{COP}_{el} = \eta_{drive}\,\mathrm{COP}_{gas}, \qquad \eta_{drive} = \eta_{mech}\,\eta_{motor}\,\eta_{VFD} \approx 0.86\text{–}0.92 \tag{7.2} $$

Per the [README](../README.md) nomenclature, **quoted plant COP is $\mathrm{COP}_{el}$**; Eq. (7.1) is what the first-law balance checks. At the anchor point (worked state values from [doc 03](03-compressor-models.md)/[doc 05](05-valves-mixing-junctions.md): $\dot W_1 \approx 8.6$ kW, $\dot W_2 \approx 14$ kW gas, $\dot Q_{cond} = 50$ kW):

$$ \mathrm{COP}_{gas} \approx \frac{50}{22.6} \approx 2.2, \qquad
\mathrm{COP}_{el} \approx 0.88 \times 2.2 \approx 1.9 $$

Cooling and heating COP obey the steady-state identity (energy balance, Eq. 6.5 of [doc 06](06-cycle-coupling-steady-state.md)):

$$ \mathrm{COP}_{heat} = \mathrm{COP}_{cool} + 1 \tag{7.3} $$

Second-law (exergetic) efficiency:

$$ \eta_{II} = \frac{\mathrm{COP}_{el}}{\mathrm{COP}_{Carnot}} \tag{7.4} $$

## 7.2 Carnot benchmark and the sanity rule

Between the thermodynamic reservoirs $T_e = 283.15$ K and $T_c = 398.15$ K:

$$ \mathrm{COP}_{Carnot} = \frac{T_c}{T_c - T_e} = \frac{398.15}{115.0} = 3.46 \tag{7.5} $$

Well-executed vapor-compression plants reach 45–55 % of Carnot; hence the canonical target band **COP ≈ 1.6–1.9** ($\eta_{II} \approx 0.46$–0.55). Two bold sanity rules for every model, spreadsheet and simulation in this project:

> **Rule 1.** Any computed heating COP above 3.46 at the nominal lift is a bug — usually a sign error or a broken energy balance, never a discovery.
>
> **Rule 2.** Any COP quoted without stating its basis (gas vs electrical, Eq. 7.1 vs 7.2) is ambiguous by ±10 %; this project quotes electrical.

*(True reservoir temperatures — source water 20 °C, delivered water 120 °C — give a Lorenz-type bound slightly different from Eq. 7.5; the internal $T_e/T_c$ convention is kept because compressor lift, not water lift, is what the cycle actually pays for. The distinction is noted here once and not re-litigated.)*

## 7.3 Why two-stage + economizer at this lift

**Single-stage would be absurd — the numbers:** pressure ratio $\Pi = 232.2/1.228 = 189$. From [doc 03](03-compressor-models.md): clearance volumetric efficiency $\eta_{vol} = 1 - c(\Pi^{1/k} - 1)$ collapses to zero near $c = 0.02$ (no real machine); the isentropic discharge from 10 °C saturated vapor is $T_{dis,s} \approx 283 \times 189^{0.242} \approx 283\times 3.56 \approx 1008$ K ≈ **735 °C** ($\gg 600$ °C even before the $1/\eta_{is}$ inflation) — far beyond any material/lubrication limit.

**Splitting the ratio** ($\Pi_1 = \Pi_2 = 13.8$) tames per-stage discharge to ≈ 261 °C isentropic / ≈ 370–390 °C actual, and the economizer + spray chain then buys three separable effects:

| Effect | Mechanism | Magnitude at anchor |
|---|---|---|
| (a) Interstage desuperheating | Injection vapor $h_{10}$ (+ spray $h_7$) cools state 4 | Vapor alone: −30 K; with spray: to $T_{sat}+15$ K, enabling stage 2 at all |
| (b) Subcooling | $h_6 \to h_7$ raises refrigerating effect $(h_1 - h_{11})$ | $x_{11}$: 0.186 → ≈ 0.084; +12 % evaporator effect per kg |
| (c) Staged flow | Only $\dot m_e$ (not $\dot m_c$) is lifted through the full 189:1 | Stage-1 work applies to $1/F \approx 70$ % of condenser flow |

**Comparison table** (consistent anchor assumptions: $\eta_{is} = 0.70$ per stage, $\eta_{drive} = 0.88$, spray to $T_{sat}(P_{int})+15$ K where fitted):

| Cycle | Feasible? | $T_{dis}$ | COP$_{el}$ |
|---|---|---|---|
| Single-stage | No (η_vol ≈ 0, $T_{dis}$ ≈ 735 °C isentropic) | — | — |
| Two-stage, no economizer, no spray | Thermally marginal (stage-2 inlet ≈ 370 °C → $T_5$ ≈ 1000 °C-class) | ✗ | (1.7 on paper, unbuildable) |
| Two-stage + economizer injection only | Still marginal ($T_4 \approx 340$ °C) | ✗ | ≈ 1.8 on paper |
| **Two-stage + economizer + interstage spray** | **Yes** ($T_4 \approx 71$ °C, $T_5 \approx 480$–500 °C → manageable with condenser DSH zone) | ⚠ high but bounded | **≈ 1.9** |

The table's message: for R718 the economizer chain is not a few-percent COP tweak as in halocarbon practice — it is the difference between a buildable machine and an impossible one, and the *binding* consideration is discharge temperature, not efficiency. (The residual $T_5 \approx 480$ °C still demands the multi-zone condenser of [doc 04](04-heat-exchanger-models.md) and the high-$T_5$ override of [doc 09](09-control-design.md); a second spray stage upstream of the condenser is a design option if materials dictate.)

## 7.4 Optimal intermediate pressure

Three estimates in increasing order of truth:

1. **Equal pressure ratio (geometric mean)** — minimizes total *ideal-gas* work for equal, perfect stages:

$$ P_{int}^{gm} = \sqrt{P_1 P_4} = \sqrt{1.228 \times 232.2} = 16.9\ \text{kPa} \quad (T_{sat} = 56.4\ ^\circ\text{C}) \tag{7.6} $$

2. **Heat-pump-corrected heuristic** — accounts for the temperature asymmetry of the stages:

$$ P_{int}^{corr} = \sqrt{P_1 P_4}\,\sqrt{T_c/T_e} \approx 16.9 \times \sqrt{398/283} \approx 20\ \text{kPa} \quad (T_{sat} \approx 60\ ^\circ\text{C}) \tag{7.7} $$

3. **Numerical optimum** — sweep $P_{int}$ in the [doc 06](06-cycle-coupling-steady-state.md) solver at fixed everything-else and read off the COP maximum.

The sweep (reproducible live in the calculator) shows a **flat optimum**: COP varies by < 1 % across $P_{int} \approx 14$–24 kPa. Flatness is good news *and* the key design insight:

- Good news: the design $V_{disp,1}/V_{disp,2}$ ratio need not be perfect, and runtime drift of the emergent $P_{int}$ ([doc 06](06-cycle-coupling-steady-state.md) §6.4) costs little COP.
- Key insight: with COP flat, **the stage-2 discharge-temperature constraint, not COP, selects $P_{int}$** for R718. Raising $P_{int}$ raises $T_{sat}(P_{int})$, hence the sprayed state-4 temperature, hence $T_5$; lowering $P_{int}$ loads stage 1. The practical setpoint band **17–20 kPa** quoted in the README is the constrained optimum: as high as the $T_5 \le$ limit allows while keeping stage-1 ratio ≤ ~14 — a *constrained* optimum, found where the constraint binds rather than where $\partial\mathrm{COP}/\partial P_{int} = 0$.
- Interaction with spray: more spray (lower $\Delta T_{sh,4}$ target) relaxes the $T_5$ constraint and shifts the admissible $P_{int}$ band upward slightly; the joint choice $(P_{int}, \Delta T_{sh,4})$ is a 2-D constrained problem the supervisory layer of [doc 09](09-control-design.md) solves by lookup table.

## 7.5 Sensitivities at the nominal point

Computed by central differences around the anchor with the design-mode march (the same numbers the calculator sliders reproduce):

| Perturbation | Effect on COP$_{el}$ | Comment / control relevance |
|---|---|---|
| $T_e$ +1 K | **+2.5–3 %** | Strongest lever; motivates source-pump optimization and generous evaporator area |
| $T_c$ −1 K | **+2 %** | Bounded below by pinch (Eq. 4.11); condenser area buys COP directly |
| $\eta_{is}$ +0.05 (both stages) | **+7 %** | Dominant hardware sensitivity; justifies premium compressors |
| $\Delta T_{ec}$ −2 K | +0.5 % | Mild; economizer area is cheap COP but the effect saturates |
| $\varepsilon_{rec}$ +0.15 | +0.3 % COP, **+15–20 K on $T_3$** | Efficiency-neutral, thermally expensive — see doc 04 §4.5 trade-off |
| Superheat $\Delta T_{sh}$ +2 K | −0.3 %, +4 K on $T_3$ | Keep minimal subject to droplet protection |
| $P_{int}$ ±3 kPa | < ±1 % | Flat optimum (§7.4); constraint-driven choice |
| $\eta_{drive}$ +0.02 | +2.3 % | Direct multiplier (Eq. 7.2) |

Control take-aways ([doc 09](09-control-design.md) §9.5 uses these): the supervisory layer chases $T_e$ (via source flow) and $T_c$ (via demanded lift) because those gradients dwarf the $P_{int}$ gradient; the regulatory layer's job is mostly to *hold constraints* ($T_5$, SH) at negligible COP cost.

## 7.6 Formal design optimization (class-2 variables)

$$
\begin{aligned}
\max_{x_{design}}\quad & \mathrm{COP}_{el}(x_{design};\ d^{nom}) \\
x_{design} = \ & \big[\,UA_{evap}, UA_{dsh}, UA_{cond}, UA_{ec}, \varepsilon_{rec},\ V_{disp,1}/V_{disp,2},\ K_{v,mv}, K_{v,iv},\ \dot m_{spray}^{max}\,\big] \\
\text{s.t.}\quad
& T_{proc,out} \ge 120\ ^\circ\text{C} \quad \text{(supply spec)} \\
& T_5 \le T_{dis}^{max},\qquad T_3 \le T_{dis}^{max} \\
& \Delta T_{pinch,k} \ge \Delta T_{min}\ \ \forall k \quad \text{(every exchanger)} \\
& \eta_{vol,i} \ge 0.3,\qquad \text{surge margin} \ge 10\ \% \\
& \textstyle\sum_k c_k A_k \le \text{Budget},\qquad P_1 \ge 0.9\ \text{kPa} \\
\end{aligned} \tag{7.8}
$$

Properties of (7.8): ~9 variables, smooth objective (the design-mode march is $C^1$ in all inputs away from zone-boundary switches), inexpensive evaluations (milliseconds) — **SQP or even a coarse grid + local refinement solves it**; global sophistication is unnecessary except for one caution: the flat $P_{int}$ direction (§7.4) plus the active $T_5$ constraint create a curved valley, so **multi-start** (3–5 starts across the $V_{disp}$-ratio axis) is cheap insurance against sliding to the valley's wrong end. Constraint qualification holds everywhere physical; report active sets with the optimum — at this design the binding set is $\{T_5, \Delta T_{pinch,cond}, \text{Budget}\}$, which is itself the design story in one line: *discharge temperature and condenser pinch, not efficiency curvature, shape the machine.*

## 7.7 Off-design performance maps

With the design frozen, sweep the [doc 06](06-cycle-coupling-steady-state.md) solver over the disturbance box:

- $T_{src,in} \in \{15, 20, 25\}$ °C × load $\in \{30, 50, 70, 100\}$ % → matrices of COP, $T_5$, $P_{int}$, SH margins.

Uses: (i) the **supervisory setpoint tables** of [doc 09](09-control-design.md) ($P_{int}$ setpoint, spray target, source-pump speed vs conditions); (ii) **commissioning acceptance** — the Phase-2 test matrix of [doc 10](10-prototype-design.md) §10.9 measures exactly this map and compares model vs plant within a stated tolerance band (±8 % COP, ±5 K discharge); (iii) **envelope charting** — the map's infeasible corner (low source temperature × high load, where $T_5$ or freeze limits bind) becomes the published operating envelope of the prototype.
