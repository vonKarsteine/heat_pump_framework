# Prototype Design (50 kW Laboratory Unit)

*This document turns the theory of docs 01–07 into buildable hardware: design basis and full nominal state table, compressor selection (the pivotal decision), heat exchangers, valves and piping, materials and vacuum integrity, instrumentation tied to the canonical state points, actuators and safety, control/DAQ hardware, and the commissioning + test matrix that closes the loop back to the models ([doc 08](08-dynamic-model.md) §8.8). Anchors per the [README](../README.md).*

---

## 10.1 Design basis

**Why 50 kW.** Large enough that industrially representative components exist off the shelf (MVR-class steam blowers, DN250+ vacuum piping, PLC-class controls); small enough for laboratory services (≈ 30 kW electrical feed, ≈ 27 kW of 20 °C source water ≈ 1.3 kg/s from a chilled/well loop, 8.6 m³/h process loop). One-line derivations:

$$ \dot m_{proc} = \frac{50\ \text{kW}}{4.19 \times (120 - 115)} \approx 2.39\ \text{kg/s} \;(8.6\ \text{m}^3/\text{h}); \qquad
\dot m_{src} = \frac{27}{4.19 \times 5} \approx 1.3\ \text{kg/s} \tag{10.1} $$

**Nominal state table** (design-mode march at the anchor: $T_e = 10$ °C, $T_c = 125$ °C, $P_{int} = 17$ kPa, $\Delta T_{sh} = 3$ K, $\varepsilon_{rec} = 0.15$, $\Delta T_{ec} = 5$ K, spray to $T_{sat}+15$ K, $\eta_{is} = 0.70$, $\eta_{drive} = 0.88$; live-recomputable in [`html/index.html`](../html/index.html)):

| State | Location | T (°C) | P (kPa) | h (kJ/kg) | ṁ (kg/s) |
|---|---|---|---|---|---|
| 1 | Evaporator out | 13 | 1.228 | 2525 | 0.0120 |
| 2 | LP suction | ≈ 20 | 1.228 | 2538 | 0.0120 |
| 3 | LP discharge | ≈ 385 | 17 | 3244 | 0.0120 |
| 4 | HP suction (post mix + spray) | ≈ 71 | 17 | 2632 | 0.0171 |
| 5 | HP discharge | ≈ 485 | 232.2 | 3455 | 0.0171 |
| 6 | Condenser liquid out | 125 | 232.2 | 525 | 0.0171 |
| 7 | Economizer hot out / tap | ≈ 61 | 232.2 | 257 | 0.0171 → split |
| 8 | Recuperator hot out | ≈ 58 | 232.2 | 244 | 0.0120 |
| 9 | Secondary valve out | 56.4 | 17 | 257 | ≈ 0.0019 |
| 10 | Injection vapor | 56.4 | 17 | 2602 | ≈ 0.0019 |
| 11 | Evaporator in | 10 | 1.228 | 244 | 0.0120 |
| — | Spray branch | 61 | → 17 | 257 | ≈ 0.0031 |

Headline results: $\dot W_{gas} \approx 23$ kW ($\approx 8.6 + 14.1$), $\dot W_{el} \approx 26$ kW, $\dot Q_{evap} \approx 27$ kW, **COP$_{el}$ ≈ 1.9**, closure $50 \approx 27 + 23$ ✓ ([doc 06](06-cycle-coupling-steady-state.md) Eq. 6.5).

## 10.2 Compressor selection — the pivotal decision

**The two facts that dominate everything:**

$$ \dot V_{suc,1} = \dot m_e\, v_2 \approx 0.0120 \times 106.3/ (1.0) \approx 1.3\ \text{m}^3/\text{s} \;(\approx 4\,700\ \text{m}^3/\text{h}) \quad \text{at } 1.228\ \text{kPa} \tag{10.2} $$

$$ \dot V_{suc,2} = \dot m_c\, v_4 \approx 0.0171 \times 9.0 \approx 0.15\ \text{m}^3/\text{s} \quad \text{at } 17\ \text{kPa} \tag{10.3} $$

— a **9:1 volume-flow disparity** between stages, and a per-stage pressure ratio ≈ 13.8 that exceeds what any single wheel or lobe pair can do:

| Technology | Per-unit PR | 1.3 m³/s @ 1 kPa? | Oil-free? | Verdict stage 1 | Verdict stage 2 |
|---|---|---|---|---|---|
| Multi-wheel high-speed **centrifugal** train (MVR practice) | 1.8–2.5 per wheel | Yes (this is MVR territory) | Yes | **Selected**: 4–6 wheels or 2–3 gear-driven rotors in series | Viable |
| **Roots** blowers in series + spray intercooling | 2–4 per unit | Yes at low PR | Yes | Backup option (3 units in series, spray between) | Marginal (PR) |
| Water-injected **twin screw** | up to 8–15 with liquid injection | No (volume) | With water injection | ✗ volume flow | **Selected option B** |
| Reciprocating | high | No | No | ✗ | ✗ volume/oil |

