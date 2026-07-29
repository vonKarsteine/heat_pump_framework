# Lumped-Parameter Dynamic Model and Linearization

*This document builds the control-oriented transient model (瞬态仿真) of the two-stage R718 heat pump: a lumped-parameter state-space system $\dot x = f(x, u, d)$ with ~10 states, derived from mass and energy conservation with IF97 property closures, then linearized at the nominal anchor for the control design of [doc 09](09-control-design.md). Conventions and anchors per the [README](../README.md); component models from [docs 03](03-compressor-models.md)–[05](05-valves-mixing-junctions.md).*

---

## 8.1 Modeling goal

The model must predict *dominant dynamics* — how supply temperature, pressures and superheat move over seconds-to-minutes when speeds, valves or boundary conditions change — accurately enough to design and tune controllers. It is **not** CFD: spatial detail inside exchangers is collapsed to lumps. Requirements:

1. **Smoothness**: $f \in C^1$ in $(x, u, d)$ so linearization and MPC are well-posed — no if/else discontinuities in the physics path.
2. **Correct time-constant ordering** (validated against step tests, [doc 10](10-prototype-design.md) Phase 3).
3. **Modest state count** (~10): enough for the physics, small enough for real-time simulation and observer design.

## 8.2 Time-scale audit

| Phenomenon | Time scale | Treatment |
|---|---|---|
| Compressor rotor + motor torque loop | 50–500 ms | **Quasi-static map** $\dot m(N, P_{suc}, P_{dis})$; $N$ follows its VFD ramp as an input filter |
| Valve actuator travel | ~1 s | First-order lag on $u_{mv}, u_{iv}$ (or algebraic if the positioner is fast) |
| Vapor pressure dynamics (manifold + shells) | 1–10 s | **States** — large vacuum volumes give surprisingly slow pressure modes despite tiny mass |
| HX refrigerant + metal thermal storage | 30–300 s | **States** — the dominant modes |
| Secondary water loops (incl. transport delay) | 10–100 s | **States** (outlet temps) + dead time where piping is long |
| Economizer / recuperator holdup | < few s, small mass | **Quasi-static** (algebraic balances of docs 04–05) |

Justification of quasi-static compressors/valves: their internal settling (< 1 s) is at least an order faster than the fastest retained state (pressure, ~2 s); interaction is negligible (singular-perturbation argument), and their *actuator ramps* are kept as explicit input filters so rate limits still bind in simulation.

## 8.3 Moving-boundary vs fixed-zone: the choice and its justification

Two standard families for HX transients:

- **Moving-boundary (MB)**: track zone interface positions (superheated/two-phase/subcooled lengths) as states. Sharp superheat dynamics for DX coils; but zone appearance/disappearance requires switching logic that breaks smoothness exactly where this plant lives.
- **Fixed-zone lumped / finite-volume (FV)**: fixed control volumes, averaged properties.

**Chosen: fixed-zone lumped**, for three plant-specific reasons:

1. The falling-film vacuum evaporator has **no meaningful two-phase length to track**: the incoming flash mixture is > 99.9 % vapor *by volume* ([doc 05](05-valves-mixing-junctions.md) §5.2.2) and evaporation happens from a recirculated film at essentially uniform shell pressure — the MB "interface position" does not exist physically.
2. The condenser desuperheat zone **never vanishes** (15–25 % of duty at all loads, [doc 04](04-heat-exchanger-models.md) §4.2), so the MB advantage (capturing zone birth/death) buys nothing; a static 3-zone split with the zone-boundary enthalpies pinned to saturation is both simpler and smooth.
3. Lumped volumes give globally smooth $f(x,u,d)$ — prerequisite for the linearization (§8.7) and the offset-free MPC option of [doc 09](09-control-design.md).

Honest caveat: for dry-expansion coil evaporators MB models track evaporator-exit superheat transients more sharply; if a later prototype variant used a DX arrangement, the evaporator block should be revisited (the rest of the model is unaffected).

## 8.4 State selection

