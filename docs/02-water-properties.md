# 02 — Thermophysical Properties of Water/Steam (R718)

*This document assembles every property relation the project uses for the working fluid (工质) water/steam: the IAPWS-IF97 saturation curve (Region 4) with worked evaluations at the cycle anchor points, the Region 1/2 Gibbs-energy formulations for liquid and vapor, the Clausius–Clapeyron slope that governs the vacuum-side pressure-drop penalty, ideal-gas shortcuts for hand calculation with quantified errors, and transport-property tables and fits. It feeds the compressor models of [doc 03](03-compressor-models.md), the heat-exchanger correlations of [doc 04](04-heat-exchanger-models.md), the throttling/mixing balances of [doc 05](05-valves-mixing-junctions.md), and the numerical cycle solver of [doc 06](06-cycle-coupling-steady-state.md).*

---

## 2.1 Why water is a peculiar refrigerant

Water (H₂O, refrigerant designation **R718**) has a molar mass of

$$
M = 18.015 \ \text{kg/kmol}, \qquad
R = \frac{\bar{R}}{M} = \frac{8.31446}{18.015} = 0.461526 \ \text{kJ/(kg·K)} ,
\tag{2.1}
$$

the IF97 specific gas constant. This is 5–8 times the specific gas constant of the halocarbon refrigerants (R134a: 0.0815 kJ/(kg·K)), and together with strong hydrogen bonding it produces an extreme property profile. Fixed points: triple point 0.01 °C / 611 Pa, critical point 373.95 °C / 22.064 MPa — the whole cycle of this project (1.228–232.2 kPa, 10–125 °C saturation) sits deep inside the subcritical vapor dome, far from the critical region.

Three features dominate the system design:

1. **Enormous latent heat, enormous vapor volume.** $h_{fg}(10\,^\circ\text{C}) = 2477\ \text{kJ/kg}$ means only $\dot m_c \approx 0.02$ kg/s circulates for 50 kW of heating — but the saturated-vapor specific volume at the evaporator (蒸发器) exit is $v_g(10\,^\circ\text{C}) = 106.3\ \text{m}^3/\text{kg}$, so that tiny mass flow is **≈ 1 m³/s (≈ 3 500 m³/h) of suction volume**. Water heat pumps are volume-flow machines, not mass-flow machines.

2. **A steep, strongly curved saturation line.** $P_{sat}$ grows by a factor of 189 from 10 °C to 125 °C. On the vacuum side the absolute slope is small (≈ 82 Pa/K at 10 °C), so modest suction-line pressure drops translate into large saturation-temperature penalties (§2.3).

3. **"Drying" isentropes.** In the $T$–$s$ plane the saturated-vapor line of water has $\mathrm{d}s_g/\mathrm{d}T < 0$ everywhere ($s_g$ = 8.900 kJ/(kg·K) at 10 °C but only 7.077 at 125 °C). An isentropic compression that starts at (or near) saturated vapor therefore moves **into** the superheat region and keeps moving away from the dome — the opposite of "wet-compression" or retrograde fluids (e.g. long-chain organics) whose compression paths can graze the dome. Two consequences: droplet impingement is *not* a compression concern (good), but discharge superheat is extreme — ≈ 261 °C isentropic and ≈ 370 °C at $\eta_{is}=0.70$ for the first stage alone (bad; this drives the interstage desuperheating architecture of [doc 05](05-valves-mixing-junctions.md)).

**Consequences table**

| Property of R718 | Value / behavior | System consequence |
|---|---|---|
| $R = 0.4615$ kJ/(kg·K), $k \approx 1.32$ (vapor) | high $kRT$ | Large temperature rise per pressure ratio → two stages + desuperheating mandatory |
| $h_{fg} \approx 2477 \to 2188$ kJ/kg (10 → 125 °C) | ~10× halocarbons | Very small mass flow (ṁ_c ≈ 0.02 kg/s @ 50 kW); tiny pipes on the liquid side |
| $v_1 = 106\ \text{m}^3/\text{kg}$ at 1.228 kPa | deep vacuum | Huge LP compressor displacement; suction-line sizing critical |
| $\mathrm{d}P_{sat}/\mathrm{d}T = 82$ Pa/K at 10 °C | flat curve at vacuum | 100 Pa suction ΔP ≈ 1.2 K lost lift; air in-leakage degrades vacuum |
| $\mathrm{d}s_g/\mathrm{d}T < 0$ (drying isentropes) | compression → superheat | 250–370 °C discharge; oil-free or high-temperature lubrication |
| Triple point 611 Pa / 0.01 °C | hard lower bound | Evaporation below ~0 °C impossible; freeze-up interlock ([doc 09](09-control-design.md)) |
| Critical point 22.064 MPa / 373.95 °C | far above cycle | Entire cycle comfortably subcritical; no near-critical property pathologies |
| ODP = 0, GWP ≈ 0, non-flammable, free | — | Ideal environmental/safety profile; the whole case for R718 high-temperature heat pumps |

