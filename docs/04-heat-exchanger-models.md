# Heat Exchanger Models: Evaporator, Condenser, Economizer, Recuperator

*This document derives the steady-state models of the four heat exchangers — evaporator (蒸发器), condenser (冷凝器), economizer (经济器) and recuperator (回热器) — as mass/energy balances plus rate equations with heat-transfer constitutive relations (传热系数关联式). It supplies the closure equations consumed by the system assembly in [doc 06](06-cycle-coupling-steady-state.md) and the UA/geometry bridge used by the design optimization in [doc 07](07-performance-cop-optimization.md). State numbering and anchor values follow the [README](../README.md) and [doc 01](01-system-overview.md).*

---

## 4.1 Common framework

Every heat exchanger in the plant is modeled with the same three ingredients:

1. **Stream energy balances.** For a stream entering at $h_{in}$ and leaving at $h_{out}$ with mass flow $\dot m$:

$$ \dot Q = \dot m\,(h_{out} - h_{in}) \tag{4.1} $$

2. **A rate equation** connecting $\dot Q$ to the driving temperature difference through the conductance $UA$.

3. **A constitutive layer** (Section 4.6) turning geometry and flow conditions into the film coefficients inside $UA$.

### 4.1.1 LMTD form

For counterflow with constant-property streams,

$$ \dot Q = UA\,\Delta T_{lm},\qquad
\Delta T_{lm} = \frac{\Delta T_a - \Delta T_b}{\ln(\Delta T_a/\Delta T_b)} \tag{4.2} $$

with $\Delta T_a,\ \Delta T_b$ the terminal temperature differences. The LMTD form is compact for *design* (area from known duties) but hostile to *simulation*: during Newton iteration a trial state can make $\Delta T_a/\Delta T_b \le 0$ and the logarithm undefined.

### 4.1.2 ε-NTU form (preferred)

$$ \dot Q = \varepsilon\, C_{min}\,(T_{h,in} - T_{c,in}), \qquad
NTU = \frac{UA}{C_{min}}, \qquad C = \dot m c_p \tag{4.3} $$

For counterflow with capacity ratio $C_r = C_{min}/C_{max}$:

$$ \varepsilon = \frac{1 - \exp[-NTU(1 - C_r)]}{1 - C_r \exp[-NTU(1 - C_r)]} \tag{4.4} $$

and for any exchanger in which one side changes phase ($C_{max}\to\infty$, $C_r = 0$):

$$ \varepsilon = 1 - e^{-NTU} \tag{4.5} $$

The ε-NTU form is bounded ($0 \le \varepsilon < 1$) for every physically admissible input, which is why the system solver of [doc 06](06-cycle-coupling-steady-state.md) and the transient model of [doc 08](08-dynamic-model.md) use it exclusively. **Design rule adopted here: rate equations are written in ε-NTU; LMTD appears only in sizing spot-checks in [doc 10](10-prototype-design.md).**

---

## 4.2 Condenser (冷凝器) — a mandatory multi-zone model

### 4.2.1 Why one zone is wrong for R718

The vapor entering the condenser (state 5) is not near saturation — it is superheated by hundreds of kelvin. At the nominal point with interstage spray desuperheating the HP discharge arrives at roughly 480 °C against a condensing temperature of 125 °C ([doc 03](03-compressor-models.md), §3.5). The desuperheating duty per kilogram,

$$ \Delta h_{dsh} = h_5 - h_g(T_c) \approx 3455 - 2713 \approx 742\ \text{kJ/kg}, $$

is **≈ 25 % of the total condenser duty** $h_5 - h_6 \approx 2930$ kJ/kg — the top of the canonical 15–25 % band — and it is transferred at a *gas-phase* heat-transfer coefficient roughly an order of magnitude below the film-condensation coefficient. A single-zone model with one $UA$ therefore misallocates a quarter of the area and misplaces the pinch. The model must resolve at least three zones:

| Zone | Refrigerant path | Duty (nominal) | Dominant HTC |
|------|------------------|----------------|--------------|
| DSH (desuperheat) | $h_5 \to h_g(T_c)$, vapor cools ≈ 485 → 125 °C | ~25 % | $\alpha_v \approx 0.1\text{–}0.5$ kW/m²K |
| COND (condensing) | $h_g(T_c) \to h_f(T_c)$ at $T_c$ | ~75–80 % | $\alpha_{film} \approx 5\text{–}10$ kW/m²K |
| SC (subcool) | $h_f(T_c) \to h_6$ (small or zero) | 0–5 % | $\alpha_l \approx 1\text{–}3$ kW/m²K |

### 4.2.2 Zone equations

Let the process water (m$_{proc}$, inlet $T_{proc,in} = 115$ °C) traverse SC → COND → DSH in counterflow. With $\dot Q_i$ the zone duties and $T_{w,i}$ the intermediate water temperatures:

$$ \dot Q_{dsh} = \dot m_c\,(h_5 - h_g(T_c)) = \varepsilon_{dsh}\,C_{min,dsh}\,(T_5 - T_{w,2}) \tag{4.6} $$

$$ \dot Q_{cond} = \dot m_c\,(h_g(T_c) - h_f(T_c)) = \left(1 - e^{-UA_{cond}/C_{proc}}\right) C_{proc}\,(T_c - T_{w,1}) \tag{4.7} $$

$$ \dot Q_{sc} = \dot m_c\,(h_f(T_c) - h_6) = \varepsilon_{sc}\,C_{min,sc}\,(T_c - T_{proc,in}) \tag{4.8} $$

$$ C_{proc}\,(T_{w,1} - T_{proc,in}) = \dot Q_{sc},\qquad
C_{proc}\,(T_{w,2} - T_{w,1}) = \dot Q_{cond},\qquad
C_{proc}\,(T_{proc,out} - T_{w,2}) = \dot Q_{dsh} \tag{4.9} $$

with the water-side energy balance closing the set:

$$ \dot Q_{cond,tot} = \dot m_{proc}\, c_{p,w} \,(T_{proc,out} - T_{proc,in}) = \dot m_{proc}\, c_{p,w}\,(120 - 115) \approx 50\ \text{kW} \tag{4.10} $$

which fixes $\dot m_{proc} = 50/(4.19 \times 5) \approx 2.39$ kg/s.

### 4.2.3 Pinch analysis: why $T_c = 125$ °C and not 120 °C

The process water leaves at 120 °C. In the condensing zone the refrigerant is isothermal at $T_c$; the tightest approach (the pinch) occurs where the water is hottest against the condensing plateau — at the COND/DSH boundary, where the water is already close to $T_{proc,out}$. Requiring a workable pinch $\Delta T_{pinch} \ge 3\text{–}5$ K there gives

$$ T_c \ge T_{proc,out} + \Delta T_{pinch} \approx 120 + (3\text{–}5)\ \Rightarrow\ T_c \approx 123\text{–}127\ ^\circ\text{C} \tag{4.11} $$

The canonical anchor $T_c = 125$ °C ($P_4 = 232.2$ kPa) sits in the middle of this band. Choosing $T_c = 120$ °C would demand infinite area; every extra kelvin of $T_c$ above the bound costs ≈ 2 %/K of COP ([doc 07](07-performance-cop-optimization.md)), so condenser area and COP trade directly through Eq. (4.11).

The desuperheat zone *does* deliver heat above $T_c$ (vapor enters at ~480 °C), which slightly relieves the pinch: in a counterflow arrangement the final water heating from $T_{w,2}$ to 120 °C can be done by the superheated gas even if $T_c$ were marginally below 123 °C. A conservative design does not rely on this — gas-side HTC is poor and the DSH zone area would balloon — but the effect is real and the multi-zone model captures it automatically.

---