| # | State | Symbol | Unit | Physical meaning |
|---|-------|--------|------|------------------|
| 1 | Evaporator pressure | $P_e$ | kPa | Vapor mass storage in evaporator shell + suction volume; equivalently $T_e = T_{sat}(P_e)$ |
| 2 | Evaporator liquid holdup | $M_{l,e}$ | kg | Film + sump inventory (level) |
| 3 | Evaporator wall/metal temp | $T_{w,e}$ | °C | Tube + shell thermal mass |
| 4 | Condensing pressure | $P_c$ | kPa | Vapor storage in condenser + discharge volume |
| 5 | Condenser liquid holdup | $M_{l,c}$ | kg | Condensate inventory (drains to economizer) |
| 6 | Condenser wall temp | $T_{w,c}$ | °C | Tube + shell thermal mass |
| 7 | Interstage manifold pressure | $P_{int}$ | kPa | The emergent pressure of [doc 06](06-cycle-coupling-steady-state.md) §6.4, now dynamic |
| 8 | Interstage enthalpy | $h_4$ | kJ/kg | Mixed-stream energy storage in the manifold volume |
| 9 | Source outlet temp | $T_{src,out}$ | °C | Water-side thermal lag, evaporator |
| 10 | Process outlet temp | $T_{proc,out}$ | °C | Water-side thermal lag, condenser (the primary CV) |

Economizer and recuperator are algebraic (docs 04–05 balances evaluated at current states). Superheat $SH_1 = T_1 - T_{sat}(P_e)$ is an *output*, algebraic in $(P_e, M_{l,e}, T_{w,e})$ through the film-dryout relation of §8.5.

## 8.5 Conservation ODEs — full derivation for the evaporator, compact for the rest

### 8.5.1 Evaporator control volume

Take the shell vapor space (volume $V_e$, vapor mass $M_v = V_e\,\rho_v$) and the liquid film/sump ($M_{l,e}$) as two coupled lumps.

**Vapor mass:**

$$ V_e\,\frac{d\rho_v}{dt} = \dot m_{evap} - \dot m_1 \tag{8.1} $$

with $\dot m_{evap}$ the film evaporation rate ($= \dot Q_{evap}/h_{fg}(P_e)$) and $\dot m_1$ the compressor draw. With $\rho_v = \rho_g(P_e)$ on the saturation line:

$$ \frac{dP_e}{dt} = \frac{\dot m_{evap} - \dot m_1}{V_e\,\left(d\rho_g/dP\right)_{sat}} \tag{8.2} $$

The derivative $(d\rho_g/dP)_{sat}$ comes from IF97 ([doc 02](02-water-properties.md)); at 1.228 kPa, $\rho_g \approx 0.0094$ kg/m³ and the ideal-gas estimate $d\rho_g/dP \approx \rho_g/P \cdot (1 - \frac{P}{h_{fg}\rho_g}\frac{dh_{fg}}{dP}\ldots) \approx 7.4\times10^{-3}$ kg/(m³·kPa) is adequate. **Note the scale**: with $V_e \approx 1$ m³, a 1 % flow imbalance (~10⁻⁴ kg/s) moves $P_e$ by only ~0.013 kPa/s ≈ 1 % of $P_1$ per second — the vacuum-side pressure mode has $\tau \approx$ seconds despite the minuscule vapor mass, because $d\rho/dP$ is also minuscule.

**Liquid mass:**

$$ \frac{dM_{l,e}}{dt} = \dot m_e^{valve}(u_{mv}) \,(1 - x_{11}) + \dot m_{recirc,ret} - \dot m_{evap,net} \tag{8.3} $$

(flash vapor $x_{11}\dot m_e^{valve}$ joins the vapor space directly). $M_{l,e}$ is the level the flooded-side controller or the falling-film sump pump manages.

**Energy / wall:** film and wall exchange with source water:

$$ M_{w,e}\,c_{p,w,metal}\,\frac{dT_{w,e}}{dt} = \dot Q_{src \to w} - \dot Q_{w \to film},\qquad
\dot Q_{w \to film} = \alpha_{ff} A_e (T_{w,e} - T_{sat}(P_e)) \tag{8.4} $$