---

## 2.2 Saturation curve — IAPWS-IF97 Region 4

The project's canonical $P$–$T$ anchor pairs — $T_e = 10\,^\circ\text{C} \leftrightarrow P_1 = 1.228$ kPa and $T_c = 125\,^\circ\text{C} \leftrightarrow P_4 = 232.2$ kPa — come from the IAPWS-IF97 (2007) Region 4 saturation equation, which is exact enough (reproduces IAPWS-95 within ±0.025 %) to serve as the single source of truth for every document.

### 2.2.1 Formulation

Region 4 is a single implicit quadratic that is *simultaneously* quadratic in a transformed pressure $\beta$ and a transformed temperature $\vartheta$:

$$
\beta^2\vartheta^2 + n_1\beta^2\vartheta + n_2\beta^2 + n_3\beta\vartheta^2 + n_4\beta\vartheta + n_5\beta + n_6\vartheta^2 + n_7\vartheta + n_8 = 0 ,
\tag{2.2}
$$

with the dimensionless variables

$$
\beta = \left(\frac{P_{sat}}{P^*}\right)^{1/4}, \qquad
\vartheta = \frac{T_{sat}}{T^*} + \frac{n_9}{T_{sat}/T^* - n_{10}}, \qquad
P^* = 1\ \text{MPa}, \quad T^* = 1\ \text{K}.
\tag{2.3}
$$

Because (2.2) is quadratic in $\beta^2$-groupings either variable can be isolated in closed form — no iteration is ever needed for saturation properties. The ten coefficients:

| $i$ | $n_i$ | $i$ | $n_i$ |
|---|---|---|---|
| 1 | 0.116 705 214 527 67 × 10⁴ | 6 | 0.149 151 086 135 30 × 10² |
| 2 | −0.724 213 167 032 06 × 10⁶ | 7 | −0.482 326 573 615 91 × 10⁴ |
| 3 | −0.170 738 469 400 92 × 10² | 8 | 0.405 113 405 420 57 × 10⁶ |
| 4 | 0.120 208 247 024 70 × 10⁵ | 9 | −0.238 555 575 678 49 |
| 5 | −0.323 255 503 223 33 × 10⁷ | 10 | 0.650 175 348 447 98 × 10³ |

**Forward direction** $P_{sat}(T)$: with $\vartheta$ from (2.3),

$$
A = \vartheta^2 + n_1\vartheta + n_2, \qquad
B = n_3\vartheta^2 + n_4\vartheta + n_5, \qquad
C = n_6\vartheta^2 + n_7\vartheta + n_8,
$$

$$
P_{sat} = P^* \left[ \frac{2C}{-B + \sqrt{B^2 - 4AC}} \right]^{4}.
\tag{2.4}
$$

**Backward direction** $T_{sat}(P)$: with $\beta = (P/P^*)^{1/4}$,

$$
E = \beta^2 + n_3\beta + n_6, \qquad
F = n_1\beta^2 + n_4\beta + n_7, \qquad
G = n_2\beta^2 + n_5\beta + n_8, \qquad
D = \frac{2G}{-F - \sqrt{F^2 - 4EG}},
$$

$$
T_{sat} = T^* \cdot \frac{n_{10} + D - \sqrt{(n_{10}+D)^2 - 4\,(n_9 + n_{10}D)}}{2}.
\tag{2.5}
$$

**Validity:** $273.15\ \text{K} \le T \le 647.096\ \text{K}$ (i.e. 611.2 Pa ≤ $P$ ≤ 22.064 MPa) — the full liquid–vapor coexistence line, covering every operating condition this system can physically reach.

### 2.2.2 Worked example 1: evaporator, $T_e = 10\,^\circ\text{C}$

$T = 283.15$ K. From (2.3): $\vartheta = 283.15 + \dfrac{-0.238556}{283.15 - 650.175} = 283.15 + 0.000650 = 283.150650$.

$$
A = -3.13587\times 10^{5}, \qquad B = -1.19773\times 10^{6}, \qquad C = 2.35211\times 10^{5},
$$

$$
\sqrt{B^2 - 4AC} = \sqrt{1.43457\times10^{12} + 2.95059\times10^{11}} = 1.31515\times10^{6},
$$

$$
\frac{2C}{-B+\sqrt{\cdot}} = \frac{4.70422\times10^{5}}{2.51289\times10^{6}} = 0.187203
\;\;\Rightarrow\;\;
P_{sat} = (0.187203)^4\ \text{MPa} = 1.2282\ \text{kPa}. \;\checkmark
$$

This is the canonical $P_1 = 1.228$ kPa of the [README](../README.md) — an absolute pressure of ~1.2 % of one atmosphere.

