# Expansion Valves, Injection Mixing, and Junction Equations

*This document models the two throttling devices — main expansion valve (主膨胀阀) and secondary/injection valve (次膨胀阀) — the interstage injection mixing node (中间补气混合点), and the optional-but-practically-mandatory liquid-spray desuperheater. It closes with the formal node-equation set consumed by the system assembly in [doc 06](06-cycle-coupling-steady-state.md). Conventions and anchors per the [README](../README.md).*

---

## 5.1 Isenthalpic throttling model

A throttling device exchanges no work and (over its short length) negligible heat. The steady-flow energy balance between inlet *in* and outlet *out*,

$$ h_{in} + \tfrac{1}{2}c_{in}^2 = h_{out} + \tfrac{1}{2}c_{out}^2 \tag{5.1} $$

reduces to the isenthalpic model

$$ h_{out} = h_{in} \tag{5.2} $$

provided kinetic energy stays negligible. That deserves a check for R718 because the main valve discharges into deep vacuum: the outlet mixture at $x_{11} \approx 0.08$, $P_1 = 1.228$ kPa is > 99.9 % vapor **by volume** (§5.3), so outlet velocity is set by vapor volume flow. With a generous outlet header ($c_{out} \lesssim 30$ m/s), $\tfrac12 c^2 \approx 0.45$ kJ/kg against $h \approx 244$ kJ/kg and enthalpy differences of hundreds of kJ/kg — a < 0.5 % effect, acceptable inside the model's accuracy. (Inside the valve throat itself velocities are far higher; Eq. (5.2) applies between upstream and *fully recovered* downstream stations.)

---

## 5.2 Main expansion valve (主膨胀阀): 8 → 11

### 5.2.1 Outlet state and flash quality

With $h_{11} = h_8$ and the outlet at $P_1$:

$$ x_{11} = \frac{h_8 - h_f(T_e)}{h_{fg}(T_e)} \tag{5.3} $$

Worked values at $T_e = 10$ °C ($h_f = 42.0$ kJ/kg, $h_{fg} = 2477.2$ kJ/kg):

| Liquid state entering the valve | $h_8$ (kJ/kg) | $x_{11}$ |
|---|---|---|
| Saturated at $T_c = 125$ °C (no economizer, no recuperator) | 525.0 | **0.195** ≈ the canonical 0.186 at 120 °C reference |
| Subcooled to 60 °C by the economizer chain | 251.2 | **0.084** |

The canonical pair $x_{11}: 0.186 \to 0.084$ ([README](../README.md)) quantifies why the liquid heat-recovery chain matters: flash vapor produced at the valve has already "used up" its latent heat and transits the evaporator uselessly.

### 5.2.2 Volumetric quality — a design landmine

Mass quality hides the real picture at 1.2 kPa. The void (volume) fraction of the outlet mixture is

$$ \alpha_{void} = \frac{x\,v_g}{x\,v_g + (1-x)\,v_f}
= \frac{0.084 \times 106.3}{0.084 \times 106.3 + 0.916 \times 0.001} \approx 0.9999 \tag{5.4} $$

($v_g/v_f \approx 10^5$.) The "two-phase" stream is thus **> 99.9 % vapor by volume**: outlet piping, the evaporator distributor and the demister see essentially a vapor duct carrying a fine liquid mist. Consequences: (i) distributor design must spread the *liquid* evenly though it occupies almost no volume; (ii) outlet pipe sizing follows vapor-velocity limits (see the 100 Pa ≈ 1.2 K rule, [doc 02](02-water-properties.md) §2.3); (iii) slug flow is impossible, but droplet carryover is a first-order concern.

### 5.2.3 Flow-rate constitutive relation

The valve is also a *flow element*; the control system meters $\dot m_e$ through it. The incompressible orifice equation with opening-dependent area $A(u_{mv})$:

$$ \dot m_e = C_d\, A(u_{mv})\, \sqrt{2\rho_{in}\,\Delta P},\qquad \Delta P = P_4 - P_1 \tag{5.5} $$

Engineering (valve-industry) form, equivalent physics:

$$ \dot m = 
N_6\, K_v(u)\, \sqrt{\rho_{in}\, \Delta P} \tag{5.6} $$

with $K_v$ the metric flow coefficient. **Caveats specific to this duty:**