$$ \dot Q_{src \to w} = \dot m_{src} c_{p,w}(T_{src,in} - T_{src,out}),\qquad
(\rho V c_p)_{src}\frac{dT_{src,out}}{dt} = \dot m_{src} c_{p,w}(T_{src,in} - T_{src,out}) - \dot Q_{src \to w} \tag{8.5} $$

### 8.5.2 The general (P, h) transformation

For volumes where the fluid is *not* pinned to the saturation line (the interstage manifold, superheated zones), conservation is written in $(M, U)$ and transformed to the numerically friendly pair $(P, h)$. With $M = \rho V$, $U = M h - P V$ (flow work included):

$$ \frac{dM}{dt} = \Sigma \dot m, \qquad \frac{dU}{dt} = \Sigma (\dot m h)_{ports} + \dot Q \tag{8.6} $$

$$
V\begin{bmatrix}
\left.\dfrac{\partial \rho}{\partial P}\right|_h & \left.\dfrac{\partial \rho}{\partial h}\right|_P \\[2mm]
h\left.\dfrac{\partial \rho}{\partial P}\right|_h - 1 & \; h\left.\dfrac{\partial \rho}{\partial h}\right|_P + \rho
\end{bmatrix}
\begin{bmatrix} \dot P \\ \dot h \end{bmatrix}
=
\begin{bmatrix} \Sigma \dot m \\ \Sigma (\dot m h)_{ports} + \dot Q \end{bmatrix} \tag{8.7}
$$

The 2×2 matrix is inverted analytically each step; the property partials $\left.\partial\rho/\partial P\right|_h,\ \left.\partial\rho/\partial h\right|_P$ come from IF97 Region-2 derivatives (analytic Gibbs-energy second derivatives, [doc 02](02-water-properties.md) §2.5) — no finite differencing inside the ODE right-hand side, preserving smoothness.

**Interstage manifold** (volume $V_m$, states $P_{int}, h_4$): apply Eq. (8.7) with ports = LP discharge in ($\dot m_e, h_3$), injection in ($\dot m_{inj}, h_{10}$), spray in ($\dot m_{spray}, h_7$, with an evaporation lag $\tau_{evap} \approx 0.5$–1 s on its enthalpy delivery), HP suction out ($\dot m_c, h_4$). This volume is where the emergent-$P_{int}$ physics of [doc 06](06-cycle-coupling-steady-state.md) §6.4 lives dynamically: Eq. (6.2)'s residual $\Phi$ is exactly the $\Sigma \dot m$ term driving $\dot P_{int}$.

**Condenser**: same pattern as the evaporator with condensation replacing evaporation; the 3-zone split of [doc 04](04-heat-exchanger-models.md) is retained with fixed zone volumes and the DSH-zone gas treated by Eq. (8.7).

## 8.6 Algebraic layer and DAE structure

Around the 10 ODEs sits the algebraic layer: compressor maps $\dot m_i(N_i, P_{suc}, P_{dis})$ and $h_{dis}$ ([doc 03](03-compressor-models.md)), valve orifices ([doc 05](05-valves-mixing-junctions.md)), economizer/recuperator balances ([doc 04](04-heat-exchanger-models.md)), and IF97 property calls. Every algebraic unknown is *explicitly* computable from the states and inputs (the design-mode march of [doc 06](06-cycle-coupling-steady-state.md) §6.7 evaluated at the current dynamic states), so the system is a **semi-explicit index-1 DAE that reduces to a pure ODE by substitution** — no nonlinear algebraic loops remain at runtime. Solver guidance:

- Stiff BDF integrator (ode15s/CVODE class); stiffness ratio ~100 (pressure vs thermal modes).
- Reuse the Jacobian across steps; supply the analytic property partials.
- Event handling only for hard envelope hits in scenario tests (e.g. $P_e$ crossing the triple-point guard), not for normal operation.

## 8.7 Linearization at the nominal anchor

With $f$ smooth, linearize about the anchor equilibrium $(x^0, u^0, d^0)$:

$$ \delta\dot x = A\,\delta x + B\,\delta u + E\,\delta d, \qquad
A = \left.\frac{\partial f}{\partial x}\right|_0,\ B = \left.\frac{\partial f}{\partial u}\right|_0,\ E = \left.\frac{\partial f}{\partial d}\right|_0 \tag{8.8} $$

**Block structure of $A$** (states ordered as §8.4): near-block-diagonal by component — evaporator block (1–3, 9), condenser block (4–6, 10), manifold block (7–8) — with sparse coupling entries through the machine flows ($\partial \dot m_1/\partial P_e$, $\partial \dot m_c/\partial P_{int}$, $\partial \dot m_2/\partial P_c$) and the valve flows. Those coupling entries *are* the steady-state process gains the control pairing of [doc 09](09-control-design.md) §9.3 is built on.

**Expected eigenvalue pattern** (confirmed by the parameterized model; orders of magnitude, not decimals):

| Mode group | $\tau = -1/\mathrm{Re}(\lambda)$ | Physics |
|---|---|---|
| 2 slow real modes | 60–300 s | HX metal + water thermal masses (states 3, 6, 9, 10 mixing) |
| 2–3 mid real modes | 2–20 s | Shell/manifold pressure storage (states 1, 4, 7) |
| 1 possible weak oscillatory pair | 10–40 s, ζ ≈ 0.3–0.7 | Evaporator mass–pressure coupling ($M_{l,e} \leftrightarrow P_e$ through film area and valve flow) |
| Fast real | ≤ 1 s | Manifold enthalpy, actuator filters |

**Non-minimum-phase alert.** The transfer $SH_1 \leftarrow u_{mv}$ can carry a **right-half-plane zero** (inverse response): opening the main valve first floods the film and *raises* evaporation (pressure and apparent superheat move one way) before the added throughput settles the superheat lower. Physically: mass arrives before its thermal consequence. The controller consequence — bandwidth limited by the RHP-zero frequency, derivative action counterproductive — is carried into [doc 09](09-control-design.md) §9.4. Whether the zero is present depends on holdup and $UA$ split; the commissioning step tests decide.

**Outputs for control:** $y = [T_{proc,out},\ SH_1,\ P_{int},\ T_5,\ P_e]^T = C x + $ algebraic read-outs; FOPDT fits of the $y \leftarrow u$ channels (gain, $\tau$, dead time $\theta$) tabulated from this linearization seed the tuning rules and the demo sliders in [`html/index.html`](../html/index.html).

## 8.8 Validation plan hook

Model parameters with real prior uncertainty, and the step test that pins each (executed as [doc 10](10-prototype-design.md) commissioning Phase 3):

| Parameter (uncertainty) | Identifying experiment | Observable |
|---|---|---|
| Wall masses $M_{w,e}, M_{w,c}$ (±20 %) | ±10 % step on $N$ (capacity) | Thermal settling of $T_{proc,out}$, $T_{src,out}$ |
| $UA$ zone split condenser (±30 %) | Load step at fixed water flow | $T_c$ vs $T_{proc,out}$ trajectory shape |
| $\alpha_{ff}$ evaporator (±40 %, §4.6.3) | Source-flow step | $P_e$ and SH response magnitude |
| Manifold volume $V_m$ (±10 %) | $u_{iv}$ step | $P_{int}$ rise time |
| Valve $C_d A(u)$ curves (±15 %) | Staircase on $u_{mv}, u_{iv}$ at fixed speeds | Flow (Coriolis) vs opening |
| Spray evaporation lag $\tau_{evap}$ (±100 %) | $u_{spray}$ step | $T_4$ (fast) and $T_5$ (delayed) |

Identification protocol: 1 Hz logging minimum, ±10 % perturbations around the anchor (small enough for linearity, large enough for signal/noise), PRBS optional for the fast channels; fit FOPDT/SOPDT per channel and reconcile with Eq. (8.8) by adjusting the physical parameters, not the fitted curves — the model stays physical, the data stays sovereign.