### 2.2.3 Worked example 2: condenser, $T_c = 125\,^\circ\text{C}$

$T = 398.15$ K → $\vartheta = 398.150947$,

$$
A = -1.01026\times10^{5}, \qquad B = -1.15307\times10^{6}, \qquad C = 8.49131\times10^{5},
$$

$$
\frac{2C}{-B+\sqrt{B^2-4AC}} = \frac{1.69826\times10^{6}}{2.44640\times10^{6}} = 0.694190
\;\;\Rightarrow\;\;
P_{sat} = (0.694190)^4 = 0.23222\ \text{MPa} = 232.2\ \text{kPa}. \;\checkmark
$$

Backward check with (2.5): $P = 232.2$ kPa → $\beta = 0.232200^{1/4}=0.694170$ → $D = 398.148$ → $T_{sat} = 398.147\ \text{K} = 125.00\,^\circ\text{C}$. The forward and backward branches close to < 4 mK.

Two further pairs the project quotes repeatedly: $T_{sat}(17\ \text{kPa}) = 56.6\,^\circ\text{C}$ and $T_{sat}(20\ \text{kPa}) = 60.1\,^\circ\text{C}$ — the intermediate-pressure band $P_{int} \approx 17\text{–}20$ kPa of the interstage manifold (its actual value is *emergent* from the two compressor characteristics, see [doc 06](06-cycle-coupling-steady-state.md)); and the geometric-mean guideline $\sqrt{P_1 P_4} = \sqrt{1.228 \times 232.2} = 16.9$ kPa → $T_{sat} = 56.4\,^\circ\text{C}$.

---

## 2.3 Clausius–Clapeyron: the didactic form and the vacuum-side penalty

Along the coexistence line the specific Gibbs energies of liquid and vapor are equal, $g_f(T,P_{sat}) = g_g(T,P_{sat})$. Differentiating along the line and using $\mathrm{d}g = -s\,\mathrm{d}T + v\,\mathrm{d}P$:

$$
-s_f\,\mathrm{d}T + v_f\,\mathrm{d}P = -s_g\,\mathrm{d}T + v_g\,\mathrm{d}P
\quad\Longrightarrow\quad
\frac{\mathrm{d}P_{sat}}{\mathrm{d}T} = \frac{s_g - s_f}{v_g - v_f} = \frac{h_{fg}}{T\,v_{fg}} ,
\tag{2.6}
$$

where the last step uses $s_{fg} = h_{fg}/T$ (isothermal, isobaric phase change). Equation (2.6) is exact; it becomes the familiar exponential $P_{sat} \propto e^{-h_{fg}/RT}$ if one additionally assumes ideal vapor ($v_{fg} \approx v_g = RT/P$) and constant $h_{fg}$. In this project (2.6) is used **didactically and for sensitivity estimates**; all production numbers come from Region 4.

**Evaluation at the two ends of the cycle:**

| Point | $h_{fg}$ (kJ/kg) | $T$ (K) | $v_{fg}$ (m³/kg) | $\mathrm{d}P_{sat}/\mathrm{d}T$ |
|---|---|---|---|---|
| 10 °C (evaporator) | 2477.2 | 283.15 | 106.32 | $\dfrac{2\,477\,200}{283.15 \times 106.32} = \mathbf{82.3\ Pa/K}$ |
| 60 °C (interstage) | 2357.7 | 333.15 | 7.666 | 0.92 kPa/K |
| 120 °C (condenser band) | 2202.1 | 393.15 | 0.8903 | $\mathbf{6.29\ kPa/K}$ |

(Direct numerical differentiation of Region 4 gives 82.3 Pa/K and 6.29 kPa/K — identical, confirming internal consistency. Across the condensing band the slope runs from ≈ 5 kPa/K near 112 °C to 7.1 kPa/K at 125 °C.)

**Engineering meaning — the pressure-drop-to-saturation-temperature penalty.** Any pressure drop $\Delta P$ in a line that operates at (or feeds) a saturation state costs an equivalent saturation-temperature shift

$$
\Delta T_{penalty} = \frac{\Delta P}{\mathrm{d}P_{sat}/\mathrm{d}T}.
\tag{2.7}
$$

- **Vacuum side:** 100 Pa of suction-line pressure drop (a very easy number to accumulate at 1 m³/s of vapor flow) costs $100/82.3 \approx 1.2$ K of effective thermal lift — the evaporator must run 1.2 K colder to deliver the same compressor-inlet pressure. This single number drives the generous suction-duct sizing of [doc 10](10-prototype-design.md) and the decision to superheat with a recuperator (回热器) rather than long suction lines.
- **Pressure side:** the same 100 Pa at 120 °C costs 0.016 K — completely negligible. High-side pressure drop is an economic, not thermodynamic, concern.
- **Instrumentation corollary:** inferring $T_{sat}$ from a pressure sensor on the vacuum side requires ~40 Pa absolute accuracy for 0.5 K of temperature resolution — a 0.04 %-of-atmosphere spec. [Doc 09](09-control-design.md) therefore pairs every low-side pressure transmitter with a direct temperature measurement.