## 4.3 Evaporator (蒸发器) — falling-film under deep vacuum

### 4.3.1 Configuration choice

At $P_1 = 1.228$ kPa the working pressure is ~1 % of atmospheric. A **flooded pool** evaporator is ruled out by hydrostatics: a liquid column of only 0.1 m adds

$$ \Delta P_{hyd} = \rho g H \approx 1000 \times 9.81 \times 0.1 \approx 1\ \text{kPa} \approx 0.8\,P_1 \tag{4.12} $$

— the saturation temperature at the bottom of a 10 cm pool is ≈ 7 K above that at the surface ($dP_{sat}/dT \approx 82$ Pa/K, [doc 02](02-water-properties.md)), suppressing boiling in exactly the region where the tubes are. The adopted configuration is therefore a **falling-film (or spray-fed) horizontal-tube shell-and-tube evaporator**: recirculated liquid is distributed as a thin film over tubes carrying the source water; evaporation occurs from the film surface at (essentially) the uniform shell pressure $P_1$.

### 4.3.2 Model equations

Refrigerant side is isothermal at $T_e = T_{sat}(P_1)$, so Eq. (4.5) applies on the source-water side ($C_{min} = \dot m_{src} c_{p,w}$):

$$ \dot Q_{evap} = \dot m_e\,(h_1 - h_{11}) \tag{4.13} $$

$$ \dot Q_{evap} = \left(1 - e^{-UA_{evap}/(\dot m_{src} c_{p,w})}\right)\,\dot m_{src}\, c_{p,w}\,(T_{src,in} - T_e) \tag{4.14} $$

$$ \dot Q_{evap} = \dot m_{src}\, c_{p,w}\,(T_{src,in} - T_{src,out}) \tag{4.15} $$

At the nominal point ($\dot Q_{evap} \approx 27$ kW, $T_{src,in} = 20$ °C, $T_e = 10$ °C, source outlet ≈ 15 °C): $\dot m_{src} = 27/(4.19\times5) \approx 1.3$ kg/s and the required effectiveness is $\varepsilon = 27/(1.3\cdot4.19\cdot10) \approx 0.5$, i.e. $NTU \approx 0.69$, $UA_{evap} \approx 3.8$ kW/K — comfortably achievable with $A \approx 3\text{–}5$ m² at $U \approx 1\text{–}1.5$ kW/m²K (film-side coefficients at 1 kPa are modest; see §4.6.3).

### 4.3.3 Vapor-side integrity

Two practical constraints belong in the model as design inequalities:

- **Carryover / demister.** The huge outlet volume flow (~1.3 m³/s at $v_g \approx 106$ m³/kg) means droplet entrainment at even modest shell velocities; a demister with a vapor-approach velocity limit $u \le 3$–5 m/s sets the minimum shell cross-section.
- **Vapor-side pressure drop is temperature.** Every 100 Pa lost between film surface and compressor suction costs ≈ 1.2 K of effective lift ([doc 02](02-water-properties.md), §2.3). The shell headspace and suction duct must be sized so that $\Delta P_{shell+duct} \lesssim 100$ Pa; this constraint reappears in the pipe sizing of [doc 10](10-prototype-design.md).

---

## 4.4 Economizer (经济器) — heat-exchanger type with vapor injection

### 4.4.1 Balances

Per the task sheet the economizer is the *heat-exchanger* type (not a flash tank): the full condensate stream $\dot m_c$ passes the hot side from state 6 to state 7, while the injection branch — throttled to $P_{int}$ by the secondary valve (次膨胀阀), states 7 → 9 — evaporates on the cold side from state 9 to state 10:

$$ \dot m_c\,(h_6 - h_7) = \dot m_{inj}\,(h_{10} - h_9), \qquad h_9 = h_7 \tag{4.16} $$

(The branch taps at state 7, so $h_9 = h_7$ by the isenthalpic-valve model of [doc 05](05-valves-mixing-junctions.md).)

