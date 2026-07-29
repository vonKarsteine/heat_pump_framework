# Two-Stage R718 (Water) High-Temperature Heat Pump

**Theoretical derivation and prototype design** for a two-stage vapor-compression high-temperature
heat pump using water (H₂O, refrigerant designation **R718**) as the working fluid. The system
absorbs heat from a ~20 °C source and delivers process heat at ≥ 120 °C (condenser heats process
water from 115 °C to ~120 °C), per the assignment restated in [TASK.md](TASK.md).

> **Interactive companion:** open [`html/index.html`](html/index.html) by double-clicking it —
> it is fully self-contained (no internet needed) and includes a live cycle calculator,
> an interactive T–s diagram, the system schematic, and a control-loop step-response demo.

---

## Document index

| # | Document | Contents |
|---|----------|----------|
| 01 | [System overview](docs/01-system-overview.md) | Background, R718 motivation, corrected state-point map, input-variable taxonomy |
| 02 | [Water properties](docs/02-water-properties.md) | IAPWS-IF97 property relations (Regions 4/2/1), Clausius–Clapeyron, ideal-gas checks |
| 03 | [Compressor models](docs/03-compressor-models.md) | Isentropic/volumetric/mechanical efficiency, mass flow, discharge temperature |
| 04 | [Heat exchanger models](docs/04-heat-exchanger-models.md) | Evaporator, multi-zone condenser, economizer, recuperator; ε-NTU and correlations |
| 05 | [Valves, mixing, junctions](docs/05-valves-mixing-junctions.md) | Isenthalpic throttling, orifice flow, injection mixing node, spray desuperheater |
| 06 | [Cycle coupling & steady state](docs/06-cycle-coupling-steady-state.md) | Full equation assembly, degrees-of-freedom audit, solution algorithms |
| 07 | [Performance & optimization](docs/07-performance-cop-optimization.md) | COP derivation, optimal intermediate pressure, sensitivities, design optimization |
| 08 | [Dynamic model](docs/08-dynamic-model.md) | Lumped-parameter state-space model ẋ = f(x,u,d), linearization |
| 09 | [Control design](docs/09-control-design.md) | Objectives, pairing/RGA, layered PID + overrides, MPC outline, sequencing, interlocks |
| 10 | [Prototype design](docs/10-prototype-design.md) | 50 kW design point, component sizing/selection, instrumentation, commissioning plan |

Reading order = numerical order. Docs 01–07 are the steady-state theory (task item "系统建模" /
system modeling and steady-state analysis); docs 08–09 cover transient simulation and control
(task item "动态响应和控制策略"); doc 10 is the prototype design.

---

## Canonical state-point numbering

> ⚠ **Correction to the task figure.** The diagram in the original task sheet (restated in [TASK.md](TASK.md)) uses the label **T10 twice**:
> once for the economizer vapor outlet (T10, P5) and once for the evaporator inlet (T10, P1).
> Throughout this project the evaporator inlet is renumbered **state 11**; all other labels match
> the original figure.

| State | Location | Pressure level | Phase (nominal) |
|-------|----------|----------------|-----------------|
| 1 | Evaporator vapor outlet | P1 | Slightly superheated vapor |
| 2 | Recuperator cold-side outlet = LP compressor suction | P1 | Superheated vapor |
| 3 | LP compressor discharge | P2 ≈ P_int | Highly superheated vapor |
| 4 | After injection mixing = HP compressor suction | P3 ≈ P_int | Superheated vapor |
| 5 | HP compressor discharge = condenser inlet | P4 | Highly superheated vapor |
| 6 | Condenser liquid outlet | P4 | Saturated/subcooled liquid |
| 7 | Economizer hot-side outlet (branch tap point) | P4 | Subcooled liquid |
| 8 | Recuperator hot-side outlet = main valve inlet | P4 | Subcooled liquid |
| 9 | Secondary valve outlet = economizer cold-side inlet | P5 ≈ P_int | Two-phase |
| 10 | Economizer cold-side (vapor) outlet = injection stream | P5 ≈ P_int | Sat./slightly superheated vapor |
| 11 | Main valve outlet = evaporator inlet *(renumbered, see note)* | P1 | Two-phase |

Adopted convention (stated explicitly because the figure is ambiguous): the liquid branch to the
secondary expansion valve **taps at state 7** (after the economizer hot side, before the
recuperator), so h9 = h7. Pressure closure at the injection node: P2 ≈ P3 ≈ P5 = P_int (interstage
manifold). Tapping at 6 or 8 instead only changes which enthalpy enters the branch equations.

**Mass flows:** ṁ_e (evaporator/main stream, states 11→1→2→3), ṁ_inj (injection stream, 9→10),
ṁ_c = ṁ_e + ṁ_inj (condenser stream, 4→5→6→7). Injection ratio **r = ṁ_inj/ṁ_e**. When the
optional interstage spray desuperheater is fitted, its water w_s·ṁ_e joins upstream of state 4,
so ṁ_c = ṁ_e(1 + r + w_s) — see the anchor table below.

---

## Nominal design point (anchor for all documents)

