# System-Level Coupling and Steady-State Solution

*This document assembles the component models of [doc 03](03-compressor-models.md), [doc 04](04-heat-exchanger-models.md) and [doc 05](05-valves-mixing-junctions.md) into one closed algebraic system — the task's system-level equation set (系统级方程组) for steady-state analysis (稳态分析) — audits its degrees of freedom against the task's input-variable classes, and derives two solution algorithms plus the simplified design-mode march used by the interactive calculator in [`html/index.html`](../html/index.html). Conventions per the [README](../README.md).*

---

## 6.1 Problem statement

Collect every steady-state relation into

$$ F(z;\, u,\, d,\, p) = 0 \tag{6.1} $$

- $z$ — unknown cycle state (enthalpies, pressures, temperatures, flows),
- $u$ — runtime manipulated inputs (task class 3, 运行中可调控): $u = (N_1, N_2, u_{mv}, u_{iv}, u_{spray}, \dot m_{src}, \dot m_{proc})$,
- $d$ — disturbances/boundary conditions: $d = (T_{src,in}, T_{proc,in}, \dot Q_{demand})$,
- $p$ — parameters: class 1 (fixed component data: $\eta_{is}, \eta_{vol}, V_{disp}, \ldots$) and class 2 (frozen design: $UA$'s, $K_v$'s, volumes).

A well-posed model has exactly as many independent equations as unknowns once $u, d, p$ are given — that is the audit of §6.3.

## 6.2 Unknown vector

Enthalpy-level unknowns are reduced first by the identities $h_9 = h_7$ and $h_{11} = h_8$ (isenthalpic valves) and $h_{10} = h_g(P_{int})$ (saturated injection vapor by the economizer closure choice). The independent set is then:

| Group | Unknowns | Count |
|-------|----------|-------|
| Enthalpies | $h_1, h_2, h_3, h_4, h_5, h_6, h_7, h_8$ | 8 |
| Pressures | $P_1, P_{int}, P_4$ | 3 |
| Refrigerant flows | $\dot m_e, \dot m_{inj}, \dot m_{spray}$ | 3 |
| Secondary outlets | $T_{src,out}, T_{proc,out}$ | 2 |
| Zone/aux temps | $T_c$ (condensing), $T_e$ (evaporating), $T_7$ | 3 |
| **Total** | | **19** |

(Temperatures at states 1–5, 8 are *dependent*: recovered from $(P, h)$ by property inversion, not counted. $T_e = T_{sat}(P_1)$ and $T_c = T_{sat}(P_4)$ are listed separately only to keep the saturation relations explicit below.)

## 6.3 Equation list and degrees-of-freedom audit

Numbered residuals (each block cites its source document):

