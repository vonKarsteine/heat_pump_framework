# 03 — Compressor Models (Both Stages)

*Derives the steady-state models of the low-pressure compressor (压缩机-1, 第一级压缩) and the high-pressure compressor (压缩机-2, 第二级压缩): mass-flow and volumetric-efficiency relations for positive-displacement machines, the energy balance giving specific work, discharge enthalpy and discharge temperature, worked numbers at the nominal design point, the turbomachine (map-based) alternative, and the operating-envelope inequalities. These models are consumed by the cycle assembly in [doc 06](06-cycle-coupling-steady-state.md), the optimizer in [doc 07](07-performance-cop-optimization.md), the override controllers in [doc 09](09-control-design.md), and the machine selection in [doc 10](10-prototype-design.md).*

---

## 3.1 Fixed compressor parameters (task class 1) and their model role

The task sheet's first input-variable class, "component performance parameters that cannot be changed" (无法改变的部件性能参数), lists for the compressors:

| Task-sheet item (Chinese) | Symbol | Role in this document |
|---|---|---|
| Isentropic efficiency (等熵效率) | $\eta_{is}$ | Converts ideal to actual specific work, §3.4 |
| Volumetric efficiency (容积效率) | $\eta_{vol}$ | Scales swept volume to delivered flow, §3.2–3.3 |
| Mechanical efficiency (机械效率) | $\eta_{mech}$ | Shaft-to-gas power conversion, §3.4 |
| Displacement (排量) or max swept volume | $V_{disp}$ | Sets capacity of a positive-displacement machine, §3.2 |
| Rotor / motor characteristics (转子/电机特性) | inertia $J$, $\eta_{motor}(N,\tau)$ | Electrical power (§3.4); dynamics in [doc 08](08-dynamic-model.md) |
| Maximum allowed speed or torque (允许的最大转速或转矩限值) | $N_{max}$, $\tau_{max}$ | Envelope inequalities, §3.7 |

Once a machine is purchased these are **frozen constants or vendor-supplied maps** — the model may only *evaluate* them, never optimize them (contrast with class-2 design variables such as heat-exchanger area, and class-3 runtime inputs such as the speeds $N_1$, $N_2$, which are the principal control handles of [doc 09](09-control-design.md)).

Both compressor models have the same interface. Per the canonical numbering in the [README](../README.md):

- **LP stage:** suction = state 2 $(P_1,\,T_2)$, discharge = state 3 $(P_{int})$, flow $\dot m_e$.
- **HP stage:** suction = state 4 $(P_{int},\,T_4)$, discharge = state 5 $(P_4)$, flow $\dot m_c=\dot m_e+\dot m_{inj}$ (plus spray water $w_s\dot m_e$ when the interstage desuperheater of [doc 05](05-valves-mixing-junctions.md) is fitted).

---

## 3.2 Positive-displacement baseline: the mass-flow law

A positive-displacement (PD) machine (piston, screw, scroll, Roots) traps a geometric volume $V_{disp}$ (m³ per revolution) of suction-state gas $N$ times per second. The ideal volumetric intake rate is therefore

$$
\dot V_{ideal} = V_{disp}\,N \qquad \left[\frac{\mathrm{m^3}}{\mathrm{rev}}\cdot\frac{\mathrm{rev}}{\mathrm{s}}=\frac{\mathrm{m^3}}{\mathrm{s}}\right].
$$

Real machines ingest less because of clearance-gas re-expansion, suction-valve pressure drop, internal leakage, and heat pickup in the suction port; all of these are lumped into the volumetric efficiency $\eta_{vol}\le 1$, defined on the *suction* state. The delivered mass flow is the delivered volume flow times the suction density $\rho_{suc}=1/v(P_{suc},T_{suc})$:

$$
\boxed{\;\dot m = \eta_{vol}\,\rho_{suc}\,V_{disp}\,N = \frac{\eta_{vol}\,V_{disp}\,N}{v(P_{suc},T_{suc})}\;}
\tag{3.1}
$$

with $\dot m$ in kg/s, $v$ in m³/kg from the IAPWS-IF97 Region-2 relations of [doc 02](02-water-properties.md). Two structural consequences drive the whole design:

