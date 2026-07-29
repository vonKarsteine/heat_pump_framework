# 09 — Control Strategy Design

*Defines the control objectives, controlled/manipulated-variable selection, loop pairing (RGA), the layered PID + override architecture with tuning rules, an optional MPC supervisory layer, start-up/shutdown sequencing, and the safety interlock matrix — all built on the dynamic model of [doc 08](08-dynamic-model.md) and verified against the scenarios in §9.8; the resulting setpoints and instrument list feed [doc 10](10-prototype-design.md).*

---

## 9.1 Control problem statement

The task sheet requires the study of "系统在不同工况下（环境温度、负荷变化等）的动态响应和控制策略" — the
dynamic response and control strategy of the system under varying ambient/source conditions and
load. Translated into a control specification for the 50 kW prototype of
[doc 10](10-prototype-design.md):

> **Primary objective.** Hold the process-water outlet temperature at
> $T_{proc,out} = 120\,^\circ\mathrm{C} \pm 0.5\ \mathrm{K}$ for load demands between 30 % and
> 100 % of the 50 kW design capacity and source-water inlet temperatures
> $T_{src,in} = 15\ldots25\,^\circ\mathrm{C}$, while never violating the operating-envelope
> constraints of §9.2.

The runtime manipulated variables (MVs) are exactly the task's class-3 inputs
(运行过程中可调控的运行变量):

| MV | Symbol | Actuator | Range | Rate limit |
|----|--------|----------|-------|-----------|
| LP compressor speed (第一级压缩机转速) | $N_1$ | VFD | 30–100 % of $N_{1,max}$ | ≤ 2 %/s |
| HP compressor speed (第二级压缩机转速) | $N_2$ | VFD | 30–100 % of $N_{2,max}$ | ≤ 2 %/s |
| Main expansion valve (主膨胀阀) | $u_{mv}$ | Stepper/EEV | 0–100 % | ≤ 10 %/s |
| Injection valve (补气阀) | $u_{iv}$ | Stepper/EEV | 0–100 % | ≤ 10 %/s |
| Interstage spray valve (喷液减温阀, if fitted per [doc 05](05-valves-mixing-junctions.md)) | $u_{spray}$ | EEV | 0–100 % | ≤ 20 %/s |
| Source pump speed (水源泵转速) | $N_{p,src}$ | VFD | 20–100 % | ≤ 5 %/s |
| Process pump speed (工艺水泵转速) | $N_{p,proc}$ | VFD | 20–100 % | ≤ 5 %/s |

Measured disturbances $d(t)$: source inlet temperature $T_{src,in}$ and the demanded load
$\dot Q_{dem}$ (equivalently the process-water flow $\dot m_{proc}$ imposed by the plant). Both
are available to the controller as feedforward signals; neither can be manipulated by the heat
pump itself.

Two features of the R718 cycle dominate the control design and recur throughout this document:

1. **Deep vacuum on suction** ($P_1 = 1.228$ kPa, $v_1 \approx 106$ m³/kg,
   $\mathrm{d}P_{sat}/\mathrm{d}T \approx 82$ Pa/K at 10 °C): tiny absolute-pressure excursions are
   large saturation-temperature excursions, and the triple point (0.611 kPa) is a hard floor.
2. **Extreme discharge superheat** (stage-1 isentropic discharge ≈ 261 °C, ≈ 370 °C at
   $\eta_{is}=0.70$): the discharge-temperature override and the spray desuperheater are not
   refinements — they are what keeps the machine alive, since economizer injection alone
   ($r \approx 0.11$) cools the interstage mix by only ~30 K.

---

## 9.2 Controlled variables and operating envelope

### 9.2.1 CV selection