### 4.4.2 Closure and the injection ratio

The second relation closing the economizer is a thermal-contact statement. Two equivalent options:

- **Approach temperature** (used as the design closure):

$$ T_7 = T_{sat}(P_{int}) + \Delta T_{ec}, \qquad \Delta T_{ec} \approx 3\text{–}8\ \text{K} \tag{4.17} $$

- **Effectiveness**: $\varepsilon_{ec} = (T_6 - T_7)/(T_6 - T_{sat}(P_{int}))$ with Eq. (4.5) on the boiling cold side.

With Eq. (4.17), and taking the cold side to leave as saturated vapor $h_{10} = h_g(P_{int})$, divide Eq. (4.16) by $\dot m_e$. Writing $\dot m_c = \dot m_e (1 + r)$ (no interstage spray for the moment) and $r = \dot m_{inj}/\dot m_e$:

$$ (1+r)(h_6 - h_7) = r\,(h_{10} - h_7)
\ \Longrightarrow\
r = \frac{h_6 - h_7}{h_{10} - h_6} \tag{4.18} $$

**Worked value.** At the anchor: $h_6 = h_f(125\,^\circ\text{C}) = 525.0$ kJ/kg; with $P_{int} = 17$ kPa ($T_{sat} = 56.4$ °C) and $\Delta T_{ec} = 5$ K, $h_7 = h_f(61.4\,^\circ\text{C}) \approx 257$ kJ/kg; $h_{10} = h_g(56.4\,^\circ\text{C}) \approx 2602$ kJ/kg:

$$ r = \frac{525 - 257}{2602 - 525} = \frac{268}{2077} \approx 0.13 $$

Tightening or relaxing $\Delta T_{ec}$ moves $r$ across the canonical band **$r \approx 0.11$–0.13** quoted in the [README](../README.md). When the interstage spray desuperheater is active the hot-side flow becomes $\dot m_e(1 + r + w_s)$ and $r$ couples to the spray ratio $w_s$; the simultaneous (still linear, still explicit) solution is derived in [doc 05](05-valves-mixing-junctions.md), §5.6, giving $r \approx 0.16$, $w_s \approx 0.26$ at the same anchor.

### 4.4.3 What the economizer buys

Two distinct benefits, quantified in [doc 07](07-performance-cop-optimization.md):

1. **Liquid subcooling** $h_6 \to h_7$ (525 → 257 kJ/kg) slashes the flash quality at the main valve: $x_{11}$ falls from ≈ 0.19 (throttling saturated liquid from $T_c$; 0.195 at the 125 °C anchor, canonical 0.186 at the 120 °C reference — [doc 05](05-valves-mixing-junctions.md) §5.2.1) to ≈ 0.09 — nearly all of the canonical $x_{11}: 0.186 \to 0.084$ improvement is the economizer's doing (the recuperator contributes the last few points, §4.5).
2. **Interstage vapor injection** at $h_{10}$ cools the HP suction mix — though only by ~30 K (from ≈ 370–390 °C to ≈ 340–360 °C), which is why the spray desuperheater of [doc 05](05-valves-mixing-junctions.md) remains practically mandatory.

---

## 4.5 Recuperator (回热器) — a double-edged sword for R718

### 4.5.1 Balances

The liquid branch splits at state 7 *before* the recuperator ([README](../README.md) convention), so **both recuperator sides carry the same flow $\dot m_e$**: hot side 7 → 8 (subcooled liquid), cold side 1 → 2 (low-pressure vapor):

$$ \dot m_e\,(h_7 - h_8) = \dot m_e\,(h_2 - h_1)
\quad\Longrightarrow\quad
h_7 - h_8 = h_2 - h_1 \equiv q_{rec} \tag{4.19} $$

With equal mass flows the capacity-rate ratio is set by the specific heats; $c_{p,v} \approx 1.9 < c_{p,l} \approx 4.2$ kJ/(kg·K), so the vapor side is $C_{min}$ and