1. **Capacity is proportional to suction density.** At state 2 the density is $\rho_{suc}\approx 1/106\ \mathrm{m^3/kg}\approx 9.4\times10^{-3}\ \mathrm{kg/m^3}$ — three orders of magnitude below ambient air. Equation (3.1) says the LP machine must swallow enormous *volume* to move milligram-scale *mass*: at the nominal $\dot m_e\approx 0.010$–$0.013$ kg/s, the suction volume flow is
$$
\dot V_{suc,1}=\dot m_e\,v_2 \approx 0.0125\times 106 \approx 1.3\ \mathrm{m^3/s}\ (\approx 4\,800\ \mathrm{m^3/h}),
$$
consistent with the README anchor "~1 m³/s (~3 500 m³/h) per 50 kW", which corresponds to the low
end of the $\dot m_e$ range; over the full range 0.010–0.013 kg/s the suction volume flow spans
≈ 1.1–1.4 m³/s. A PD machine delivering this at, say, $N=100$ rev/s and $\eta_{vol}=0.75$ would need $V_{disp}\approx 0.017$ m³/rev — a 17-litre swept volume, an industrial-blower scale machine. The HP stage, by contrast, sees $v_4\approx 9.2\ \mathrm{m^3/kg}$ at $(P_{int}\!\approx\!17\ \mathrm{kPa},\ T_4\!\approx\!65\,^\circ\mathrm{C})$, so $\dot V_{suc,2}\approx 0.017\times9.2\approx0.16\ \mathrm{m^3/s}$ — about **8× smaller volumetrically** than stage 1 despite carrying *more* mass. The two machines are geometrically very different; they cannot be identical units.
2. **Speed $N$ is the natural capacity control.** For fixed suction state, $\dot m\propto N$ — this linearity is what makes $N_1,N_2$ the primary manipulated variables in [doc 09](09-control-design.md).

Also note from (3.1) the sensitivity to suction-line pressure drop: with $\mathrm{d}P_{sat}/\mathrm{d}T\approx 82$ Pa/K at 10 °C ([doc 02](02-water-properties.md)), a mere 100 Pa loss between evaporator (蒸发器) and suction flange lowers $\rho_{suc}$ by ~8 % and simultaneously raises the effective temperature lift by ~1.2 K.

---

## 3.3 Clearance-volume model of $\eta_{vol}$ — and why one PD stage cannot do PR ≈ 13.8

For a reciprocating machine with clearance volume $V_{cl}=c\,V_{disp}$ (clearance fraction $c$, typically 0.03–0.15), the gas left in the clearance at discharge pressure $P_{dis}$ re-expands polytropically (exponent $n\approx k$, the isentropic exponent; $k\approx1.32$ for low-pressure steam) as the piston retreats. Suction gas cannot enter until the clearance gas has re-expanded down to $P_{suc}$, occupying

$$
V_{re} = V_{cl}\left(\frac{P_{dis}}{P_{suc}}\right)^{1/k} = c\,V_{disp}\,\Pi^{1/k},
\qquad \Pi \equiv \frac{P_{dis}}{P_{suc}}.
$$

The fresh charge is $V_{disp}+V_{cl}-V_{re}$, hence

$$
\boxed{\;\eta_{vol} = 1 + c - c\,\Pi^{1/k} = 1 - c\left(\Pi^{1/k}-1\right)\;}
\tag{3.2}
$$

Setting $\eta_{vol}=0$ gives the **maximum attainable pressure ratio** of a single PD stage:

$$
\Pi_{max} = \left(1+\frac{1}{c}\right)^{k}.
\tag{3.3}
$$

At the nominal per-stage pressure ratio $\Pi\approx13.8$ (overall $\approx 189$ split evenly, README), $\Pi^{1/k}=13.8^{0.758}\approx 7.30$:

| Clearance $c$ | $\eta_{vol}$ at $\Pi=13.8$ | $\Pi_{max}$ from (3.3) | Verdict |
|---|---|---|---|
| 0.04 (excellent, special design) | $1-0.04\times6.30 \approx 0.75$ | ≈ 74 | Feasible but demanding |
| 0.10 (typical industrial) | $1-0.10\times6.30 \approx 0.37$ | ≈ 24 | Capacity collapses to ⅓ |
| 0.15 (ordinary/valve-heavy) | $1-0.15\times6.30 \approx 0.05$ | ≈ 15 | Essentially zero delivery |

**Conclusion.** With ordinary clearance fractions a *single* PD compression element cannot realize $\Pi\approx13.8$: the machine spends its stroke re-expanding its own clearance gas. The practical options, developed in [doc 10](10-prototype-design.md), are