| Rank | CV | Definition | Setpoint (nominal) | Why it is controlled |
|------|----|-----------|--------------------|----------------------|
| Primary | $T_{proc,out}$ | Condenser (冷凝器) process-water outlet | 120 °C | The product spec — the only CV the customer sees |
| Regulatory | $SH_1$ | $T_1 - T_{sat}(P_1)$, evaporator (蒸发器) exit superheat | 4 K (band 3–8 K) | Droplet protection for the LP compressor; at 1.2 kPa a droplet carry-over erodes blades and collapses $\eta_{vol}$ |
| Regulatory | $P_{int}$ | Interstage manifold pressure | 17–19 kPa (from L2 optimizer, §9.5) | $P_{int}$ is *emergent* — it settles where the two compressor characteristics intersect ([doc 06](06-cycle-coupling-steady-state.md)); left uncontrolled it drifts with load and drags both stage pressure ratios with it |
| Constraint | $T_5$ | HP discharge temperature | — | Hard limit, see below |
| Constraint | $SH_4$ | HP suction superheat $T_4 - T_{sat}(P_{int})$ | ≥ 5 K | Spray desuperheater must not flood the HP suction |

### 9.2.2 Envelope constraints (the "never-violate" table)

| # | Constraint | Limit | Basis |
|---|-----------|-------|-------|
| E1 | $T_5 \le 250\,^\circ\mathrm{C}$ (hard trip), alarm 240 °C | Discharge | Lubricant/seal/material limit; without spray the stage-1 discharge alone would reach ≈ 370 °C at $\eta_{is}=0.70$ |
| E2 | $P_1 \ge 0.9$ kPa ($T_{sat} \approx 5.5\,^\circ\mathrm{C}$) | Suction | Triple point 0.611 kPa + margin; below it ice forms in the evaporator |
| E3 | $T_{src,out} \ge 3\,^\circ\mathrm{C}$ | Source water | Freeze protection of the source loop (§9.6.3) |
| E4 | $SH_1 \ge 1$ K at all times (target band 3–8 K) | LP suction | Droplet protection; upper bound limits suction specific volume penalty ($v \propto T$ at fixed $P$) |
| E5 | $N_i \le N_{i,max}$, $\dot N_i \le 2$ %/s | Both VFDs | Motor/rotor mechanical limit; torque margin at 106 m³/kg suction volume |
| E6 | Surge margin ≥ 10 % (if centrifugal stages per [doc 03](03-compressor-models.md)) | Both stages | Distance of operating point from surge line on the $\Pi$–$\dot V$ map |
| E7 | $0 \le u_{mv}, u_{iv}, u_{spray} \le 100$ % | Valves | Actuator range; controllers must be anti-windup-protected (§9.5) |
| E8 | Subcooling $\ge 2$ K at states 8 and 7 (valve inlets) | Liquid lines | Prevents flash-induced choking and erratic $K_v$; flash quality downstream is by design ($x_{11} = 0.084$ with the economizer + recuperator liquid heat-recovery chain, 0.186 without, per [doc 05](05-valves-mixing-junctions.md)) |
| E9 | $P_4 \le 270$ kPa (trip); PRV set 300 kPa | Condenser shell | $P_{sat}(130\,^\circ\mathrm{C}) = 270$ kPa; design point 232.2 kPa at $T_c = 125\,^\circ\mathrm{C}$ |
| E10 | Condensate level in hotwell within 20–80 % | Charge inventory | Pump NPSH / liquid seal on the main valve |

The envelope is enforced three times over: as saturation limits in the L1 regulatory loops, as
override selectors (§9.5), and as hard interlocks (§9.7). MPC, if fitted, additionally enforces
E1–E9 as explicit inequality constraints (§9.5.5).

---

## 9.3 Degrees of freedom, gain structure, and pairing

### 9.3.1 Degrees-of-freedom audit

Seven MVs face effectively **three to four fast CVs**. The surplus is resolved by assigning the
slow/auxiliary MVs away from the fast regulatory layer:

- $\dot m_{proc}$ (process pump) is **set by the external load** — the plant demands a flow;
  for the heat pump it is a *measured disturbance*, not a control handle. (Where the heat pump
  owns the pump, $N_{p,proc}$ holds a constant $\Delta T_w = 5$ K by ratio control to load.)