- **Flashing/choked flow.** The pressure ratio across the valve is ~189; downstream pressure is far below the liquid's saturation pressure at inlet temperature, so the flow chokes once vapor forms in the throat (two-phase critical flow). Beyond choking, $\dot m$ becomes independent of $P_1$ and Eq. (5.5) is evaluated with the *critical* pressure drop $\Delta P_{crit} \approx P_4 - P_{sat}(T_8)$ (homogeneous-equilibrium-model, HEM, approximation). Since $T_8 \approx 58$ °C gives $P_{sat}(T_8) \approx 18$ kPa, effectively the full upstream pressure drives the flow and choking is the *normal* operating condition.
- **Subcooled inlet is mandatory**: minimum ~3–5 K subcooling at state 8 (guaranteed by the recuperator) prevents pre-flashing upstream of the seat, which would collapse $\rho_{in}$ and the flow authority.
- **Turndown**: load range 30–100 % ⇒ $K_v$ turndown ≥ 5:1 with equal-percentage trim; sizing numbers in [doc 10](10-prototype-design.md) §10.4 (the required $K_v \approx 0.03$ is *micro*-valve territory — a reminder that ṁ_e ≈ 0.012 kg/s of liquid is a trickle even though the vapor it becomes is ~1.3 m³/s).

---

## 5.3 Secondary / injection valve (次膨胀阀): 7 → 9

Same physics at milder ratio: $\Delta P = P_4 - P_{int} = 232.2 - 17 \approx 215$ kPa, outlet two-phase at $P_{int}$:

$$ h_9 = h_7, \qquad x_9 = \frac{h_7 - h_f(P_{int})}{h_{fg}(P_{int})} \approx \frac{257 - 236}{2366} \approx 0.009 \tag{5.7} $$

— the branch enters the economizer cold side almost entirely liquid (as it should: its job is to *evaporate* there, absorbing $h_{10} - h_9 \approx 2345$ kJ/kg). Flow relation identical to Eq. (5.5) with opening $u_{iv}$; the same subcooled-inlet requirement applies at the tap — and it is met with room to spare: the branch line runs at $P_4$, against which state 7 is hugely subcooled ($T_{sat}(P_4) - T_7 \approx 63$ K), so there is no pre-flash risk upstream of the valve.

The injection valve's control role differs from the main valve's: it trims the economizer approach / injection superheat, and through it the interstage temperature relief — see the pairing analysis in [doc 09](09-control-design.md) §9.3.

---

## 5.4 Injection mixing node (中间补气混合点)

Between the LP discharge (state 3) and HP suction (state 4) the injection vapor (state 10) merges into the interstage manifold. The node has no storage, no work, no heat loss (insulated manifold), so at steady state:

**Mass:**

$$ \dot m_c = \dot m_e + \dot m_{inj} \tag{5.8} $$

**Energy — an *enthalpy* balance, never a temperature average:**

$$ \dot m_e\, h_3 + \dot m_{inj}\, h_{10} = \dot m_c\, h_4 \tag{5.9} $$

**Pressure closure (manifold assumption):**

$$ P_2^{(dis)} = P_3 = P_5^{(inj)} = P_{int} \tag{5.10} $$