- **PD trains**: e.g. two elements in series per thermodynamic stage at $\Pi\approx\sqrt{13.8}\approx3.7$ each, where (3.2) gives a healthy $\eta_{vol}\approx0.83$ even at $c=0.10$; or
- **dynamic (multi-wheel centrifugal/axial) machines**, which have no clearance-re-expansion mechanism at all and are the standard choice for R718 at ≥ 1 m³/s suction flow (§3.6). For these, $\eta_{vol}$ in (3.1) is replaced by the map of §3.6.

Screw and Roots machines follow (3.2) only loosely (their losses are leakage-dominated), but the qualitative collapse of delivered flow with $\Pi$ at low suction density is the same; vendor $\eta_{vol}(\Pi,N)$ tables are used verbatim as task-class-1 data.

---

## 3.4 Energy balance: specific work, power, discharge state

Model the compression as adiabatic (no jacket cooling; heat loss is negligible next to the enormous superheat generated). For suction state $(P_{suc}, h_{in}, s_{in})$ and discharge pressure $P_{dis}$:

**Isentropic reference.** The ideal outlet state has the *same entropy* and the discharge pressure, so

$$
w_{is} = h\!\left(P_{dis},\, s_{in}\right) - h_{in}
\tag{3.4}
$$

evaluated with the IF97 Region-2 backward/forward relations of [doc 02](02-water-properties.md).

**Actual gas work.** The isentropic efficiency (等熵效率) is defined $\eta_{is} \equiv w_{is}/w$, hence

$$
w = \frac{w_{is}}{\eta_{is}}.
\tag{3.5}
$$

**Discharge enthalpy and temperature.** All aerodynamic loss ends up in the gas (adiabatic machine), so the steady-flow energy balance across the machine is simply

$$
h_{out} = h_{in} + w, \qquad T_{out} = T\!\left(P_{dis},\, h_{out}\right),
\tag{3.6}
$$

the last inversion again from Region 2. Note the loss enters *twice*: it costs shaft work **and** it raises $T_{out}$, worsening the discharge-temperature constraint of §3.7.

**Power chain.** Gas (indicated) power, shaft power and electrical power are

$$
\dot W = \dot m\, w, \qquad
\dot W_{el} = \frac{\dot W}{\eta_{mech}\,\eta_{motor}},
\tag{3.7}
$$

where $\eta_{mech}$ (机械效率, bearings/gear/seal friction, dissipated outside the gas path) and $\eta_{motor}$ (from the vendor motor-efficiency curve, a task-class-1 map $\eta_{motor}(N,\tau)$) are typically 0.93–0.97 each. Shaft torque, needed for the torque limit in §3.7 and the rotor dynamics in [doc 08](08-dynamic-model.md), is $\tau = \dot W/(\eta_{mech}\,2\pi N)$.

**Ideal-gas cross-check.** Low-pressure steam is nearly ideal ([doc 02](02-water-properties.md)); with $k=1.32$, $(k-1)/k = 0.242$:

$$
T_{out,is} \approx T_{in}\,\Pi^{0.242} \quad (\text{temperatures in K}),
\tag{3.8}
$$

a one-line sanity check on every IF97 evaluation below.

---

## 3.5 Worked numbers at the nominal design point

Reference case for the LP stage: saturated vapor at $T_e=10$ °C (the recuperator (回热器) superheat of 5–10 K at state 2 raises the discharge temperature by the amplification factor $\Pi^{(k-1)/k}\approx1.9$ per kelvin of suction superheat and is carried exactly by the solver of [doc 06](06-cycle-coupling-steady-state.md)). Properties from IF97.

### Stage 1 (LP): $P_1 = 1.228\ \mathrm{kPa} \rightarrow P_{int} \approx 17\ \mathrm{kPa}$, $\Pi_1 = 13.84$