- $N_{p,src}$ (source pump) is handed to the **L2 slow optimizer**: more source flow raises
  $T_e$ and COP but costs pump power; the optimum moves on a timescale of minutes (§9.5.4).
- $u_{spray}$ is a **constraint controller**, active only through the $T_5$/interstage-temperature
  override — in normal operation it holds $T_4$ near $T_{sat}(P_{int}) + SH_4^{sp}$.

That leaves the **3×3 fast core**: MVs $\{N, u_{mv}, u_{iv}\}$ against CVs
$\{T_{proc,out}, SH_1, P_{int}\}$, where $N$ is the *common speed command* and the ratio
$\rho_N = N_1/N_2$ is a fourth, quasi-static handle (see §9.3.4):

$$
N_1 = N\,\rho_N^{1/2}, \qquad N_2 = N\,\rho_N^{-1/2}. \tag{9.1}
$$

### 9.3.2 Steady-state gain signs

Qualitative gains at the nominal point, obtained from the linearization
$G(0) = -C A^{-1} B$ of the 10-state model in [doc 08](08-dynamic-model.md) (signs verified
against the steady-state sensitivities of [doc 07](07-performance-cop-optimization.md)):

| $\partial \mathrm{CV} / \partial \mathrm{MV}$ | $N$ (common) | $u_{mv}$ | $u_{iv}$ |
|---|---|---|---|
| $T_{proc,out}$ | **+ large** (flow ∝ speed → capacity) | + small (feeds evaporator, mild capacity gain) | + small (adds $\dot m_{inj}$ to condenser stream) |
| $SH_1$ | + moderate (draws evaporator down) | **− large** (floods evaporator) | ≈ 0 (weak, via $P_{int}$ back-pressure) |
| $P_{int}$ | ≈ 0 (both stages scale together — see Eq. 9.4) | + small (more LP throughput) | **+ large** (injects directly into the manifold) |

### 9.3.3 RGA on the 3×3 core

With the gains scaled by MV span and CV allowable range, a representative nominal gain matrix is

$$
G(0) =
\begin{bmatrix}
1.00 & 0.15 & 0.20\\
0.40 & -1.00 & 0.05\\
0.05 & 0.25 & 1.00
\end{bmatrix},
\qquad
\text{rows} = \{T_{proc,out},\, SH_1,\, P_{int}\},\ \ \text{cols} = \{N,\, u_{mv},\, u_{iv}\}. \tag{9.2}
$$

The relative gain array $\Lambda = G \otimes (G^{-1})^{\mathsf T}$ (element-wise product) evaluates to

$$
\Lambda =
\begin{bmatrix}
0.97 & 0.06 & -0.03\\
0.04 & 0.95 & 0.01\\
-0.01 & -0.01 & 1.02
\end{bmatrix}. \tag{9.3}
$$

All diagonal elements are within 0.95–1.02: the diagonal pairing is nearly decoupled at steady
state, interactions are one-way and mild, and decentralized PID is structurally justified. The
small negative off-diagonal entries (< 0.03 in magnitude) flag no pairing instability.

**Recommended pairing (from Eq. 9.3):**

| CV | ← MV | Loop name |
|----|------|-----------|
| $T_{proc,out}$ | $N$ (common speed) | Capacity loop |
| $SH_1$ | $u_{mv}$ | Superheat loop |
| $P_{int}$ | $\rho_N = N_1/N_2$ (primary), $u_{iv}$ trims economizer approach | Interstage loop |

**Alternative pairing** $P_{int} \leftarrow u_{iv}$ (valve does the pressure work, $\rho_N$ fixed):

| | $P_{int} \leftarrow \rho_N$ + $u_{iv}$ trim *(recommended)* | $P_{int} \leftarrow u_{iv}$ *(alternative)* |
|---|---|---|
| Pros | Injection ratio $r$ stays free to track the economizer-optimal value from [doc 07](07-performance-cop-optimization.md); valve stays mid-range; decouples from capacity (Eq. 9.4) | Fast (valve slews in ~1 s); single-VFD installations possible; simpler commissioning |
| Cons | $\rho_N$ authority limited by per-stage speed windows; slightly slower (VFD ramp) | $r$ becomes a slave of pressure control → COP gives up 2–4 %; valve saturates at low load, losing $P_{int}$ control exactly when interaction is worst |