---

## 2.4 Saturated liquid: $h_f$, $s_f$ and the Region 1 formulation

Over the project's liquid range (10–130 °C) liquid water is nearly incompressible with a slowly varying specific heat $c_{p,f} = 4.19\text{–}4.25$ kJ/(kg·K). Referenced to the triple point ($h_f \approx 0$ at 0.01 °C):

$$
h_f(T) \approx \bar{c}_{p,f}\,\bigl(T - T_{tr}\bigr), \qquad \bar{c}_{p,f} \approx 4.20\ \text{kJ/(kg·K)},
\tag{2.8}
$$

$$
s_f(T) \approx \bar{c}_{p,f}\,\ln\!\frac{T}{T_{tr}}, \qquad T_{tr} = 273.16\ \text{K}.
\tag{2.9}
$$

Check against IF97: (2.8) gives $h_f(125\,^\circ\text{C}) \approx 4.20 \times 125 = 525.0$ kJ/kg vs the exact 525.0; (2.9) gives $s_f(125\,^\circ\text{C}) \approx 4.20\,\ln(398.15/273.16) = 1.582$ vs the exact 1.5816 kJ/(kg·K). Errors stay below ~0.5 % up to 130 °C — fine for hand calculations of the liquid line, the economizer (经济器) hot side, and flash qualities.

For full consistency the interactive property engine does **not** use (2.8)–(2.9): it implements the IF97 **Region 1** Gibbs formulation,

$$
\frac{g(P,T)}{RT} = \gamma(\pi,\tau) = \sum_{i=1}^{34} n_i\,(7.1-\pi)^{I_i}\,(\tau - 1.222)^{J_i},
\qquad \pi = \frac{P}{16.53\ \text{MPa}}, \quad \tau = \frac{1386\ \text{K}}{T},
\tag{2.10}
$$

from which $v$, $h$, $s$, $c_p$ follow by the same derivative pattern as Region 2 (§2.5). Saturated-liquid states are evaluated as Region 1 at $(T, P_{sat}(T))$. The pressure dependence of $h_f$ is tiny here (subcooled liquid at 232.2 kPa vs saturation: < 0.25 kJ/kg), which is why (2.8) works so well.