| Quantity | Value | Source |
|---|---|---|
| Suction $h_{in},\ s_{in},\ v_{suc}$ | 2519.2 kJ/kg, 8.900 kJ/(kg·K), 106.3 m³/kg | sat. vapor, 10 °C |
| Isentropic discharge $T_{3s}$ | ≈ 261 °C | $s=8.900$ at 17 kPa |
| — ideal-gas check (3.8) | $283.15\times13.84^{0.242}=534.8\ \mathrm{K}=261.7$ °C ✓ | |
| $w_{is}$ (3.4) | $\approx 3000-2519 \approx 480$ kJ/kg | |
| $w$ at $\eta_{is}=0.70$ (3.5) | ≈ 685 kJ/kg | |
| $h_3$, $T_3$ (3.6) | ≈ 3205 kJ/kg, **≈ 370 °C** | README anchor |
| $\dot m_e$ (nominal) | ≈ 0.0125 kg/s | README range 0.010–0.013 |
| $\dot W_1$ (3.7) | ≈ 8.6 kW gas power | |
| Suction volume flow | ≈ 1.3 m³/s (≈ 4800 m³/h) | eq. (3.1) discussion |

With the recuperator superheat included ($T_2\approx15$–20 °C), (3.8) gives $T_{3s}\approx 271$–281 °C — since $T_{out,s}=T_{in}\,\Pi^{0.242}$, each kelvin of suction superheat adds $\Pi^{0.242}\approx1.9$ K, i.e. roughly **+1.9 K of discharge temperature per +1 K of suction superheat**. The recuperator is still worthwhile because its COP benefit comes from liquid subcooling (state 7→8), not from the vapor side; see [doc 04](04-heat-exchanger-models.md) and [doc 07](07-performance-cop-optimization.md).

### Stage 2 (HP): $P_{int} \approx 17\ \mathrm{kPa} \rightarrow P_4 = 232.2\ \mathrm{kPa}$, $\Pi_2 = 13.66$