**Tip-speed physics bounds the wheel count**: steam's sonic speed at 10 °C is $a = \sqrt{kRT} \approx \sqrt{1.32 \times 461.5 \times 283} \approx 415$ m/s; keeping tip Mach ≲ 1.1–1.3 (titanium impellers, ~450–550 m/s tip) limits per-wheel PR to ≈ 1.8–2.5 for steam ⇒ $\ln 13.8/\ln 2.2 \approx 3.3$ → **4–6 wheels per thermodynamic stage**. Recommendation: **stage 1 = multi-wheel centrifugal steam blower train; stage 2 = water-injected twin screw or a 2-wheel centrifugal**, both oil-free (water contamination destroys vacuum and fouls surfaces), canned/hermetic or dry-gas-seal shafts, discharge-rated ≥ 300 °C (stage 1) with the spray chamber immediately downstream.

**System-level alternatives if procurement fails**, with their trade-offs stated:

- Raise $T_e$ 10 → 15–20 °C: $v_g$ falls 106 → 78 → 58 m³/kg (−45 % machine size) at the cost of source ΔT (needs 25 °C source or larger source flow); COP actually *improves* (+2.5 %/K) — the preferred fallback when the source allows.
- Go to **3 thermodynamic stages** (per-stage PR ≈ 5.7): each stage becomes 2 wheels; more hardware, easier wheels, lower discharge temperatures; criterion: choose 3 stages when per-stage PR of the 2-stage layout exceeds ~15 or when $T_5$ cannot be held ≤ limit with available spray.

## 10.3 Heat exchangers

| Duty | Type | Sizing basis | Result |
|---|---|---|---|
| Evaporator, 27 kW | **Falling-film horizontal shell-and-tube** (flooded ruled out: 0.1 m head ≈ 1 kPa ≈ 0.8 P1, [doc 04](04-heat-exchanger-models.md) §4.3) | $U \approx 1$–1.5 kW/m²K, LMTD ≈ 5–7 K | $A \approx 3$–5 m²; large vapor dome + demister (approach ≤ 4 m/s); sump + recirculation pump |
| Condenser, 50 kW | **Shell-and-tube or welded/brazed plate**; gasketed plate excluded (elastomer limit ~150 °C vs 490 °C inlet); inlet pass = desuperheat zone with gas-side area margin | 3-zone: DSH $U \approx 0.3$, COND $U \approx 3$–5, SC $U \approx 1$ kW/m²K | $A \approx 4$–6 m² total, of which ~40 % DSH area for ~20 % duty; optional separate desuperheater vessel upstream |
| Economizer, ≈ 4.5 kW | Compact **welded-plate** | $\Delta T_{ec} = 5$ K target, ε ≈ 0.75 | ~0.5–1 m² |
| Recuperator, ≈ 0.2 kW | Small coil-in-duct | $\varepsilon_{rec} = 0.15$ (deliberately modest, [doc 04](04-heat-exchanger-models.md) §4.5) | < 0.5 m²; sized for droplet-drying protection, not efficiency |

## 10.4 Valves and piping

**Control valves** (both steam/vacuum-rated industrial globe valves with positioners — refrigeration EEVs are not rated for deep-vacuum steam service):

- **Main valve**: liquid 0.0120 kg/s at ΔP ≈ 231 kPa, ρ ≈ 984 kg/m³ → required $K_v \approx Q\sqrt{\rho/(1000\,\Delta P_{bar})} = 0.044 \times \sqrt{0.984/2.31} \approx$ **0.03** — micro-flow trim; equal-percentage, turndown ≥ 5:1 (30–100 % load), flashing-service trim (hardened seat; choked-flow sizing per [doc 05](05-valves-mixing-junctions.md) §5.2.3).
- **Injection valve**: 0.0019 kg/s, ΔP ≈ 215 kPa → $K_v \approx 0.005$; **spray valve**: 0.0031 kg/s with fine atomizing nozzle rack (droplets ≤ 50 µm).
- Positioner feedback 4–20 mA + HART; stroke time ≤ 2 s.

**Suction line (the 82 Pa/K line):** target ΔP ≤ 100 Pa total (≈ 1.2 K penalty). For 1.3 m³/s at 20 m/s → $A \approx 0.065$ m² → **DN 300** (v ≈ 18 m/s); ΔP/m at 1.2 kPa, ρ ≈ 0.0094 kg/m³: $\Delta p = f \frac{L}{D}\frac{\rho v^2}{2} \approx 0.015 \times \frac{1}{0.3} \times 1.5 \approx 0.08$ Pa/m — the straight run is irrelevant (even 50 m ≈ 4 Pa); the 100 Pa budget is eaten by *local* losses (bends, demister, evaporator inlet: ζ × ≈ 1.5–2 Pa each), so still keep the run short with few fittings. DN 250 (v ≈ 26 m/s, ≈ 0.2 Pa/m) only if layout forces it; DN 350–400 buys margin cheaply. Interstage line at 17 kPa: DN 100; discharge line: DN 40.