$$ q_{rec} = \varepsilon_{rec}\; c_{p,v}\,\big(T_7 - T_1\big) \tag{4.20} $$

### 4.5.2 The trade-off, quantified

**Cost — discharge temperature.** Each kelvin of extra suction superheat is amplified through the LP stage by the isentropic temperature ratio $\Pi^{(k-1)/k} \approx 13.8^{0.242} \approx 1.9$ ([doc 03](03-compressor-models.md), §3.5): **+1 K at state 2 ⇒ ≈ +1.9 K at state 3**, in a machine already flirting with its ~250–300 °C material limit.

**Benefit — subcooling.** $h_8 = h_7 - q_{rec}$ enters the main valve colder, cutting flash quality (Eq. 5.3 of [doc 05](05-valves-mixing-junctions.md)) and raising the refrigerating effect per kilogram.

| $\varepsilon_{rec}$ | $T_2$ (°C) | $T_3$ (°C) | $x_{11}$ | $\dot Q_{evap}/\dot m_e$ (kJ/kg) |
|------|------|------|------|------|
| 0 (none) | 13 | ≈ 372 | 0.087 | 2266 |
| 0.15 (nominal) | ≈ 20 | ≈ 385 | 0.081 | 2280 |
| 0.35 | ≈ 30 | ≈ 405 | 0.077 | 2292 |
| 0.60 | ≈ 42 | ≈ 428 | 0.070 | 2308 |

(Rows computed at the anchor: $T_e = 10$ °C + 3 K superheat, $P_{int} = 17$ kPa, $\Delta T_{ec} = 5$ K, $\eta_{is} = 0.70$; the interactive calculator in [`html/index.html`](../html/index.html) reproduces this sweep live.)

The evaporator-effect gain is ~1–2 % while the discharge-temperature penalty is tens of kelvin. **Design recommendation: modest $\varepsilon_{rec} \approx 0.15$–0.3**, sized primarily to guarantee dry, mildly superheated suction (droplet protection for a high-tip-speed machine) rather than for cycle-efficiency gain. For R718 the recuperator is a *protective* component; the *thermodynamic* subcooling work is done by the economizer.

---

## 4.6 Heat-transfer correlation menu (传热系数关联式)

The rate equations need film coefficients. The constitutive menu below is the "insert here" layer the task assigns to component modeling; correlations are stated with validity ranges, and property values come from [doc 02](02-water-properties.md) §2.7.

### 4.6.1 Single-phase forced convection (tube side, both water loops and gas-phase zones)

**Gnielinski** (preferred, $3\times10^3 < Re < 5\times10^6$, $0.5 < Pr < 2000$):

$$ Nu = \frac{(f/8)(Re - 1000)\,Pr}{1 + 12.7\sqrt{f/8}\,(Pr^{2/3} - 1)}, \qquad f = (0.79 \ln Re - 1.64)^{-2} \tag{4.21} $$

**Dittus–Boelter** (quick estimates, fully turbulent): $Nu = 0.023\,Re^{0.8} Pr^{n}$, $n = 0.4$ heating / 0.3 cooling.

### 4.6.2 Film condensation (condenser COND zone, horizontal tube bank)

**Nusselt** theory with tube-row correction:

$$ \alpha_{film} = 0.729 \left[ \frac{\rho_l (\rho_l - \rho_v)\, g\, h_{fg}\, \lambda_l^3}{\mu_l\, (T_{sat} - T_{wall})\, D} \right]^{1/4} N_{row}^{-1/6} \tag{4.22} $$

At 125 °C water gives $\alpha_{film} \approx 6$–10 kW/m²K — the best coefficient in the plant, which is why the DSH zone (gas-phase, Eq. 4.21 at $\alpha \approx 0.1$–0.5 kW/m²K) dominates area allocation despite carrying a fifth of the duty.

### 4.6.3 Falling-film evaporation (evaporator, sub-atmospheric caveat)