Suction state 4 assumes the interstage spray desuperheater ([doc 05](05-valves-mixing-junctions.md)) has trimmed the mix to ≈ 9 K superheat, $T_4\approx65$ °C at $T_{sat}(17\ \mathrm{kPa})\approx56$ °C. (Without spray, economizer (经济器, 中间补气) vapor injection alone at $r\approx0.11$ cools the 370 °C stage-1 discharge by only ~30 K — the README's argument for why spray is practically mandatory.)

| Quantity | Value | Source |
|---|---|---|
| Suction $h_4$ | ≈ 2620 kJ/kg | $h_g(17\ \mathrm{kPa})\!\approx\!2603$ + 9 K superheat |
| Isentropic discharge $T_{5s}$ | ≈ 364 °C | $s_4\approx8.02$ at 232.2 kPa |
| — ideal-gas check (3.8) | $338.15\times13.66^{0.242}=636.8\ \mathrm{K}=363.7$ °C ✓ | |
| $w_{is}$ | ≈ 580 kJ/kg | |
| $w$ at $\eta_{is}=0.70$ | ≈ 830 kJ/kg | |
| $h_5$, $T_5$ (unconstrained model) | ≈ 3450 kJ/kg, ≈ 480 °C | see remark below |
| $\dot m_c$ (nominal) | ≈ 0.017 kg/s | README range 0.016–0.021 |
| $\dot W_2$ | ≈ 14 kW gas power | |
| Suction volume flow | ≈ 0.16 m³/s | |

Two consequences of the stage-2 row deserve emphasis:

- **The raw discharge temperature (~480 °C) violates any realistic material/lubrication limit** (§3.7). The two-stage *thermodynamic* model concentrates all irreversibility heating at the stage exit; a real machine must break the stage internally (multiple wheels, §3.6, possibly with inter-wheel spray). This is a model *finding*, not a model error — it is exactly why $T_{dis}$ appears as an explicit constraint and why doc 09 carries a discharge-temperature override.
- **Condenser desuperheat share.** With $h_g(125\,^\circ\mathrm{C})=2713$ kJ/kg and $h_f=525$ kJ/kg, the desuperheat duty fraction is $(3450-2713)/(3450-525)\approx25\%$ — the upper end of the README's 15–25 % band, confirming the need for the multi-zone condenser (冷凝器) model of [doc 04](04-heat-exchanger-models.md).

### Cycle-level sanity check

Total gas power $\dot W_1+\dot W_2 \approx 23$ kW; with $\eta_{mech}\eta_{motor}\approx0.88$, electrical input $\approx26$ kW, so

$$
\mathrm{COP} \approx \frac{50}{26} \approx 1.9,
$$

inside the README target band 1.6–1.9 (≈ 55 % of the Carnot value 3.46 for the 283 K → 398 K lift — no configuration of this machine approaches COP 2 at the full 10 → 125 °C lift). Evaporator duty $\approx 50-23\approx27$ kW, matching the README's 22–27 kW. The exact flow split ($r$, $w_s$) and $P_{int}$ come from the coupled solution in [doc 06](06-cycle-coupling-steady-state.md); the per-kg works above are insensitive to that split.

---

## 3.6 Turbomachine alternative: maps, corrected flow, tip-speed limits

At $\dot V_{suc}\gtrsim1$ m³/s and $\Pi\approx13.8$ per stage, industrial R718 practice (MVR and recent HTHP prototypes) uses **dynamic machines** — centrifugal or axial — because they deliver large volume flow from compact rotors, are oil-free (water tolerates no lubricant carry-over), and have no clearance-collapse mechanism. Their steady-state model replaces (3.1)–(3.2) with a **performance map** in corrected variables:

$$
\dot m_{corr} = \dot m\,\frac{\sqrt{T_{suc}/T_{ref}}}{P_{suc}/P_{ref}}, \qquad
N_{corr} = \frac{N}{\sqrt{T_{suc}/T_{ref}}},
\qquad
\Pi = f_{map}\!\left(\dot m_{corr},\, N_{corr}\right),\quad
\eta_{is} = g_{map}\!\left(\dot m_{corr},\, N_{corr}\right),
\tag{3.9}
$$

both maps being task-class-1 vendor data. The map is bounded by the **surge line** (low-flow limit: below it the blade loading stalls and flow reverses violently — an absolute operating prohibition) and the **choke line** (high-flow limit: some internal throat reaches Mach 1 and $\dot m_{corr}$ saturates regardless of back-pressure). Because each compressor imposes its own $\Pi(\dot m)$ characteristic at given speed, the interstage pressure **$P_{int}$ is not a set-point but an emergent equilibrium** — it settles where the LP delivery characteristic intersects the HP swallowing characteristic (README; solved in [doc 06](06-cycle-coupling-steady-state.md), exploited for control in [doc 09](09-control-design.md)).

**How many wheels?** The Euler work of one radial wheel is $\Delta h_{wheel}\approx\lambda\,u_{tip}^2$ with work-input coefficient $\lambda\approx0.55$–0.75. Tip speed is limited by impeller stress (≈ 350 m/s for aluminium, ≈ 450–550 m/s for titanium) and by compressibility: the steam sonic speed at suction is

$$
a=\sqrt{kRT}=\sqrt{1.32\times461.5\times283}\approx415\ \mathrm{m/s} \quad (10\ ^\circ\mathrm{C}),
$$

so $u_{tip}\approx350$–450 m/s keeps inlet relative Mach numbers tolerable. Then $\Delta h_{wheel}\approx0.65\times(350\text{–}450)^2\approx80$–130 kJ/kg per wheel, against the stage demand $w_{is}\approx480$–580 kJ/kg — equivalently a per-wheel pressure ratio of ≈ 1.8–2.5, so each *thermodynamic* stage needs

$$
n_{wheel} = \left\lceil \frac{\ln \Pi_{stage}}{\ln \Pi_{wheel}} \right\rceil
= \left\lceil \frac{\ln 13.8}{\ln(1.8\text{–}2.5)} \right\rceil = 3\text{–}5,
\tag{3.10}
$$

in practice **4–6 wheels per stage** after leaving surge margin. Water's high gas constant ($R=461.5$ J/(kg·K), ~5× R134a) is a double-edged sword: it gives a high sonic speed (fast wheels allowed) but also a huge specific-work demand ($w_{is}\approx480$ kJ/kg vs ~40 kJ/kg for a typical organic refrigerant), which is why R718 machines are multi-wheel trains where an R134a machine would be a single impeller. Machine architecture selection for the 50 kW prototype — including the PD-train fallback of §3.3 — is finalized in [doc 10](10-prototype-design.md).

For system-level modeling both machine families expose the *same* interface: (3.9) or (3.1)–(3.2) supply $\dot m(\Pi,N,\text{suction})$, then §3.4 supplies energy quantities. The cycle solver never needs to know which architecture is inside the box.

---

## 3.7 Operating-envelope constraints (feed-forward to docs 06/07/09)

The optimizer ([doc 07](07-performance-cop-optimization.md)) and the controller ([doc 09](09-control-design.md)) treat the compressor envelope as the inequality set, per stage $j\in\{1,2\}$:

$$
\begin{aligned}
&N_{min,j} \le N_j \le N_{max,j} && \text{(motor/VFD limit, 允许的最大转速)}\\[2pt]
&T_{dis,j} \le T_{max} \approx 250\text{–}300\ ^\circ\mathrm{C} && \text{(impeller creep, seals, O-rings, bearing lube)}\\[2pt]
&\dot m_{corr,j} \ge (1+SM)\,\dot m_{surge}(\Pi_j,N_{corr,j}),\quad SM\approx0.10 && \text{(surge margin, dynamic machines)}\\[2pt]
&\dot m_{corr,j} \le \dot m_{choke}(N_{corr,j}) && \text{(choke)}\\[2pt]
&\tau_j = \dfrac{\dot W_j}{\eta_{mech}\,2\pi N_j} \le \tau_{max}(N_j), \qquad \dot W_{el,j} \le \dot W_{motor,max} && \text{(torque/power curve, 转矩限值)}\\[2pt]
&\Pi_j < \Pi_{max} = (1+1/c)^k && \text{(PD machines only, eq. 3.3)}
\end{aligned}
\tag{3.11}
$$

Notes:

- The **$T_{dis}$ constraint is the binding one** at the nominal lift: §3.5 showed ~370 °C (stage 1) and ~480 °C (stage 2, unconstrained) at $\eta_{is}=0.70$, both above $T_{max}$. Feasibility is restored by the interstage spray ([doc 05](05-valves-mixing-junctions.md)) and by intra-stage measures ([doc 10](10-prototype-design.md)); at runtime, the doc 09 override loop cuts speed or opens the spray valve when measured $T_{dis}$ approaches $T_{max}$.
- **Surge** interacts with the emergent $P_{int}$: a load drop that pushes the HP machine toward surge shifts the characteristic intersection and drags the LP stage with it — the coupled surge-avoidance logic (speed ratio management, hot-gas/injection bypass) is a doc 09 topic, but the constraint itself lives here.
- Each inequality in (3.11) becomes either an optimization constraint (steady-state, doc 07) or an **override controller** with a selector (min/max logic) on the corresponding speed command (doc 09).

---

## 3.8 Model summary box (interface to doc 06)

**Inputs (per stage):** suction state $(P_{suc}, h_{suc})$; discharge pressure $P_{dis}$; speed $N$ (control input, class 3).
**Parameters (class 1):** $\eta_{is}$ (or map $g_{map}$), $\eta_{vol}$ (or eq. 3.2 with $c,k$; or map $f_{map}$), $\eta_{mech}$, $\eta_{motor}(N,\tau)$, $V_{disp}$, $N_{max}$, $\tau_{max}(N)$, $T_{max}$, surge/choke lines.
**Outputs:** $\dot m$, $w$, $\dot W$, $\dot W_{el}$, $\tau$, $h_{out}$, $T_{out}$.

$$
\boxed{
\begin{aligned}
\dot m &= \eta_{vol}\,\frac{V_{disp}\,N}{v(P_{suc},T_{suc})} &&\text{(PD, 3.1–3.2)} \quad\text{or}\quad \dot m = \dot m_{map}(\Pi, N_{corr})\cdot\frac{P_{suc}/P_{ref}}{\sqrt{T_{suc}/T_{ref}}} \;\;\text{(dynamic, 3.9)}\\
w &= \frac{h(P_{dis},s_{suc}) - h_{suc}}{\eta_{is}}, \qquad h_{out} = h_{suc} + w, \qquad T_{out}=T(P_{dis},h_{out}) &&\text{(3.4–3.6)}\\
\dot W &= \dot m\,w, \qquad \dot W_{el} = \frac{\dot W}{\eta_{mech}\,\eta_{motor}} &&\text{(3.7)}\\
&\text{subject to the envelope set (3.11).}
\end{aligned}}
$$

In the cycle assembly of [doc 06](06-cycle-coupling-steady-state.md) this block is instantiated twice — LP: $(P_1,h_2)\!\to\!(P_{int},h_3)$ with $\dot m_e$; HP: $(P_{int},h_4)\!\to\!(P_4,h_5)$ with $\dot m_c$ — and the shared unknown $P_{int}$ is closed by requiring both instances to pass the same mass flow through the interstage node ($\dot m_{HP} = \dot m_{LP} + \dot m_{inj} + \dot m_{spray}$), which is precisely the "characteristics intersection" that makes $P_{int}$ emergent.