The prototype implements the recommended pairing and retains the alternative as a fallback
selectable in software.

### 9.3.4 Why ratio-trim decouples capacity from $P_{int}$

Steady mass balance on the interstage manifold (states 3, 10 and the spray stream in; state 4
out), with displacement machines per [doc 03](03-compressor-models.md),
$\dot m = \eta_{vol}\,\rho_{suc}\,N\,V_{disp}$, and the spray mass fraction $w_s \approx 0.25$
when the desuperheater is active ($w_s = 0$ without it, per
[doc 05](05-valves-mixing-junctions.md)):

$$
\underbrace{\eta_{vol,1}\rho_2 N_1 V_{disp,1}}_{\dot m_3\, =\, \dot m_e}(1+r+w_s) \;=\; \underbrace{\eta_{vol,2}\,\rho_4(P_{int})\,N_2 V_{disp,2}}_{\dot m_4\, =\, \dot m_c}
\;\;\Longrightarrow\;\;
\rho_4(P_{int}) = \frac{\eta_{vol,1} V_{disp,1}}{\eta_{vol,2} V_{disp,2}}\,(1+r+w_s)\,\rho_2\,\frac{N_1}{N_2}. \tag{9.4}
$$

With $\rho_4 \approx P_{int}/(R\,T_4)$ ($R$ the specific gas constant of water, Eq. 2.1; the
ideal-gas form is accurate to ~1 % at 18 kPa per [doc 02](02-water-properties.md)), Eq. (9.4) says $P_{int} \propto N_1/N_2$ **and is invariant
under a common scaling** $N_1, N_2 \to \alpha N_1, \alpha N_2$: multiplying both speeds by
$\alpha$ scales capacity by ~$\alpha$ while leaving $P_{int}$ untouched. Hence the coordinate
change (9.1): the capacity loop moves $N$, the interstage loop moves $\rho_N$, and the two
commands are orthogonal to first order — this is precisely why $\Lambda_{31} \approx 0$ in
Eq. (9.3).

---

## 9.4 Expected loop dynamics

First-order-plus-dead-time (FOPDT) fits $G(s) = K e^{-\theta s}/(\tau s + 1)$ of the
[doc 08](08-dynamic-model.md) step responses at the nominal point:

| Loop | $K$ (scaled) | $\tau$ | $\theta$ | Notes |
|------|-------------|--------|----------|-------|
| $T_{proc,out} \leftarrow N$ | ≈ +0.05 K/% | 60–300 s | 5–20 s | Dominated by condenser wall + water thermal mass; $\tau$ grows toward 300 s at 30 % load (lower water flow) |
| $SH_1 \leftarrow u_{mv}$ | ≈ −0.8 K/% | 5–20 s | 1–3 s | Moving-boundary evaporator; **possible inverse response** at large opening steps (fresh flash liquid transiently depresses $P_1$ before the wetted length grows) — bounds the achievable bandwidth |
| $P_{int} \leftarrow \rho_N$ | ≈ +0.3 kPa/% | 2–10 s | ~1 s | Small manifold vapor volume; fast and clean |
| $P_{int} \leftarrow u_{iv}$ | ≈ +0.2 kPa/% | 2–10 s | < 1 s | Same volume, valve actuation faster than VFD |
| $T_4 \leftarrow u_{spray}$ | ≈ −2 K/% | 1–5 s | < 1 s | Near-instant evaporation of spray at 18 kPa |

The time-scale separation is a factor of 10–30 between the capacity loop and the regulatory
loops. Consequences: (i) cascade/decentralized control is well posed — the fast loops are at
quasi-steady state as seen by the capacity loop; (ii) the capacity loop may treat
$SH_1$ and $P_{int}$ as perfectly regulated; (iii) interaction that the RGA missed dynamically
(the transient $SH_1$ dip when $N$ steps up) is absorbed by the fast superheat loop before the
slow loop notices.