- Slope all condensate-bearing lines ≥ 1:100 to drain points; no liquid traps on vapor lines.
- Insulation: hot side (120–500 °C) mineral wool w/ cladding, class per surface-temp code; vacuum side insulated to prevent condensation *inside* ducts from ambient heat leak being useful — actually to protect personnel and to stabilize the model's adiabatic assumptions.

## 10.5 Materials and vacuum integrity

- **304/316L stainless** wetted parts throughout; welded construction preferred over flanges; where flanged: graphite or spiral-wound gaskets on the hot side, elastomer limits respected (EPDM ≤ 150 °C, FKM ≤ 200 °C, none at the 490 °C discharge — metal-seated there).
- **Water chemistry**: deaerated demineralized water, pH 9–10 (ammonia or film-forming amine), oxygen scavenged — corrosion at 120–500 °C metal temperatures is the life-limiter.
- **Leak-tightness program**: helium leak test every joint, acceptance < 1×10⁻⁶ mbar·L/s per joint; plant vacuum-hold criterion **< 50 Pa/h rise** isolated at 1 kPa (a 1 kPa system breathing 50 Pa/h of air stops condensing within hours otherwise).
- **Permanently installed vacuum pump + non-condensable purge** taking suction from the condenser top (where non-condensables accumulate); automatic purge cycle triggered by the soft sensor of §10.6.

## 10.6 Instrumentation (tied to canonical state points)

| Tag(s) | Location (state) | Type | Range | Accuracy |
|---|---|---|---|---|
| T1, T2, T11 | Evaporator out, LP suction, evap in | Pt100 cl. A, thermowell | −10…+60 °C | ±0.15 K |
| T3, T4 | LP discharge, HP suction | Pt100 cl. A (T3 high-temp) | 0…450 °C | ±0.35 K |
| T5 | HP discharge | Type-N TC or HT-Pt100 | 0…600 °C | ±1 K |
| T6–T9 | Liquid chain | Pt100 cl. A | 0…150 °C | ±0.15 K |
| T-src-in/out, T-proc-in/out | Water loops | Pt100 cl. A, paired/matched | 0…150 °C | ±0.1 K matched |
| P1 | Evaporator shell | **Absolute capacitance** manometer | 0–10 kPa | ±0.25 % rdg |
| P-int (=P2/P3/P5 manifold) | Interstage | Absolute capacitance | 0–50 kPa | ±0.25 % |
| P4 | Condenser | Absolute piezo | 0–400 kPa | ±0.1 % FS |
| P-vac | Evacuation/purge line | Pirani | 10⁻¹–10⁵ Pa | decade |
| F-liq | Main liquid line (8) | **Coriolis** | 0–0.05 kg/s | ±0.2 % |
| F-inj, F-spray | Branch liquid lines | Coriolis (mini) | 0–0.01 kg/s | ±0.5 % |
| F-src, F-proc | Water loops | Electromagnetic | 0–3 kg/s | ±0.5 % |
| E1, E2 | Both VFDs | Power meters | 0–40 kW | ±0.5 % |
| N1, N2, Zmv, Ziv, Zspray | Speeds + valve positions | Encoder / positioner feedback | — | — |
| L-e | Evaporator sump | Guided-wave radar | 0–0.5 m | ±2 mm |

**Deliberate omission**: no vapor-side flow meters — at 1.2 kPa and 106 m³/kg any insertion element's Δp is a multi-kelvin penalty (82 Pa/K rule) and thermal mass flow needs density knowledge you don't have with mist. **Soft sensors instead**: $\dot m_1$ from the stage-1 map (speed + pressures, [doc 03](03-compressor-models.md)); superheat $SH_1 = T1 - T_{sat}(P1)$; non-condensable indicator $\Delta_{nc} = P4 - P_{sat}(T6)$ (healthy ≈ 0; alarm at ≥ 2 kPa sustained — realistic against the 0.1 % P4 sensor, i.e. ≈ 0.4 K equivalent, rather than a heroic 150 Pa threshold).

## 10.7 Actuators, electrical, safety devices