(the task figure's labels $P_2, P_3, P_5$ all denote the interstage level; the model treats them as one manifold pressure — pressure losses between the three ports are folded into the compressor maps).

Dividing Eq. (5.9) by $\dot m_e$, with $r = \dot m_{inj}/\dot m_e$:

$$ h_4 = \frac{h_3 + r\,h_{10}}{1 + r} \tag{5.11} $$

**Worked number (why injection alone is not enough).** At the anchor: $h_3 \approx 3205$ kJ/kg ($T_3 \approx 370$–390 °C), $h_{10} \approx 2602$ kJ/kg (saturated vapor at 56.4 °C), $r = 0.13$:

$$ h_4 = \frac{3205 + 0.13 \times 2602}{1.13} \approx 3136\ \text{kJ/kg}
\;\Rightarrow\; T_4 \approx 340\ ^\circ\text{C} $$

The mix cools by only **~30 K**. Temperature-wise the cold injection stream is 300 K colder, but it carries just 12 % of the flow — the enthalpy balance (5.11), not intuition about temperatures, gives the honest answer. Compressing 340 °C vapor through another ratio-13.7 stage would drive state 5 toward ~1000 °C-class temperatures ([doc 03](03-compressor-models.md) §3.5): unacceptable. Hence §5.5.

---

## 5.5 Liquid-spray desuperheater — optional component, practically mandatory

### 5.5.1 Model

Spray a controlled liquid flow $\dot m_{spray} = w_s\,\dot m_e$, drawn at state 7 ($h_7$), into the interstage manifold, targeting an HP-suction temperature just above saturation:

$$ T_4^{set} = T_{sat}(P_{int}) + \Delta T_{sh,4}, \qquad \Delta T_{sh,4} \approx 10\text{–}15\ \text{K} \tag{5.12} $$

(enough margin that complete droplet evaporation upstream of the HP inlet is guaranteed — wet ingestion at high tip speed erodes impellers). The extended node balance:

$$ \dot m_e h_3 + \dot m_{inj} h_{10} + \dot m_{spray} h_7 = (\dot m_e + \dot m_{inj} + \dot m_{spray})\, h_4^{set} \tag{5.13} $$

### 5.5.2 Coupled solution with the economizer (explicit, linear)

With spray active the condenser stream is $\dot m_c = \dot m_e(1 + r + w_s)$ and the economizer hot side (which carries $\dot m_c$) sees more flow, so $r$ and $w_s$ couple. Both balances are *linear* in $(r, w_s)$ and solve in closed form. Define, per unit $\dot m_e$, from Eq. (5.13):

$$ w_s = a + b\,r, \qquad
a = \frac{h_4^{set} - h_3}{h_7 - h_4^{set}}, \qquad
b = \frac{h_4^{set} - h_{10}}{h_7 - h_4^{set}} \tag{5.14} $$

Substituting into the economizer balance $(1 + r + w_s)(h_6 - h_7) = r\,(h_{10} - h_7)$ ([doc 04](04-heat-exchanger-models.md), Eq. 4.16 with $h_9 = h_7$):

$$ r = \frac{(1 + a)\,(h_6 - h_7)}{(h_{10} - h_7) - (1 + b)\,(h_6 - h_7)} \tag{5.15} $$

**Worked numbers** (anchor: $h_3 \approx 3250$ kJ/kg for the recuperated $T_2 \approx 20$ °C suction case, $h_{10} = 2602$, $h_7 = 257$, $h_6 = 525$, $h_4^{set} = h(P_{int}, 71.4\,^\circ\text{C}) \approx 2632$ kJ/kg):

$$ a = \frac{2632 - 3250}{257 - 2632} \approx 0.26,\qquad b = \frac{2632 - 2602}{257 - 2632} = -0.013 $$

$$ r = \frac{1.260 \times 268}{2345 - 0.987 \times 268} \approx 0.16, \qquad w_s = a + b r \approx 0.26 $$

so the condenser stream is $\dot m_c \approx 1.42\,\dot m_e$ — matching the canonical $w_s \approx 0.25$–0.28 band and the README mass-flow row ($\dot m_e \approx 0.012$, $\dot m_c \approx 0.017$ kg/s at 50 kW). **≈ 25 % of the main flow must be sprayed** to tame the interstage — spray is not a trim, it is a first-class stream that the compressor-2, condenser and valve sizing must all carry.

### 5.5.3 Practical constraints

- **Evaporation length**: droplet residence at ~5–15 m/s manifold velocity vs 20–50 µm droplet evaporation times sets a minimum straight length (order 1 m) or a dedicated spray chamber upstream of the HP inlet.
- **Carryover protection**: $\Delta T_{sh,4}$ setpoint (Eq. 5.12) is the protective margin; the high-$T_5$ override of [doc 09](09-control-design.md) opens the spray valve ($u_{spray}$) as its first action.
- **Where the heat goes**: spray desuperheating is *internal* heat recovery — the absorbed superheat re-enters the cycle as extra vapor to compress in stage 2 (that is why $\dot m_c > \dot m_e(1+r)$), not a loss; its COP effect is analyzed in [doc 07](07-performance-cop-optimization.md) §7.4.

---

## 5.6 Formal node-equation set for the assembly

Sign convention: flows positive in the direction of the arrows of the [doc 01](01-system-overview.md) schematic; every node equation is written as $\Sigma(\text{in}) - \Sigma(\text{out}) = 0$.

**N1 — Liquid split at state 7** (tap for injection branch and spray line):

$$ \dot m_7^{in} - \dot m_e - \dot m_{inj} - \dot m_{spray} = 0, \qquad \dot m_7^{in} = \dot m_c \tag{5.16} $$

$$ h\ \text{uniform at the node}: h_{7,e} = h_{7,inj} = h_{7,spray} = h_7 \tag{5.17} $$

**N2 — Main valve 8 → 11**: $\ h_{11} = h_8$; flow by Eq. (5.5) with $u_{mv}$. **(2 eqs)**

**N3 — Secondary valve 7 → 9**: $\ h_9 = h_7$; flow by Eq. (5.5) with $u_{iv}$. **(2 eqs)**

**N4 — Interstage mixing (+ optional spray)**: Eqs. (5.8), (5.13) with $h_4^{set}$ replaced by the free unknown $h_4$ when the spray controller is inactive ($\dot m_{spray} = 0$ ⇒ Eq. 5.11). **(2 eqs)**

**N5 — Manifold pressure closure**: Eq. (5.10). **(1 eq)**

Together with the four heat-exchanger blocks ([doc 04](04-heat-exchanger-models.md) §4.8) and the two compressor blocks ([doc 03](03-compressor-models.md) §3.8), these nine relations complete the component library; [doc 06](06-cycle-coupling-steady-state.md) counts them against the unknowns and assembles the closed system.