---

## 9.5 Layered control architecture

```
L3  (optional)  Offset-free MPC setpoint governor          — minutes
L2  Supervisory: P_int optimizer, SH adaptation, src-pump  — 1–10 min
L1  Regulatory PID/PI loops + override selectors           — 1–60 s
L0  Hardwired interlocks and protections (§9.7)            — < 1 s, no software
```

### 9.5.1 L0 — hardwired protection

Independent of the PLC: PRV on the condenser shell (300 kPa), discharge thermal cutout
switch at 260 °C, motor overcurrent, low-low level float. These act even if every controller
above is wrong. Full cause–action matrix in §9.7.

### 9.5.2 L1 — regulatory loops and tuning

All loops are PI (derivative only on the superheat loop) with **SIMC tuning** from the FOPDT
fits of §9.4:

$$
K_c = \frac{1}{K}\,\frac{\tau}{\tau_c + \theta}, \qquad
\tau_I = \min\!\bigl(\tau,\ 4(\tau_c + \theta)\bigr), \tag{9.5}
$$

| Loop | $\tau_c$ choice | Rationale |
|------|-----------------|-----------|
| $T_{proc,out} \leftarrow N$ | $\tau_c = 3\theta \approx 30$–60 s | Smooth capacity moves; the ±0.5 K spec allows detuning, and VFD rate limit E5 must not be the binding element |
| $SH_1 \leftarrow u_{mv}$ | $\tau_c = \theta \approx 2$ s, but ≥ inverse-response zero time constant | Tight — E4 is the constraint most easily violated; PID with derivative on measurement, filter $T_f = \tau_D/8$ |
| $P_{int} \leftarrow \rho_N$ | $\tau_c = \theta \approx 1$–2 s | Fast, clean loop; tight tuning is safe |
| $u_{iv}$ economizer-approach trim | $\tau_c \approx 5$ s | Holds economizer cold-side approach $T_{sat}(P_{int}) + 2\ \mathrm{K} - T_{10} \to 0$, i.e. state 10 just at/above saturation |
| $T_4 \leftarrow u_{spray}$ | $\tau_c = 2$ s | Setpoint $T_{sat}(P_{int}) + SH_4^{sp}$, $SH_4^{sp} = 5$–10 K |

Implementation rules applied to **every** loop:

- **Back-calculation anti-windup**, tracking time $\tau_t = \sqrt{\tau_I \tau_D}$ (PI: $\tau_t = \tau_I/2$):
  $$\dot I = \frac{K_c}{\tau_I}e + \frac{1}{\tau_t}\,(u_{applied} - u_{demanded}), \tag{9.6}$$
  so that when a valve pins at 0/100 % or the VFD hits its rate limit the integral bleeds back
  instead of winding up — essential because override selectors (below) routinely disconnect loops
  from their actuators.
- **Rate limiting on $N$** (E5) implemented *inside* the controller output stage so the
  anti-windup sees the true applied value.
- **Gain scheduling** on load and per-stage pressure ratio: a 3-point schedule
  (30 %, 65 %, 100 % load) of $(K_c, \tau_I)$ per loop, linearly interpolated. The plant gain of
  the capacity loop roughly doubles at 30 % load (same $\Delta N$ moves a larger *fraction* of
  capacity, and $\tau$ grows), so the scheduled $K_c$ halves; without scheduling the loop rings
  at low load.

### 9.5.3 L1 overrides (selector logic)

| Override | Selector | Action |
|----------|----------|--------|
| High $T_5$ | low-select on $N$ demand; parallel PI drives $u_{spray}$ open from $T_5 > 235\,^\circ\mathrm{C}$ | Spray first (cheap), speed cut second; capacity loop loses authority smoothly via (9.6) |
| Low $P_1$ (< 1.0 kPa) | low-select on $N_1$ demand | Cuts LP speed to arrest suction-pressure collapse (source too cold / evaporator starving); pairs with freeze mode §9.6.3 |
| Surge (E6) | high-select opens hot-gas/interstage bypass valve; low-select on speed | Classic surge controller with 10 % margin line; bypass is sized in [doc 10](10-prototype-design.md) |
| High $P_4$ (> 255 kPa) | low-select on $N_2$ demand | Backs the machine away from E9 before the interlock trips |