| Quantity | Value |
|----------|-------|
| Heating capacity Q̇_cond | 50 kW |
| Process water (condenser secondary side) | 115 °C → 120 °C |
| Source water (evaporator secondary side) | 20 °C → ~15 °C |
| Evaporating temperature / pressure | T_e = 10 °C, P1 = 1.228 kPa |
| Condensing temperature / pressure | T_c = 125 °C, P4 = 232.2 kPa (125 °C, not 120 °C, because of the condenser pinch vs 115→120 °C water) |
| Intermediate pressure | P_int ≈ 17–20 kPa (T_sat ≈ 56–60 °C); geometric mean √(P1·P4) = 16.9 kPa |
| Evaporator exit superheat | 3–5 K |
| Isentropic efficiency (per stage) | 0.70 |
| Overall / per-stage pressure ratio | ≈ 189 / ≈ 13.8 |
| Carnot COP (283 K → 398 K) | 3.46 |
| Realistic COP target | ≈ 1.6–1.9 electrical (~50 % of Carnot). The idealized design-mode solve at this exact anchor gives ≈ 1.95; off-design margins, fouling and part-load drive losses bring the practical band down |
| Compressor power / evaporator duty | ≈ 23–28 kW / ≈ 22–27 kW |
| Mass flows | ṁ_e ≈ 0.010–0.013 kg/s; ṁ_c = ṁ_e(1 + r + w_s) ≈ 0.016–0.021 kg/s — the condenser stream carries the main flow **plus** economizer injection (r ≈ 0.11–0.16 depending on approach temperature and spray) **plus** interstage spray (w_s ≈ 0.25 when fitted); the exact split comes from the cycle solver in `html/index.html` |

### Key physical facts that shape everything (verified with IAPWS-IF97)

- **Deep vacuum evaporator:** P1 ≈ 1.23 kPa absolute. Suction specific volume ≈ **106 m³/kg** →
  ~1 m³/s (~3 500 m³/h) of vapor per 50 kW. dP_sat/dT ≈ 82 Pa/K at 10 °C, so 100 Pa of suction-line
  pressure drop costs ≈ 1.2 K of effective lift. Triple point (611 Pa, 0.01 °C) is a hard lower bound.
- **Extreme discharge superheat:** compressing saturated steam from 10 °C at pressure ratio 13.8
  gives an isentropic discharge of ≈ 261 °C, and ≈ 370 °C at η_is = 0.70. Economizer vapor
  injection (r ≈ 0.11–0.13) cools the interstage mix by only ~30–35 K → **liquid-spray interstage
  desuperheating is practically mandatory** and is modeled as an explicit optional component.
- **P_int is emergent at runtime** — it settles where the two compressor characteristics intersect;
  the geometric-mean rule is a design guideline only.
- **The condenser needs a multi-zone model**: desuperheating is 15–25 % of the duty at a gas-phase
  heat-transfer coefficient an order of magnitude below the condensing one.

---

## Input-variable taxonomy (from the task) → model roles

| Task class | Examples | Model role |
|------------|----------|-----------|
| 1. Fixed component parameters | η_is, η_vol, η_mech, displacement, motor curves, sensor/actuator limits | Model constants / performance maps |
| 2. Design-stage parameters | HX areas & geometry, pipe sizing, valve Kv, capacity selection | Design/optimization variables, frozen after build |
| 3. Runtime manipulated variables | Compressor speeds N1, N2; main valve u_mv; injection valve u_iv; secondary pump flows | Control inputs **u**(t) |
| (implicit) Boundary conditions | Source/process water inlet temperatures and demanded load | Disturbances **d**(t) |

---

## Nomenclature (core symbols)

| Symbol | Meaning | Unit |
|--------|---------|------|
| P, T, h, s, v, x | Pressure, temperature, spec. enthalpy, spec. entropy, spec. volume, vapor quality | kPa, °C (K in ratios), kJ/kg, kJ/(kg·K), m³/kg, – |
| ṁ_e, ṁ_inj, ṁ_c | Evaporator, injection, condenser mass flow | kg/s |
| r | Injection ratio ṁ_inj/ṁ_e | – |
| Q̇, Ẇ | Heat flow, power | kW |
| η_is, η_vol, η_mech | Isentropic, volumetric, mechanical efficiency | – |
| N, V_disp | Compressor speed, displacement | rev/s, m³/rev |
| UA, ε, NTU | Conductance, effectiveness, transfer units | kW/K, –, – |
| u, x, d | Control input vector, state vector, disturbance vector | – |
| SH | Superheat T − T_sat(P) | K |
| COP | Coefficient of performance Q̇_cond/Ẇ_el, with Ẇ_el = (Ẇ1+Ẇ2)/(η_mech·η_motor) the electrical input (the gas powers Ẇ1+Ẇ2 enter the energy balance Q̇_cond = Q̇_evap + Ẇ1 + Ẇ2) | – |

Subscripts: `e` evaporator/low-pressure side, `c` condenser/high-pressure side, `int` intermediate,
`is` isentropic, `f`/`g` saturated liquid/vapor, `sat` saturation, numbers = state points.

---

*Documents are written in English; key terms from the Chinese task sheet are given in parentheses
where they first appear. All property values from IAPWS-IF97; the JavaScript property engine in
`html/index.html` implements IF97 Regions 1, 2 and 4 directly and self-tests against the official
IF97 verification tables on load.*