**Where it matters in the cycle.** The main expansion valve (主膨胀阀) flashes high-pressure liquid to $P_1$ isenthalpically; the flash quality at the evaporator inlet (state 11 — renumbered from the task figure's duplicated "T10", see [doc 01](01-system-overview.md)) is

$$
x_{11} = \frac{h_{valve\,in} - h_f(10\,^\circ\text{C})}{h_{fg}(10\,^\circ\text{C})} =
\begin{cases}
\dfrac{503.8 - 42.0}{2477.2} = 0.186 & \text{liquid at } 120\,^\circ\text{C (no internal recovery)},\\[2ex]
\dfrac{251.2 - 42.0}{2477.2} = 0.084 & \text{liquid cooled to } 60\,^\circ\text{C (economizer + recuperator)}.
\end{cases}
$$

Halving the flash quality nearly halves the vapor generated uselessly in the valve — the quantitative case for the internal heat-recovery train, developed fully in [doc 05](05-valves-mixing-junctions.md) and [doc 07](07-performance-cop-optimization.md).

---

## 2.5 Saturated and superheated vapor: IF97 Region 2

### 2.5.1 Gibbs-energy structure

All vapor states (cycle states 1–5 and 10) are evaluated with the IF97 **Region 2** dimensionless Gibbs energy, split into an ideal-gas and a residual part:

$$
\frac{g(P,T)}{RT} = \gamma(\pi,\tau) = \gamma^{\circ}(\pi,\tau) + \gamma^{r}(\pi,\tau),
\qquad \pi = \frac{P}{1\ \text{MPa}}, \quad \tau = \frac{540\ \text{K}}{T}.
\tag{2.11}
$$

**Ideal-gas part** — nine terms:

$$
\gamma^{\circ} = \ln\pi + \sum_{i=1}^{9} n_i^{\circ}\,\tau^{J_i^{\circ}},
\tag{2.12}
$$

| $i$ | $J_i^{\circ}$ | $n_i^{\circ}$ | $i$ | $J_i^{\circ}$ | $n_i^{\circ}$ |
|---|---|---|---|---|---|
| 1 | 0 | −0.969 276 865 002 17 × 10¹ | 6 | −2 | 0.142 408 191 714 44 × 10¹ |
| 2 | 1 | 0.100 866 559 680 18 × 10² | 7 | −1 | −0.438 395 113 194 50 × 10¹ |
| 3 | −5 | −0.560 879 112 830 20 × 10⁻² | 8 | 2 | −0.284 086 324 607 72 |
| 4 | −4 | 0.714 527 380 814 55 × 10⁻¹ | 9 | 3 | 0.212 684 637 533 07 × 10⁻¹ |
| 5 | −3 | −0.407 104 982 239 28 | | | |

**Residual part** — a 43-term double power series in $\pi$ and $(\tau - 0.5)$:

$$
\gamma^{r} = \sum_{i=1}^{43} n_i\,\pi^{I_i}\,(\tau - 0.5)^{J_i},
\qquad I_i \in \{1,\dots,24\},\; J_i \in \{0,\dots,58\},
\tag{2.13}
$$

with the exponent/coefficient set tabulated in IAPWS-IF97 (2007), Table 11. The 43 coefficients are **not** reproduced here; the JavaScript engine in `html/index.html` implements the full set verbatim and self-tests against the official verification states (§2.8). Region 2 covers 273.15–1073.15 K up to 100 MPa (vapor side of the dome) — every superheated state this machine can produce, including the ~370 °C stage-1 discharge, lies far inside it.

### 2.5.2 Properties from derivatives

With $\gamma_\pi = (\partial\gamma/\partial\pi)_\tau$, $\gamma_\tau = (\partial\gamma/\partial\tau)_\pi$ etc., and noting $\pi\,\gamma^{\circ}_\pi = 1$:

$$
v = \frac{RT}{P}\,\pi\bigl(\gamma^{\circ}_\pi + \gamma^{r}_\pi\bigr)
  = \frac{RT}{P}\bigl(1 + \pi\gamma^{r}_\pi\bigr), \qquad
h = RT\,\tau\bigl(\gamma^{\circ}_\tau + \gamma^{r}_\tau\bigr),
\tag{2.14}
$$

$$
s = R\Bigl[\tau\bigl(\gamma^{\circ}_\tau + \gamma^{r}_\tau\bigr) - \bigl(\gamma^{\circ} + \gamma^{r}\bigr)\Bigr], \qquad
c_p = -R\,\tau^2\bigl(\gamma^{\circ}_{\tau\tau} + \gamma^{r}_{\tau\tau}\bigr).
\tag{2.15}
$$

The form $v = (RT/P)(1+\pi\gamma^r_\pi)$ makes the compressibility factor explicit: $Z = 1 + \pi\gamma^r_\pi$, quantified in §2.6. Saturated-vapor properties are Region 2 evaluated at $(T, P_{sat}(T))$; two-phase states use quality mixing, $h = h_f + x\,h_{fg}$, $s = s_f + x\,s_{fg}$, $v = v_f + x\,v_{fg}$.

### 2.5.3 Saturation-property table (IF97)

| $t$ (°C) | $P_{sat}$ (kPa) | $v_g$ (m³/kg) | $h_f$ (kJ/kg) | $h_{fg}$ (kJ/kg) | $h_g$ (kJ/kg) | $s_f$ (kJ/(kg·K)) | $s_g$ (kJ/(kg·K)) | Cycle role |
|---|---|---|---|---|---|---|---|---|
| 0.01 | 0.612 | 206.0 | 0.00 | 2500.9 | 2500.9 | 0.0000 | 9.156 | Triple point — hard lower bound |
| 10 | 1.228 | 106.3 | 42.0 | 2477.2 | 2519.2 | 0.1511 | 8.900 | $T_e$: evaporation (state 11 → 1) |
| 20 | 2.339 | 57.76 | 83.9 | 2453.5 | 2537.4 | 0.2965 | 8.666 | Source-water inlet temperature |
| 56 | 16.53 | 9.15 | 234.4 | 2367.4 | 2601.8 | 0.781 | 7.975 | $T_{sat}(P_{int})$ lower band (~17 kPa) |
| 60 | 19.95 | 7.667 | 251.2 | 2357.7 | 2608.8 | 0.8312 | 7.908 | $T_{sat}(P_{int})$ upper band (~20 kPa) |
| 120 | 198.7 | 0.8913 | 503.8 | 2202.1 | 2706.0 | 1.5279 | 7.129 | Process-water outlet temperature |
| 125 | 232.2 | 0.7701 | 525.0 | 2188.5 | 2713.5 | 1.5816 | 7.077 | $T_c$: condensation (state 5 → 6) |
| 130 | 270.3 | 0.6681 | 546.4 | 2173.7 | 2720.1 | 1.6346 | 7.027 | Condenser design margin |

Consistency checks against the [README](../README.md) design point: $\dot m_c \approx \dot Q_{cond}/(h_5 - h_6) \approx 50/(3450 - 525) \approx 0.017$ kg/s ✓ ($h_5 \approx 3450$ kJ/kg from [doc 03](03-compressor-models.md); the flow ratio $0.017/0.012 \approx 1.4$ is consistent with the README bookkeeping $\dot m_c = \dot m_e(1 + r + w_s) \approx 1.36\,\dot m_e$ within rounding); $\dot m_e \approx \dot Q_{evap}/(h_1 - h_{11}) \approx 27/(2519 - 251) \approx 0.012$ kg/s ✓; suction volume flow $\dot m_e v_1 \approx 0.012 \times 106.3 \approx 1.3$ m³/s ✓. Note also how $h_{fg}$ *shrinks* only 12 % over a 115 K span while $P_{sat}$ grows 189× — the latent heat is robust, the pressure is not.

---

## 2.6 Ideal-gas shortcut for hand calculations

### 2.6.1 How ideal is the vapor?

Low reduced pressure ($P/P_{crit} \le 232.2/22\,064 \approx 0.011$) makes steam nearly ideal throughout the cycle. Computed from IF97 ($Z = Pv/RT$):

| State | $P$ (kPa) | $T$ (°C) | $v$ IF97 (m³/kg) | $RT/P$ (m³/kg) | $Z$ |
|---|---|---|---|---|---|
| Sat. vapor, evaporator | 1.228 | 10 | 106.32 | 106.40 | **0.999** |
| Sat. vapor, interstage | 19.95 | 60 | 7.667 | 7.708 | 0.995 |
| Superheated, HP discharge level | 200 | 250 | 1.199 | 1.207 | 0.993 |
| Sat. vapor, condenser | 232.2 | 125 | 0.7701 | 0.7913 | **0.973** |

The worst case in the whole cycle is the **saturated** vapor at the condensing pressure, $Z = 0.973$ (≈ 3 % dense-vapor correction); everywhere the compressors actually work — superheated and/or low pressure — $Z \ge 0.99$. So $Pv = RT$ is a legitimate 1–3 % tool for sizing estimates, and the ideal-gas *temperature*-only enthalpy $h \approx h(T)$ is even better away from the dome.

### 2.6.2 Ideal-gas specific heat and isentropic exponent

A standard cubic fit (Borgnakke & Sonntag, valid 250–1200 K, max error ≈ 0.5 %):

$$
c_p^{0}(T) = 1.79 + 0.107\,\theta + 0.586\,\theta^{2} - 0.20\,\theta^{3}
\quad \text{kJ/(kg·K)}, \qquad \theta = \frac{T}{1000\ \text{K}}.
\tag{2.16}
$$

Values: 1.86 at 10 °C, 1.89 at 60 °C, 1.91 at 125 °C, 2.05 at 370 °C. The isentropic exponent $k = c_p^0/(c_p^0 - R)$ then runs from 1.33 (283 K) down to 1.29 (640 K); the mean over the stage-1 compression path is

$$
k \approx 1.32 .
$$

### 2.6.3 Isentropic compression estimate and its error

For an ideal gas with constant $k$:

$$
T_{out,s} = T_{in} \left(\frac{P_{out}}{P_{in}}\right)^{\frac{k-1}{k}}, \qquad
w_{is} = \frac{kRT_{in}}{k-1}\left[\left(\frac{P_{out}}{P_{in}}\right)^{\frac{k-1}{k}} - 1\right].
\tag{2.17}
$$

**Stage 1 at the design pressure ratio** $\Pi = P_{int}/P_1 = 13.8$, saturated suction $T_{in} = 283.15$ K (canonical discharge = state 3):

$$
T_{3s} = 283.15 \times 13.8^{\,0.32/1.32} = 283.15 \times 1.8895 = 535.0\ \text{K} = 261.8\,^\circ\text{C},
$$

$$
w_{is} = \frac{1.32 \times 0.461526 \times 283.15}{0.32} \times 0.8895 = 479\ \text{kJ/kg}.
$$

The full IF97 calculation (constant $s = 8.900$ kJ/(kg·K) from 1.228 kPa to 16.9 kPa) gives **≈ 261 °C** and $\Delta h_s \approx 478$ kJ/kg — the shortcut lands **within ~1 K and ~0.3 %**. Two caveats keep this a *check*, not the production method:

- **Sensitivity to $k$:** $\partial T_{3s}/\partial k \approx +8$ K per +0.01 at this pressure ratio (k = 1.31 → 254 °C, k = 1.33 → 270 °C). The good agreement above depends on choosing the correct path-mean $k$, which itself requires property data — circular for anything but a check.
- **Real discharge:** with $\eta_{is} = 0.70$, $\Delta h = 479/0.70 = 684$ kJ/kg, giving $T_3 \approx 370\,^\circ\text{C}$ (IF97), the README anchor. At these temperatures $c_p^0$ has risen ~9 % above its suction value, and constant-$k$ formulas drift by 5–10 K if uncorrected.

Verdict: use (2.17) freely on paper and in first-pass sizing; use the IF97 engine (and [doc 03](03-compressor-models.md)'s $h$-based efficiency definitions, which never assume ideal gas) for all reported numbers.

---

## 2.7 Transport properties for the heat-exchanger models

The correlations in [doc 04](04-heat-exchanger-models.md) (Dittus–Boelter/Gnielinski single-phase, Nusselt film condensation, pool/flow boiling) need viscosity $\mu$, thermal conductivity $\lambda$, and Prandtl number $Pr = \mu c_p/\lambda$ for both phases.

### 2.7.1 Liquid water, 0–130 °C (saturation line, IAPWS 2008/2011 formulations)

| $t$ (°C) | $\mu_f$ (μPa·s) | $\lambda_f$ (W/(m·K)) | $c_{p,f}$ (kJ/(kg·K)) | $Pr_f$ |
|---|---|---|---|---|
| 0 | 1792 | 0.561 | 4.219 | 13.5 |
| 10 | 1306 | 0.580 | 4.195 | 9.45 |
| 20 | 1002 | 0.598 | 4.184 | 7.01 |
| 40 | 653 | 0.631 | 4.179 | 4.32 |
| 60 | 467 | 0.654 | 4.185 | 2.98 |
| 80 | 354 | 0.670 | 4.197 | 2.22 |
| 100 | 282 | 0.679 | 4.217 | 1.75 |
| 120 | 232 | 0.683 | 4.244 | 1.44 |
| 130 | 213 | 0.684 | 4.263 | 1.33 |

Compact fits for the solver (both within ~±2.5 % over 0–130 °C, $T$ in K):

$$
\mu_f(T) = 2.414\times10^{-5} \cdot 10^{\,247.8/(T-140)} \ \ \text{Pa·s},
\tag{2.18}
$$

$$
\lambda_f(T) = -0.5752 + 6.397\times10^{-3}\,T - 8.151\times10^{-6}\,T^{2} \ \ \text{W/(m·K)}.
\tag{2.19}
$$

(Check: (2.18) at 20 °C → 1002 μPa·s ✓; (2.19) at 20 °C → 0.600 vs 0.598 ✓.) The strong temperature dependence of $\mu_f$ — a factor 8.4 between 0 and 130 °C — is why [doc 04](04-heat-exchanger-models.md) evaluates liquid-side film coefficients at the local film temperature rather than a loop-wide constant.

### 2.7.2 Steam (low-pressure / saturated vapor), 10–370 °C

Computed from the IAPWS dilute-gas viscosity (2008) and thermal-conductivity (2011) formulations; density corrections at this project's pressures are ≤ 2 % and are noted where relevant.

| $t$ (°C) | $\mu_g$ (μPa·s) | $\lambda_g$ (W/(m·K)) | $Pr_g$ | Where it is used |
|---|---|---|---|---|
| 10 | 9.2 | 0.0174 | 0.99 | Evaporator vapor space, suction line |
| 60 | 10.9 | 0.0210 | 0.98 | Interstage manifold, economizer vapor |
| 125 | 13.3 | 0.0263¹ | 0.97¹ | Condensing-zone vapor core |
| 200 | 16.2 | 0.0331 | 0.96 | Condenser desuperheating zone |
| 300 | 20.3 | 0.0434 | 0.94 | HP discharge / desuperheater inlet |
| 370 | 23.2 | 0.0511 | 0.93 | Stage-1 discharge extreme (η_is = 0.70) |

¹ Dilute-gas values; at the actual 232.2 kPa saturation state the density corrections and the elevated real-gas $c_p$ raise the effective $Pr_g$ to ≈ 1.05–1.1. The gas-phase $\lambda_g$ being **25–40× smaller** than liquid $\lambda_f$ is the property-level root of the condenser (冷凝器) multi-zone problem flagged in the README: the desuperheating zone runs a gas-side film coefficient an order of magnitude below the condensing zone yet carries 15–25 % of the duty, so it consumes a disproportionate share of area ([doc 04](04-heat-exchanger-models.md), §4.4).

For quick interpolation over 10–370 °C, $\mu_g$ is nearly linear in $t$: $\mu_g \approx 8.9 + 0.039\,t\ (^\circ\text{C})$ μPa·s, within ±4 %; $Pr_g \approx 0.95 \pm 0.05$ may be treated as constant in first-pass gas-side correlations.

---

## 2.8 Implementation in the interactive engine, and self-tests

The JavaScript property engine embedded in [`html/index.html`](../html/index.html) implements, directly from the IAPWS-IF97 (2007) release:

- **Region 4** — equations (2.4)/(2.5), both directions, closed-form (§2.2);
- **Region 2** — full $\gamma^{\circ}$ (9 terms) + $\gamma^{r}$ (43 terms) with analytic first and second derivatives, giving $v, h, s, c_p$ via (2.14)–(2.15); used for cycle states 1, 2, 3, 4, 5, 10 and all superheated queries;
- **Region 1** — full 34-term Gibbs polynomial (2.10) with derivatives; used for states 6, 7, 8 (subcooled/saturated liquid);
- **derived services** — saturated mixtures by quality ($x_{9}$, $x_{11}$), inverse lookups $T(P,h)$ and $T(P,s)$ by safeguarded Newton iteration on the Region 2 forward equations (used for isentropic end states and condenser zone boundaries), and superheat $SH = T - T_{sat}(P)$.

Regions 3 and 5 (near-critical and >1073 K) are deliberately omitted: no reachable state of this machine enters them, and their absence is guarded by range assertions.

**Self-test values.** On load the engine evaluates the official IF97 verification tables (release Tables 5, 15, 35/36) and refuses to run the cycle solver if any check misses beyond round-off:

| Region | Input | Quantity | Required value | Role |
|---|---|---|---|---|
| 4 | $T = 300$ K | $P_{sat}$ | 3.536 589 41 kPa | Table 35 check; this engine reproduces it (§2.2 code path) |
| 4 | $P = 0.1$ MPa | $T_{sat}$ | 372.755 919 K | Backward-branch check (Table 36) |
| 2 | $T = 300$ K, $P = 3.5$ kPa | $v$ | 39.491 386 6 m³/kg | Vacuum-vapor volume — the quantity the LP compressor sizing hangs on |
| 2 | $T = 300$ K, $P = 3.5$ kPa | $h$ | 2549.911 45 kJ/kg | Vapor enthalpy datum consistency |
| 2 | $T = 300$ K, $P = 3.5$ kPa | $s$ | 8.522 389 67 kJ/(kg·K) | Entropy — isentropic end-state accuracy |
| 1 | $T = 300$ K, $P = 3$ MPa | $h$ | 115.331 273 kJ/kg | Liquid enthalpy — flash-quality accuracy |

That the Region 2 verification state (300 K, 3.5 kPa) happens to sit almost exactly in this machine's evaporator operating regime is a happy accident: the official test exercises precisely the deep-vacuum corner where this project needs the property engine to be most trustworthy.

**Datum convention.** IF97 sets internal energy and entropy of saturated liquid to zero at the triple point; all enthalpies in this project inherit that datum. Only enthalpy *differences* ever enter the balances, but mixed-datum errors are a classic failure mode when cross-checking against external tables — every number in these documents uses the IF97 datum.

---

## 2.9 Summary of property anchors used downstream

| Anchor | Value | Consumed by |
|---|---|---|
| $P_{sat}(10\,^\circ\text{C}) = 1.228$ kPa, $v_g = 106.3$ m³/kg | §2.2, §2.5 | LP compressor displacement, suction sizing ([doc 03](03-compressor-models.md), [doc 10](10-prototype-design.md)) |
| $P_{sat}(125\,^\circ\text{C}) = 232.2$ kPa | §2.2 | Condenser pressure, per-stage ratio ≈ 13.8 |
| $\mathrm{d}P_{sat}/\mathrm{d}T = 82.3$ Pa/K @ 10 °C; 6.29 kPa/K @ 120 °C | §2.3 | Pressure-drop penalties, sensor specs ([doc 04](04-heat-exchanger-models.md), [doc 09](09-control-design.md)) |
| $h_{fg}(10\,^\circ\text{C}) = 2477.2$, $h_{fg}(125\,^\circ\text{C}) = 2188.5$ kJ/kg | §2.5 | Mass-flow scale (ṁ_c ≈ 0.02 kg/s), flash qualities 0.186/0.084 ([doc 05](05-valves-mixing-junctions.md)) |
| $T_{sat}(17\text{–}20\ \text{kPa}) = 56.6\text{–}60.1\,^\circ\text{C}$ | §2.2 | Interstage/economizer band ([doc 06](06-cycle-coupling-steady-state.md), [doc 07](07-performance-cop-optimization.md)) |
| $k \approx 1.32$, $T_{3s} \approx 261\,^\circ\text{C}$ @ Π = 13.8 | §2.6 | Discharge-temperature screening; desuperheating requirement ([doc 03](03-compressor-models.md), [doc 05](05-valves-mixing-junctions.md)) |
| $\lambda_g/\lambda_f \approx 1/25\text{–}1/40$, $Pr_g \approx 0.95$ | §2.7 | Multi-zone condenser area split ([doc 04](04-heat-exchanger-models.md)) |

With the fluid fully characterized, [doc 03](03-compressor-models.md) builds the two compressor (压缩机) models on top of these relations.