All selectors use the same back-calculation channel (9.6), so whichever controller is *not*
selected tracks the applied output and takeover is bumpless in both directions.

### 9.5.4 L2 — supervisory layer

- **$P_{int}$ setpoint** from the COP map of [doc 07](07-performance-cop-optimization.md):
  nominal start value $\sqrt{P_1 P_4} = 16.9$ kPa, corrected by the stored map
  $P_{int}^{opt}(\dot Q, T_{src,in})$ (the true optimum sits 5–15 % above the geometric mean
  because the desuperheat burden is asymmetric). Update every 1–2 min, rate-limited to
  0.2 kPa/min so L1 never sees a step.
- **Superheat setpoint adaptation** (MSS — minimum stable signature): slowly lower $SH_1^{sp}$
  from 8 K toward 3 K while the variance of $SH_1$ stays below a threshold; retreat 1 K on
  detection of hunting. Recovers 1–2 % COP versus a fixed conservative setpoint (lower superheat
  → lower $v_1$ → more mass flow per rev — significant when $v_1 \approx 106$ m³/kg).
- **Source-pump optimization**: hill-climb $N_{p,src}$ on the measured objective
  $\mathrm{COP}_{sys} = \dot Q_{cond}/(\dot W_{el,1} + \dot W_{el,2} + \dot W_{pumps})$
  (electrical powers, per the COP definition in the [README](../README.md)) with a 5-min
  period, constrained by E3 ($T_{src,out} \ge 3\,^\circ\mathrm{C}$).

### 9.5.5 L3 — optional offset-free linear MPC

A setpoint governor above L1 (MPC commands the L1 setpoints, not the actuators — L1 and L0
remain untouched as the safety-relevant layer):

| Item | Choice |
|------|--------|
| Model | 10-state linearization from [doc 08](08-dynamic-model.md) at 3 scheduled points, or an identified FOPDT bank (§9.4) converted to state space |
| Sample time | $T_s = 2$–5 s |
| Horizons | $N_p = 60$–120 (2–10 min, covers the capacity-loop settling), $N_c = 5$–10 |
| Constraints | Envelope table E1–E9 as output/input inequalities; MV rate limits as hard input-rate constraints |
| Offset-free tracking | Augmented **input-disturbance model** $\hat d_{k+1} = \hat d_k$, estimated by a Kalman filter on $\{T_{proc,out}, SH_1, P_{int}, T_5\}$; guarantees zero steady-state offset under plant–model mismatch |
| Cost | $\sum \lVert y - r \rVert_Q^2 + \lVert \Delta u \rVert_R^2$, $Q$ dominated by $T_{proc,out}$, soft-constraint slack heavily penalized |

**When MPC is worth it:** frequent large load swings (batch process customers), operation
pinned against E1/E6 where constraint-aware optimization buys real capacity, and low-load
operation where interactions grow. For a steady base-load duty the decentralized PID of §9.5.2
meets the ±0.5 K spec on its own, and MPC adds only maintenance burden — the recommendation is
to commission with PID and add L3 only if the §9.8 acceptance tests fail at 30 % load.

---

## 9.6 Sequencing

### 9.6.1 Start-up

1. **Evacuate** the whole circuit to **< 100 Pa** absolute with the vacuum/purge pump
   (真空泵) — mandatory for a sub-atmospheric working fluid; air ingress destroys condenser
   heat transfer (see interlock I8).
2. **Leak-rate hold check**: isolate the pump; accept if pressure rise ≤ 10 Pa over 60 min.
   Fail → leak hunt before any charge is admitted.
3. **Charge with deaerated demineralized water** (boiled/vacuum-degassed) to the level target
   of [doc 10](10-prototype-design.md).
