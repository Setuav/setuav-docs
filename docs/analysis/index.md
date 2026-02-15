# Setuav Analysis Module

The Setuav analysis module converts a design definition into engineering outputs. Under a given analysis state (mass, CG, altitude, temperature, air density), it computes aerodynamics, propulsion, and flight performance results. In practice, this module sits between design/condition inputs and performance outputs defined in the [Setuav Specification](https://setuav.github.io/specification/).

The execution flow is fixed: input preparation, aerodynamics solve, propulsion solve, flight performance solve, then result consolidation. Flight performance depends directly on aerodynamic outputs; when propulsion data is required and propulsion solve fails, propulsion-dependent performance outputs are limited. Consolidation runs only after all available domain results are collected.

| Stage | Main input | What you get |
| --- | --- | --- |
| Input preparation | Design + conditions | Analysis-ready inputs |
| Aerodynamics | Geometry + atmosphere state | Aerodynamic behavior and efficiency indicators |
| Propulsion | Propulsion setup + atmosphere state | Thrust, power, current, and efficiency outputs |
| Flight performance | Aerodynamics (+ propulsion if available) | Stall, cruise, climb, range, endurance metrics |
| Consolidation | Domain outputs | Unified result set for report and UI |

Setuav uses AeroSandbox as the physics solver layer, while Setuav-specific logic handles specification mapping, limit checks, component integration, and final result packaging. This separation keeps physics solving consistent and reporting/product behavior predictable.

Inputs are grouped into three blocks: design definition, analysis conditions, and optional analysis settings. Outputs are grouped into three blocks as well: aerodynamics results, propulsion results, and flight performance results. This structure enables fair comparison of design alternatives under identical conditions.

When reviewing outputs, start with feasibility, then compare alternatives under the same conditions, and evaluate curve trends together with single-point metrics. Stall, thrust margin, climb, range, and endurance should be interpreted as a consistent set rather than isolated values.