- **Drives**: stage-1 motor ≈ 11 kW shaft → 15 kW motor + VFD; stage-2 ≈ 16 kW shaft → 22 kW motor + VFD (splits follow §10.1 works with margin; both 400 V/50 Hz).
- **Pumps**: source and process loop pumps on VFDs; evaporator film-recirculation pump; all with dry-run protection (the vacuum-side pump NPSH is razor thin — flooded suction mandatory).
- **Overpressure**: PRV set 3.5 bar(a) + rupture disc 4.0 bar(a) on the condenser side (vessel design pressure 4.5 bar(a)); vacuum side is self-limiting (atmosphere is the worst case → vessels rated full-vacuum external).
- **Hardwired trip chain** (independent of PLC): high T5 > 550 °C, high P4 > 3.2 bar(a), loss-of-flow switches both water loops, E-stop; all latching, manual reset. Full cause→action→reset matrix lives in [doc 09](09-control-design.md) §9.7 (single source of truth — not duplicated here).

## 10.8 Control and DAQ hardware

- **Controller**: PLC-class (Siemens S7-1500 or Beckhoff CX) for the permanent plant; NI cRIO acceptable for the research phase if control-law flexibility (MPC prototyping) outweighs industrial robustness. IO tally from §10.6–10.7: ≈ 24 AI, 8 AO, 16 DI, 12 DO + 2 drive fieldbuses → one rack.
- **Loop rates**: 100 ms for pressure/superheat/override loops, 1 s for thermal loops ([doc 09](09-control-design.md) FOPDT table justifies); safety scan 10 ms.
- **Logging**: **1 Hz minimum all channels** (the model-ID requirement of [doc 08](08-dynamic-model.md) §8.8), 10 Hz burst mode during step tests; OPC-UA northbound + CSV export; synchronized timestamps (PTP or PLC clock master).
- **HMI screens**: overview mimic (the [doc 01](01-system-overview.md) schematic with live values — same layout as the HTML page), trends, alarm list, interlock status, test-sequence panel (scripted steps for Phase 2/3), manual/auto per loop with bumpless transfer.

## 10.9 Commissioning and test matrix

**Phase 0 — Integrity & calibration (no refrigerant duty):** pressure test hot side; helium leak survey; vacuum pull-down to < 100 Pa and 24 h hold (< 50 Pa/h); water-loop balancing; sensor loop calibration incl. paired-RTD zeroing; charge with deaerated DI water; boil-out degassing run.

**Phase 1 — First vapor / mechanical:** evaporator vapor generation with vacuum pump running (no compressors); stage-1 solo run on minimum speed against the interstage relief path; vibration/temperature survey; stage-2 solo; first full-loop closure at reduced lift ($T_c \approx 60$–80 °C); verify soft-sensor flow vs Coriolis closure.

**Phase 2 — Steady-state performance map:** hold each grid point ≥ 30 min stationarity; measure COP, all states, powers.

| Grid axis | Points |
|---|---|
| Source inlet | 15, 20, 25 °C |
| Load | 30, 50, 70, 100 % |
| $P_{int}$ trim (via $N_1/N_2$) | 5 points spanning 14–24 kPa at the 20 °C/100 % point |

Acceptance: measured COP within **±8 %** and discharge temperatures within **±5 K** of the [doc 06](06-cycle-coupling-steady-state.md) solver prediction; energy closure of measurements $|\dot Q_{cond} - \dot Q_{evap} - \dot W_{gas}|/\dot Q_{cond} < 3\,\%$ (instrument-limited).

**Phase 3 — Dynamic identification** (feeds [doc 08](08-dynamic-model.md) §8.8): ±10 % steps on each MV about nominal ($N$, $u_{mv}$, $u_{iv}$, $u_{spray}$, both pump speeds), 3 repeats; optional PRBS on $u_{mv}$ and $N$; 1–10 Hz logging; deliver FOPDT fits (gain, τ, θ) per channel with confidence bounds.

**Phase 4 — Closed loop:** loop-by-loop bring-up in the [doc 09](09-control-design.md) §9.5 order (SH → P_int trim → supply temperature → supervisory), IMC/SIMC initial tunings from Phase-3 fits, relay autotune verification; disturbance-rejection demos (load step 100→50 %, source ramp 20→15 °C); provoke **every interlock** of doc 09 §9.7 under controlled conditions and log the response.

| Run-ID template | Held | Varied | Measured | Success criterion |
|---|---|---|---|---|
| P2-S20-L100-Pi17 | source 20 °C, load 100 % | — (steady) | full state vector, COP | model match ±8 % |
| P3-STEP-umv-+10 | all others | $u_{mv}$ +10 % step | SH₁(t), P1(t) | FOPDT fit R² > 0.9; detect/quantify inverse response |
| P4-DIST-LOAD-50 | controllers on | load 100→50 % | $T_{proc,out}$(t), all MVs | |ΔT_supply| ≤ 0.5 K after 300 s, SH ≥ 1 K throughout, no interlock |

Completion of Phase 4 with the acceptance rows green **is** the prototype's exit criterion — at that point the validated model + tuned controller become the deliverable for scale-up.