4. **Establish water flows**: start source and process pumps at minimum, prove both flow
   switches (I5), confirm $T_{src,in}$ within 15–25 °C.
5. **Start LP compressor at minimum speed** with a *feedforward* main-valve opening
   $u_{mv}^{ff}(N_1, T_{src,in})$ taken from the steady-state map of
   [doc 06](06-cycle-coupling-steady-state.md) — at 1.2 kPa there is no margin for the superheat
   loop to find the operating point from a closed valve. Superheat loop closes once
   $SH_1$ reads valid ($P_1 > 0.9$ kPa and rising vapor flow).
6. **Build $P_{int}$**: LP alone raises the manifold to ≥ 5 kPa; $u_{iv}$ stays closed,
   $u_{spray}$ armed.
7. **Start HP compressor** at minimum speed; interstage loop closes on $\rho_N$; injection
   valve opens to its economizer-approach trim once $P_4 > P_{int} + 20$ kPa (forward
   pressure margin for the secondary valve, 次膨胀阀).
8. **Bumpless handover**: all L1 controllers have been tracking their applied outputs via
   (9.6); switching from sequence control to closed loop transfers without a bump.
9. **Setpoint ramp**: $T_{proc,out}^{sp}$ ramps from current value to 120 °C at ≤ 1 K/min,
   letting thermal stresses in the condenser and the 115 °C process loop equalize.

### 9.6.2 Shutdown

- **Normal**: ramp $N$ to minimum over ≥ 2 min, close $u_{iv}$, stop HP then LP, close
  $u_{mv}$, stop water pumps last (removes residual heat), **retain vacuum** — the circuit
  stays sealed under its own $P_{sat}(T_{ambient}) \approx 2$–3 kPa; never vent to atmosphere.
- **Trip**: compressors off immediately, both expansion valves to their fail positions
  ($u_{mv}$ fail-closed, $u_{iv}$ fail-closed, $u_{spray}$ fail-closed), pumps per cause
  (water flow is kept on a high-$T_5$ trip to scavenge heat), alarm latched until manual reset.

### 9.6.3 Low-source / freeze-protection mode (the R718 "defrost" discussion)

An air-source R744/R410A unit defrosts by cycle reversal. R718 has no analogue: the working
fluid itself **cannot evaporate below the triple point 0.01 °C / 0.611 kPa** — there is no
"run the evaporator below zero and defrost later" regime at all. Freeze risk is instead on the
**source-water side**. Therefore, instead of defrost cycles, the controller enforces a
**reduced-capacity envelope**:

- When $T_{src,out} \to 3\,^\circ\mathrm{C}$ (E3) or $P_1 \to 1.0$ kPa, a low-select override
  cuts $N_1$ (§9.5.3) — capacity follows the available source, and $T_{proc,out}$ is allowed to
  sag below setpoint with an operator alarm rather than freezing the evaporator.
- The L2 optimizer simultaneously raises $N_{p,src}$ toward maximum (more source flow → smaller
  water-side $\Delta T$ → higher $T_e$ at the same duty).
- Below $T_{src,in} = 12\,^\circ\mathrm{C}$ sustained, the unit derates to a published
  capacity curve $\dot Q_{max}(T_{src,in})$ derived from the E2/E3 intersection in
  [doc 07](07-performance-cop-optimization.md).

---

## 9.7 Interlock matrix

