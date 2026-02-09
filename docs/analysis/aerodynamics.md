# Aerodynamics

Aerodynamic analysis is performed to determine where the aircraft can fly safely and efficiently across speed and angle-of-attack ranges. This calculation reveals lift, drag, and efficiency behavior, defines stall-related margins, and provides the core data used by downstream performance analysis.

## Method

This stage uses the AeroSandbox software. AeroSandbox is an open-source Python engineering software developed by Peter D. Sharpe to accelerate aircraft design optimization and solve aerodynamics/propulsion/structures/mission analyses in a unified numerical framework. It is selected in Setuav because it can directly model parametric aircraft geometry and flight conditions and produce consistent `CL`/`CD` outputs across angle-of-attack sweeps. From a reliability perspective, the method is transparent and auditable through open-source code, MIT licensing, public version history, public documentation, and directly citable MIT thesis work ([AeroSandbox](https://peterdsharpe.github.io/AeroSandbox/), [GitHub](https://github.com/peterdsharpe/AeroSandbox), [MIT 2021 Thesis](https://dspace.mit.edu/handle/1721.1/140023)).

The main AeroSandbox components used at this stage are:

- `Airplane`
- `Wing` and `WingXSec`
- `Fuselage` and `FuselageXSec`
- `Atmosphere`
- `OperatingPoint`
- `AeroBuildup`

## Preparation

### Geometry Preparation

In AeroSandbox, geometry is represented as a top-level `Airplane` object; wings are defined as `Wing` objects (with `WingXSec` sections), and the fuselage is defined as a `Fuselage` object (with `FuselageXSec` sections). In other words, profile/section-based geometry from the specification is mapped directly into these section-driven analysis objects, where each section carries explicit position, size, angle, and airfoil/profile information.

| Specification field | AeroSandbox mapping | Conversion / Note |
| --- | --- | --- |
| `airframe.wings[].component.geometry.profiles[].position.(x,y,z)` | `WingXSec.xyz_le` | `mm -> m` conversion is applied |
| `airframe.wings[].component.geometry.profiles[].chord` | `WingXSec.chord` | `mm -> m` conversion is applied |
| `airframe.wings[].component.geometry.profiles[].rotation.x` | `WingXSec.twist` | Passed in degrees |
| `airframe.wings[].component.geometry.profiles[].airfoil` | `WingXSec.airfoil` | `asb.Airfoil` is built from NACA / file / coordinate definitions |
| `airframe.wings[].attachment.position.(x,z)` | `WingXSec.xyz_le` offset | `mm -> m`; wing attachment `y` offset is fixed to centerline |
| `airframe.wings[].attachment.rotation.(x,y,z)` | `WingXSec.xyz_le` transform | Applied with XYZ rotation matrix |
| `airframe.wings[].attachment.mirror` | `Wing.symmetric` | `mirror` value from the model is used directly |
| `airframe.fuselage.geometry.segments[].sections[].position.(x,y,z)` | `FuselageXSec.xyz_c` | `mm -> m` conversion is applied |
| `airframe.fuselage.geometry.segments[].sections[].profile` | `FuselageXSec.width`, `FuselageXSec.height` | Dimensions are assigned by `circle/ellipse/rectangle/rounded_rectangle` profile type |
| Aerodynamic reference point | `Airplane.xyz_ref` | Set to `[0.0, 0.0, 0.0]` at this stage |

### Flight Condition Preparation

After geometry is prepared, analysis conditions are mapped to AeroSandbox atmosphere and flight-state objects. Environmental state is built with `Atmosphere`, and the instantaneous solution state is built with `OperatingPoint`.

| Analysis condition field | AeroSandbox mapping | Conversion / Note |
| --- | --- | --- |
| `conditions.altitude_msl` | `Atmosphere.altitude` | Passed directly in `m` |
| `conditions.temperature` | `create_atmosphere(..., temperature_c)` | `°C` input is used in atmosphere construction |
| `conditions.air_density` | Atmosphere remapped via `Atmosphere.temperature_deviation` | If density override is provided, temperature deviation is solved numerically to match target density |
| Aerodynamic reference velocity (analysis setup) | `OperatingPoint.velocity` | Current initial aerodynamics setup uses `15.0 m/s` |
| Initial angles | `OperatingPoint.alpha`, `OperatingPoint.beta` | Initialized at `0°`; `alpha` sweep is applied in the next solution step |

## Solution

### Aerodynamic Coefficient Solving

At this stage, an `alpha` sweep is generated and `AeroBuildup` is evaluated at each angle point to obtain aerodynamic coefficients. Pointwise `CL`/`CD` arrays are then converted by Setuav into derived metrics used in the performance report.

`AeroBuildup` is AeroSandbox's solver that computes total aircraft aerodynamics by combining component-level contributions into a single result. It uses aircraft geometry, flight state (`OperatingPoint`), and atmospheric state together to produce total lift and drag coefficients at each `alpha` point; viscous effects, stall behavior, and configuration-dependent wave-drag effects are included in this modeling stage. Setuav takes `CL` and `CD` directly from this solver and derives the remaining report metrics on top.

| Solution step | Implementation | Output |
| --- | --- | --- |
| `alpha` sweep generation | Linear sweep from `alpha_min`, `alpha_max`, `alpha_steps` | `alpha` points |
| Solver configuration | `AeroBuildup` is configured with `include_wave_drag` and resolution-derived `model_size` (`small/medium/large`) | Configuration-dependent coefficient solution |
| State assumptions | Each `alpha` point is solved with `beta=0`, `p=0`, `q=0`, `r=0` | Quasi-steady coefficient set |
| Pointwise coefficient solving | `AeroBuildup(airplane, op_point)` is evaluated at each `alpha` | `CL(alpha)`, `CD(alpha)` |
| Efficiency computation | `L/D = CL / CD` is computed at each point | `L/D(alpha)` |
| Key metric extraction | `CL_max`, `CD_min`, `L/D_max` are selected from arrays | Summary aerodynamic metrics |
| Reference geometry derivation | Span/area are computed from wing sections; `mean_chord` and `AR` are derived | `span`, `area`, `mean_chord`, `aspect_ratio` |
| Reynolds calculation | `Re = rho * V * mean_chord / mu` | `reynolds_number` |
| Report unit conversion | Geometry outputs are converted before report writeout: `m -> mm`, `m² -> mm²` | Specification-facing report units |
