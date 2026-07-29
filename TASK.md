# Task Brief — Two-Stage Water-Vapor High-Temperature Heat Pump

> This file is an **English restatement of the original assignment** (originally a Chinese task
> sheet, 4 pages, with a system schematic). The original document is intentionally not distributed
> with this repository; this brief carries all information needed to follow the project. The
> schematic's state-point labels are reproduced in [doc 01](docs/01-system-overview.md) with one
> correction (a duplicated "T10" label — see the README).

## 1. System background (系统背景)

The subject is a **high-temperature heat pump with water (H₂O, R718) as the working fluid and
two-stage compression**. It absorbs heat from the environment (e.g. ambient air or a low-temperature
water source around 20 °C) and lifts it to **above 120 °C** for heat delivery. Typical applications:
industrial process heating and high-temperature hot-water supply.

## 2. System structure (系统结构概览)

| Component | Role |
|---|---|
| **Evaporator** (蒸发器) | Absorbs heat from the external source, vaporizing the water working fluid at low pressure. Because saturation temperature at low pressure is modest, air-source (finned-tube-like) or water-source exchangers can supply the heat. |
| **Low-pressure compressor** (低压压缩机, stage 1) | Compresses the low-pressure vapor from the evaporator to an intermediate pressure; outlet temperature and pressure both rise. |
| **Economizer** (经济器, intermediate vapor injection) | Sits between the high-pressure side and the intermediate pressure: a throttled fraction of high-pressure working fluid becomes intermediate-pressure vapor, injected between the two compressor stages. Improves system efficiency and gives the high-pressure compressor a more favorable suction state. |
| **High-pressure compressor** (高压压缩机, stage 2) | Takes the mixture of stage-1 discharge and economizer injection vapor and compresses it to the condensing pressure corresponding to ≥ 120 °C. |
| **Recuperator** (回热器, if fitted) | Uses part of the high-temperature stream's heat to preheat colder streams (e.g. subcools high-pressure liquid against low-pressure suction vapor), further raising overall efficiency. |
| **Condenser** (冷凝器) | High-temperature, high-pressure steam condenses here, transferring heat to the process demand (~120 °C water). The condensate returns through the throttling/split devices to the evaporator, closing the cycle. |

The schematic shows: condenser at top (process water 115 °C in → T_out), evaporator at bottom
(source water 20 °C in → T_out cold), the two compressors in series on the right with the injection
mixing point between them, the economizer and recuperator in the middle of the liquid line, and the
main + secondary expansion valves (主/次膨胀阀). State labels T1…T10 and pressure levels P1…P5
follow the loop; this project renumbers the evaporator inlet to **state 11** because the original
figure used "T10" twice.

## 3. Core research content (核心研究内容)

1. **System modeling** (系统建模)
   - For each component above: write the mass and energy balance equations plus the necessary
     constitutive relations (water/steam equation of state, heat-transfer coefficient correlations,
     etc.). *In the original assignment this part is primarily the student's work.*
   - Couple all components mathematically into a complete **system-level equation set**, suitable
     for **steady-state analysis** and **transient simulation**. *This part — and everything
     downstream of it — is what this repository delivers* (docs [06](docs/06-cycle-coupling-steady-state.md),
     [08](docs/08-dynamic-model.md)).
2. **Objectives** (目标)
   - Optimize the system design to reach the required **120 °C supply temperature** while keeping a
     high **COP** (docs [07](docs/07-performance-cop-optimization.md), [10](docs/10-prototype-design.md)).
   - Study the system's **dynamic response and control strategy** under changing operating
     conditions — ambient/source temperature and load variations (docs
     [08](docs/08-dynamic-model.md), [09](docs/09-control-design.md)).

## 4. Input variables, in three classes (系统的输入变量)

The assignment classifies all inputs by changeability:

### Class 1 — Component performance parameters that cannot be changed (无法改变)

Fixed once the component is purchased/manufactured; intrinsic to the hardware:

- Compressor: isentropic efficiency, mechanical efficiency, volumetric efficiency, displacement
  (or maximum swept volume), rotor/motor characteristics (inertia, efficiency curves), maximum
  speed / torque limits.
- Controller & sensor hardware limits: maximum valve opening, drive frequency ceiling, sensor
  ranges and accuracies.

These enter the models as **given constants or manufacturer performance maps**.

### Class 2 — Choosable at the design stage, fixed afterwards (设计阶段可以选定, 一旦设计完成就固定)

- Heat exchanger design (incl. recuperator and economizer): heat-transfer areas (tube length,
  bundle, fin areas), tube diameter/length/fin pitch, circuit counts, parallel/series circuits.
- System piping and layout: pipe diameters and lengths (flow resistance and system volume),
  filters/driers/receivers sizing and placement.
- Overall capacity / model selection: target heating capacity, the two compressors' power range.
- Expansion-valve / throttling specifications: orifice size, maximum flow capacity (the *maximum
  adjustable range* is fixed at design time even though opening is adjustable in operation).
- Economizer / recuperator area and flow configuration — capping the internal heat-recovery
  capability.

Treated as **design/optimization variables** during modeling, frozen once hardware is built.

### Class 3 — Adjustable during operation (运行过程中可调控)

The manipulated variables of the control system:

- **Compressor speeds** — with variable-frequency drives, continuously adjustable to match load or
  target temperature.
- **Expansion-valve openings** — control evaporator superheat or pressure; an adjustable injection
  throttle belongs here too.
- **Secondary fluid flows** — air/water flows via fan or pump speed.
- **Injection valve opening / switching** — tunes the intermediate injection to optimize
  performance or manage the intermediate pressure.

These are the **control inputs u(t)**; regulating them keeps the heat pump efficient and stable
across operating conditions — the core of the control-system design in
[doc 09](docs/09-control-design.md).

## 5. Scope split

Per the original assignment: component-level modeling is the student's task; the system-level
coupling, steady-state and transient analysis, optimization, and control-strategy development are
the assisted scope — which is exactly what this repository provides, plus a prototype design and an
interactive calculator ([html/index.html](html/index.html)) for exploring the design space.