| # | Cause (latching) | Trip setting | Action | Reset |
|---|------------------|--------------|--------|-------|
| I1 | High HP discharge temp $T_5$ | > 250 °C (L0 cutout 260 °C) | Trip both compressors; water pumps ON 5 min (heat scavenge); valves fail-closed | Manual, after $T_5 < 180\,^\circ\mathrm{C}$ |
| I2 | High condenser pressure $P_4$ | > 270 kPa; PRV lifts 300 kPa | Trip HP, then LP 10 s later; process pump ON | Manual |
| I3 | Low suction pressure $P_1$ | < 0.8 kPa for 10 s | Trip LP (HP follows on low $P_{int}$) | Auto-arm after $P_1 > 1.0$ kPa; manual close |
| I4 | Freeze risk | $T_{src,out} < 3\,^\circ\mathrm{C}$ for 30 s | Enter freeze mode §9.6.3; trip LP if < 1.5 °C | Auto when > 4 °C |
| I5 | Loss of water flow (either loop) | Flow switch open 5 s | Trip both compressors (condensing without process flow drives $P_4$ through E9 in seconds) | Manual |
| I6 | Surge detected | $\Pi$–$\dot V$ point crosses surge line, or discharge-pressure oscillation > 2 Hz band energy threshold | Open bypass full; cut speed 20 %; trip after 3 events/10 min | Auto (bypass), manual (trip) |
| I7 | Hotwell level low-low | < 10 % | Trip compressors (protects pump NPSH, prevents vapor into main valve) | Manual |
| I8 | **Air-ingress detect**: $P_{meas} - P_{sat}(T_{meas}) > 150$ Pa sustained 10 min at the condenser (noncondensables partial pressure) | — | Alarm → automatic **purge sequence**: run vacuum/purge pump on condenser top vent until discrepancy < 50 Pa; trip if purge fails twice | Auto |
| I9 | Sensor plausibility | Any of: $T_1 < T_{sat}(P_1) - 1$ K (impossible), rate-of-change beyond physical bound, redundant-pair disagreement > 3 σ | Freeze affected loop at last output, alarm; trip on second confirmed failure | Manual |

Every interlock row exists in the PLC (L1) and rows I1, I2, I5, I7 additionally in hardwired
form (L0) per §9.5.1.

---

## 9.8 Closed-loop verification scenarios

To be executed against the nonlinear dynamic model of [doc 08](08-dynamic-model.md) (and later
on the rig per [doc 10](10-prototype-design.md)); the interactive step-response demo in the
project HTML page ([`html/index.html`](../html/index.html), "control" tab) runs scenario V1
live in the browser with adjustable $K_c$, $\tau_I$.

| # | Scenario | Stimulus | Pass criteria |
|---|----------|----------|---------------|
| V1 | Load step | $\dot m_{proc}$: 100 % → 50 % step at $t = 0$ | $\lvert T_{proc,out} - 120 \rvert \le 0.5$ K within 600 s; overshoot ≤ 1.0 K; $SH_1 \ge 1$ K throughout; no envelope violation |
| V2 | Source ramp | $T_{src,in}$: 20 → 15 °C over 600 s | Same $T_{proc,out}$ band; $P_1 \ge 0.9$ kPa; freeze mode NOT triggered ($T_{src,out} > 3\,^\circ\mathrm{C}$); COP degradation matches [doc 07](07-performance-cop-optimization.md) sensitivity within 5 % |
| V3 | Setpoint change | $T_{proc,out}^{sp}$: 120 → 118 °C ramp 1 K/min | Tracking error ≤ 0.5 K during ramp; no $u_{iv}$/$u_{mv}$ saturation |
| V4 | Actuator saturation | 100 % load + $T_{src,in} = 15\,^\circ\mathrm{C}$ (demand exceeds capacity, $N$ pins at $N_{max}$) | Anti-windup verified: on load relief, $N$ leaves the limit within one $\tau_I$ with no overshoot ringback; $T_5 \le 250\,^\circ\mathrm{C}$ held by spray override; graceful sag of $T_{proc,out}$ with alarm, no trip |

Acceptance across all scenarios: settling to the ±0.5 K band, overshoot ≤ 1 K, $SH_1 \ge 1$ K
at every sample, zero envelope-constraint violations, and no valve or VFD saturated for more
than 30 s continuously in V1–V3 (saturation is the *subject* of V4). Failure of V1 or V2 at the
30 % load gain-schedule point is the trigger for adding the L3 MPC layer (§9.5.5).

---

*Next: [doc 10 — Prototype design](10-prototype-design.md) sizes the actuators, sensors
(including the sub-kPa absolute-pressure transmitters this chapter depends on), and the
commissioning tests that execute §9.6 and §9.8 on hardware.*