For horizontal-tube falling films a standard engineering form is

$$ \alpha_{ff} = C\,\lambda_l \left( \frac{g}{\nu_l^2} \right)^{1/3} Re_\Gamma^{\,n},\qquad Re_\Gamma = \frac{4\Gamma}{\mu_l} \tag{4.23} $$

with $\Gamma$ the film flow per unit tube length and $C, n$ fitted per distributor design ($C \approx 0.01$–0.04, $n \approx 0.2$–0.4). **Caveat:** most published fits are for pressures ≥ 10 kPa. At 1.2 kPa the vapor density is ~0.0094 kg/m³, vapor shear reshapes the film, and nucleate-boiling contributions predicted by pool-boiling forms (Cooper, Gorenflo) **overpredict substantially** — reduced pressure $p_r = P_1/P_{crit} \approx 5.6\times10^{-5}$ lies far outside their validated ranges. The model therefore carries an explicit uncertainty band of ±30–50 % on $\alpha_{ff}$, to be collapsed by the Phase-2 commissioning data ([doc 10](10-prototype-design.md), §10.9).

### 4.6.4 Fouling

Design fouling resistances (TEMA-class): treated closed water loops $R_f \approx 1\times10^{-4}$ m²K/W per side; the refrigerant side is clean (deaerated demineralized water). Fouling enters Eq. (4.24) additively.

---

## 4.7 From geometry to UA — the class-2 design bridge

The task's class-2 (design-stage, 设计阶段可选定) variables — tube count $N_t$, diameter $D_i/D_o$, length $L$, pitch, pass count, fin geometry — enter the model **only** through the conductance:

$$ \frac{1}{UA} = \frac{1}{\eta_o\,\alpha_i A_i} + \frac{\ln(D_o/D_i)}{2\pi \lambda_{wall} N_t L} + \frac{1}{\alpha_o A_o} + R_{f,i} + R_{f,o} \tag{4.24} $$

with $A_i = \pi D_i N_t L$, $A_o = \pi D_o N_t L$ and $\eta_o$ the extended-surface efficiency (unity here; bare tubes). During design optimization ([doc 07](07-performance-cop-optimization.md)) the $UA$'s of the four exchangers are the decision variables under a total-area budget; after hardware freeze they become fixed parameters, and only the *flow-dependent* part of $\alpha(Re, Pr)$ moves in off-design simulation.

---

## 4.8 Summary boxes for the system assembly

Inputs marked ▸ are supplied by other components or the boundary; outputs ◂ feed [doc 06](06-cycle-coupling-steady-state.md).

**Evaporator** — ▸ $h_{11}, T_{src,in}, \dot m_{src}, UA_{evap}$; states: $P_1 = P_{sat}(T_e)$ — ◂ $h_1 = h_g(T_e) + c_{p,v}\Delta T_{sh}$, $\dot Q_{evap}$, $T_{src,out}$. Equations (4.13)–(4.15).

**Condenser** — ▸ $h_5, \dot m_c, T_{proc,in}, \dot m_{proc}, UA_{dsh|cond|sc}$; states: $P_4 = P_{sat}(T_c)$ — ◂ $h_6$, $T_c$, $T_{proc,out}$, $\dot Q_{cond}$. Equations (4.6)–(4.10); pinch inequality (4.11).

**Economizer** — ▸ $h_6, P_{int}, \Delta T_{ec}$ (or $UA_{ec}$) — ◂ $h_7$, $h_{10}$, $r$. Equations (4.16)–(4.18).

**Recuperator** — ▸ $h_1, h_7, \varepsilon_{rec}$ — ◂ $h_2 = h_1 + q_{rec}$, $h_8 = h_7 - q_{rec}$. Equations (4.19)–(4.20).

Every balance above is linear in enthalpies once temperatures are fixed, which is what keeps the sequential march of [doc 06](06-cycle-coupling-steady-state.md) explicit.