| # | Residual | Source |
|---|----------|--------|
| R1 | $P_1 - P_{sat}(T_e) = 0$ | [doc 02](02-water-properties.md) R4 |
| R2 | $P_4 - P_{sat}(T_c) = 0$ | doc 02 R4 |
| R3 | $h_1 - \left[h_g(T_e) + c_{p,v}\Delta T_{sh}\right] = 0$ (superheat definition; $\Delta T_{sh}$ from the SH controller or spec) | doc 04 §4.3 |
| R4 | Recuperator: $h_2 - h_1 - \varepsilon_{rec} c_{p,v}(T_7 - T_1) = 0$ | doc 04 (4.20) |
| R5 | Recuperator hot side: $(h_7 - h_8) - (h_2 - h_1) = 0$ | doc 04 (4.19) |
| R6 | Compressor 1 flow: $\dot m_e - \eta_{vol,1}\rho_2 V_{disp,1} N_1 = 0$ | doc 03 (3.2) |
| R7 | Compressor 1 energy: $h_3 - h_2 - \dfrac{h(P_{int}, s_2) - h_2}{\eta_{is,1}} = 0$ | doc 03 (3.6) |
| R8 | Mixing + spray node: $\dot m_e h_3 + \dot m_{inj} h_{10} + \dot m_{spray} h_7 - \dot m_c h_4 = 0$ | doc 05 (5.13) |
| R9 | Spray target (controller active): $h_4 - h(P_{int}, T_{sat}(P_{int}) + \Delta T_{sh,4}) = 0$; else $\dot m_{spray} = 0$ | doc 05 (5.12) |
| R10 | Compressor 2 flow: $\dot m_c - \eta_{vol,2}\rho_4 V_{disp,2} N_2 = 0$, $\dot m_c \equiv \dot m_e + \dot m_{inj} + \dot m_{spray}$ | doc 03 (3.2) |
| R11 | Compressor 2 energy: $h_5 - h_4 - \dfrac{h(P_4, s_4) - h_4}{\eta_{is,2}} = 0$ | doc 03 (3.6) |
| R12–R15 | Condenser 3-zone balances + rates (4 equations: DSH, COND, SC, water-side chain) | doc 04 (4.6)–(4.9) |
| R16 | Condenser water duty: $\dot m_{proc} c_{p,w}(T_{proc,out} - T_{proc,in}) - \dot m_c (h_5 - h_6) = 0$ | doc 04 (4.10) |
| R17 | Economizer balance: $\dot m_c (h_6 - h_7) - \dot m_{inj}(h_{10} - h_7) = 0$ | doc 04 (4.16) |
| R18 | Economizer closure: $T_7 - T_{sat}(P_{int}) - \Delta T_{ec}(UA_{ec}, \ldots) = 0$ | doc 04 (4.17) |
| R19 | Main valve flow: $\dot m_e - C_d A(u_{mv})\sqrt{2\rho_8 (P_4 - P_1)} = 0$ (choked-flow variant per doc 05 §5.2.3) | doc 05 (5.5) |
| R20 | Secondary valve flow: $\dot m_{inj} - C_d A(u_{iv})\sqrt{2\rho_7 (P_4 - P_{int})} = 0$ | doc 05 (5.5) |
| R21 | Evaporator balance: $\dot m_e (h_1 - h_8) - \dot Q_{evap} = 0$ with $\dot Q_{evap}$ from the rate eq. | doc 04 (4.13)–(4.14) |
| R22 | Evaporator water side: $\dot Q_{evap} - \dot m_{src} c_{p,w}(T_{src,in} - T_{src,out}) = 0$ | doc 04 (4.15) |

Count: the audit is made transparent by expanding the two exchanger blocks fully. The condenser block (R12–R16) is really six relations — three zone-rate equations (4.6)–(4.8) plus the three-line water chain (4.9) — closed by the duty relation R16, against seven unknowns: the outer pair $(h_6, T_{proc,out})$ and five internals (intermediate water temperatures $T_{w,1}, T_{w,2}$ and zone duties $\dot Q_{dsh}, \dot Q_{cond}, \dot Q_{sc}$ of doc 04). The evaporator likewise carries one internal flow variable $\dot Q_{evap}$, pinned one-to-one by the rate equation bundled into R21. That gives $19 + 6 = 25$ equations for $19 + 6 = 25$ unknowns; the six internals are linear-algebraic in the states and eliminate on the spot, leaving the **19 × 19 outer system — 19 equations for 19 unknowns**. The system closes exactly when $u$, $d$ and $p$ are specified, mirroring the task's input-variable classification. Adding hardware detail (pressure-drop terms, motor model, zone splits) always adds one equation per new unknown and keeps the audit balanced.

Two structural observations:

1. **Nothing in the list "sets" $P_{int}$ directly** — it appears in R7, R9, R10, R17, R18, R20 but has no dedicated equation. It is pinned *implicitly* by the interaction of the two compressor flow equations (R6, R10): that is §6.4.
2. The controller view: at fixed $u$ the plant state is unique (locally); the *control problem* of [doc 09](09-control-design.md) is choosing $u(t)$ so that dependent outputs ($T_{proc,out}$, SH, $T_5$) meet their targets.

## 6.4 Why the intermediate pressure is emergent

Eliminate all unknowns except $P_{int}$ (possible in principle since the rest of the system is solvable for any trial $P_{int}$): what remains is a single scalar residual — an interstage **mass balance**:

$$ \Phi(P_{int}) = \underbrace{\dot m_e(P_{int})}_{\text{stage-1 delivery}} + \underbrace{\dot m_{inj}(P_{int}) + \dot m_{spray}(P_{int})}_{\text{injection + spray}} - \underbrace{\dot m_c(P_{int})}_{\text{stage-2 swallowing}} = 0 \tag{6.2} $$

Stage-1 delivery *falls* with rising $P_{int}$ (back-pressure raises its pressure ratio, $\eta_{vol,1}$ drops; for turbo maps the characteristic slopes down toward surge), while stage-2 swallowing *rises* with $P_{int}$ (denser suction at fixed $N_2 V_{disp,2}$). $\Phi$ is therefore monotone in the operating band and Eq. (6.2) has a unique root: **the interstage pressure floats to wherever the two compressor characteristics intersect.** Graphically: plot delivered and swallowed mass flow vs $P_{int}$; the crossing is the operating point, and moving either speed slides its curve and thus the root.

Consequences, stated once and used everywhere:

- $P_{int}$ is **not a runtime actuator** (a point the task's class-3 list might tempt one to blur): the controller commands $N_1/N_2$ (primary) and $u_{iv}$ (trim), and $P_{int}$ *responds*.
- The geometric-mean rule $\sqrt{P_1 P_4} = 16.9$ kPa is a **design-stage guideline** for choosing $V_{disp,1}/V_{disp,2}$ so that the intersection lands near the COP-optimal value at the nominal point ([doc 07](07-performance-cop-optimization.md) §7.4).

## 6.5 Solution algorithm A — sequential substitution (teaching form)

For insight and hand calculation, march around the loop with three outer iteration variables $(T_e, T_c, P_{int})$:

```text
guess Te, Tc, Pint  (anchor values are excellent starts: 10 °C, 125 °C, 17 kPa)
repeat
  P1 = Psat(Te); P4 = Psat(Tc)
  1 → 2   recuperator (needs T7 from previous sweep; first pass: T7 = Tsat(Pint)+ΔTec)
  2 → 3   compressor 1 (η_is)
  economizer: h6 = hf(Tc) [first pass]; h7 from ΔTec; r, w_s from doc-05 eqs (5.14)–(5.15)
  mix → 4;  4 → 5  compressor 2
  condenser: h6 from 3-zone model  → update
  recuperator: h8 = h7 − (h2 − h1);  valves: h9 = h7, h11 = h8
  flows: m_e from compressor-1 flow eq (or from demanded duty in design mode)
  CORRECTIONS (one Newton/secant step each):
    Te   ← from evaporator rate residual R21 (too little area → lower Te)
    Tc   ← from condenser rate residuals R12–R15 (pinch violated → raise Tc)
    Pint ← from interstage balance Φ(Pint) = 0  (Eq. 6.2)
until |ΔTe|, |ΔTc|, |ΔPint| < tol
```

Convergence behavior: the loop is a fixed-point iteration; it converges linearly and reliably when the HX conductances are generous (weak feedback) but can limit-cycle when the condenser pinch is tight (strong $T_c \leftrightarrow h_5$ coupling through the DSH zone) or when compressor curves are flat near the $\Phi$ root (ill-conditioned $P_{int}$ correction). Damping the three corrections (relaxation factor 0.5) restores convergence in practice. When it fails persistently, switch to Algorithm B.

## 6.6 Solution algorithm B — damped Newton–Raphson (production form)

Solve Eq. (6.1) simultaneously:

$$ z^{(k+1)} = z^{(k)} - \lambda_k\, J^{-1}\!\left(z^{(k)}\right) F\!\left(z^{(k)}\right), \qquad J = \frac{\partial F}{\partial z} \tag{6.3} $$

Implementation notes that matter for *this* plant:

- **Log-pressure scaling.** The pressures span 1.228 → 232.2 kPa (two decades). Iterating on $\ln P$ instead of $P$ (and scaling enthalpy residuals by $h_{fg}$) keeps the Jacobian condition number manageable; unscaled, a 1 kPa step is 0.4 % of $P_4$ but 80 % of $P_1$ and Newton steps trample the vacuum side.
- **Jacobian**: finite differences are adequate (19×19); analytic property derivatives $(\partial \rho/\partial P,\ \partial \rho/\partial h)$ from IF97 ([doc 08](08-dynamic-model.md) uses them anyway) sharpen it at negligible cost. Reuse $J$ across iterations (Shamanskii) — property calls dominate runtime.
- **Damping/backtracking**: accept the step only if $\|F\|$ decreases; halve $\lambda_k$ otherwise (Armijo). With the anchor initial guess, 3–6 iterations reach $\|F\| < 10^{-8}$.
- **Bounds**: box constraints keep iterates physical ($P_1 > 0.7$ kPa above triple point, $T_c < 170$ °C, $0 \le x \le 1$); project-and-restart on violation.

## 6.7 Design-mode simplification (the HTML calculator)

The interactive calculator ([`html/index.html`](../html/index.html)) answers *design* questions — "what does the cycle look like **if** it runs at $T_e, T_c, P_{int}$?" — so it **specifies** the three loop temperatures/pressures that Algorithms A/B iterate on, replacing the three rate residuals (R21, R12–R15 aggregate, and Eq. 6.2) with direct inputs. What is left is a purely **sequential explicit march**:

$$ 1 \to 2 \to 3 \to \{r, w_s\} \to 4 \to 5 \to 6 \to 7 \to 8 \to 11 \tag{6.4} $$

whose only iterations are safeguarded-bisection property inversions $T(P, h)$ and $T(P, s)$ inside IF97 Region 2 — monotone, bracketed, unconditionally convergent. This is legitimate because at *steady design conditions* the HX rate equations only tell you what $UA$ must be to *realize* the assumed approach temperatures; the calculator reports the implied duties so the designer can size $UA$ afterwards ([doc 10](10-prototype-design.md)). The full solver and the design march agree by construction wherever the specified $(T_e, T_c, P_{int})$ coincide with the rate-equation solution.

## 6.8 Verification protocol

**(a) Global energy closure — an identity, not a coincidence.** Sum the component balances around the loop. Each enthalpy appears once positively (stream leaving a component) and once negatively (entering the next); internal transfers (recuperator, economizer, spray) appear once $+$ and once $-$ by construction. The telescoping sum leaves only the boundary crossings:

$$ \dot Q_{cond} = \dot Q_{evap} + \dot W_1 + \dot W_2 \tag{6.5} $$

Sketch: $\dot Q_{cond} - \dot W_2 = \dot m_c(h_4 - h_6)$; substitute the node balance R8 for $\dot m_c h_4$, the economizer R17 for $\dot m_c h_6$, and the recuperator pair R4/R5 for $(h_2, h_8)$; every internal enthalpy cancels and $\dot m_e(h_1 - h_8) + \dot m_e(h_3 - h_2) = \dot Q_{evap} + \dot W_1$ remains. Any numerical solution violating Eq. (6.5) beyond property-inversion tolerance (< 0.5 %) has a bug — this is the calculator's live "closure" badge and the reviewer rule of [doc 07](07-performance-cop-optimization.md) §7.2 (COP ≤ Carnot).

**(b) Limiting cases.**

| Limit | Expected collapse |
|---|---|
| $u_{iv} \to 0$ (so $r \to 0$) and spray off | Plain two-stage cycle; Eq. (5.11) → $h_4 = h_3$; economizer transparent ($h_7 = h_6$) |
| $\varepsilon_{rec} \to 0$ | $h_2 = h_1$, $h_8 = h_7$; recuperator vanishes |
| Both above | Textbook single-suction two-stage compression — cross-checkable against any refrigeration text |
| $N_1 = N_2$, $V_{disp,2}/V_{disp,1} \to \rho_2/\rho_4$ | $\Phi = 0$ at the design $P_{int}$: consistency of the displacement-ratio selection |

**(c) Mass closure at nodes**: $\dot m_c - \dot m_e - \dot m_{inj} - \dot m_{spray} = 0$ machine-exactly at N1/N4 (they share one variable set), and the condensate return $\dot m_e$ at the evaporator equals the vapor draw at state 1 in steady state.

All three checks are implemented in the calculator's self-test block and printed to the browser console on load.
