# Flight Performance

Flight performance analysis derives the speed envelope, mission speeds, climb capability, range, and endurance from aerodynamic and propulsion outputs. The core objective is to compare, at each speed point, the thrust/power required for level flight against the thrust/power the system can actually provide, and to keep this comparison physically consistent.

## Method

Setuav solves this as a coupled velocity sweep. The aerodynamic side first computes level-flight $C_L$, $C_D$, drag, and required power; then the propulsion side is evaluated with a single shared model. That shared model is `PropulsionAnalyzer.solve_required_thrust_sweep(...)`, which uses the same motor-propeller-battery equilibrium solver used by propulsion analysis. As a result, range/endurance, cruise, and envelope metrics in flight performance remain consistent with the propulsion electrical model.

## Preparation

### Input Preparation

The analyzer uses the following inputs and maps them into solution variables:

| Input | Solution mapping |
| --- | --- |
| `aero.cl_max` | $C_{L,\max}$ for stall-speed calculation |
| `aero.area` | Reference area $S$ (`mm^2 -> m^2`) |
| `aero.polars.cl_values`, `aero.polars.cd_values` | $C_D(C_L)$ interpolation |
| `aero.cd_min` | Constant-drag fallback when polar is missing |
| `aero.ld_max` | `max_ld_ratio`, `glide_ratio` outputs |
| `aero.operating_velocity` | `best_ld` speed output |
| `total_mass` | Weight $W = mg$ |
| `air_density` | Density $\rho$ |
| `battery_capacity` | Energy-based range/endurance (`mAh -> Ah`) |
| `design.propulsion` | Voltage-window validation and propulsion solver setup |

### Configuration Inputs

The fields used directly by this class are:

| Config field | Solution mapping |
| --- | --- |
| `config.performance.velocity_min` | Lower bound of velocity sweep |
| `config.performance.velocity_max` | Upper bound of velocity sweep |
| `config.performance.velocity_steps` | Number of velocity points |
| `config.performance.stall_margin` | Minimum safe speed: $V_{\min}=V_{\text{stall}}\cdot\text{stall\_margin}$ |
| `config.propulsion.usable_capacity_ratio` | Usable battery-energy scaling factor |

## Solution

The solution runs the same sequence over one shared velocity array and produces all curves on that array.

### Stall Speed and Velocity Bounds

Stall speed is computed from lift equilibrium:

$$
V_{\text{stall}}=\sqrt{\frac{2W}{\rho S C_{L,\max}}}
$$

Sweep start is not simply the configured minimum. It is clamped to the safe start speed:
$V_{\text{start}}=\max(V_{\text{stall}}\cdot\text{stall\_margin},\,V_{\min,config})$.

### Level-Flight Power Required

At each speed point, required lift coefficient is:

$$
C_{L,\text{req}} = \frac{W}{0.5\rho V^2 S}
$$

Then $C_D$ is obtained from polar interpolation $C_D(C_L)$; outside the polar bounds, parabolic extrapolation is used:

$$
C_D = C_{D0} + k C_L^2
$$

Drag and required power:

$$
D = 0.5\rho V^2 S C_D,
\qquad
P_{\text{req}} = D\,V
$$

### Coupled Propulsion Solve (Single Model)

For the same velocity array, the required-thrust curve is passed to propulsion:

$$
[T_{\text{avail}}(V),\;P_{\text{avail}}(V),\;P_{\text{batt,req}}(V),\;\text{feasible}(V)]
\leftarrow
\texttt{solve\_required\_thrust\_sweep}
$$

Here $P_{\text{avail}}$ is available **propulsive power**, while $P_{\text{batt,req}}$ is battery-side electrical power required to hold the level-flight point.

### Endurance and Range Curves

Energy is computed from the same electrical model output $P_{\text{batt,req}}$. Usable battery energy:

$$
E_{\text{batt}} = V_{\text{nom}}\,Q_{\text{Ah}}\,\eta_{\text{usable}}
$$

For only feasible points with positive required electrical power:

$$
\text{Endurance}(V) = \frac{E_{\text{batt}}}{P_{\text{batt,req}}(V)},
\qquad
\text{Range}(V) = V\cdot 3.6\cdot \text{Endurance}(V)
$$

### Optimal Speeds

On the feasible mission mask:

- `best_endurance`: speed with minimum $P_{\text{batt,req}}$
- `best_range`: speed with maximum `range_curve`
- `best_climb`: speed with maximum $(P_{\text{avail}}-P_{\text{req}})$
- `best_ld`: directly `aero.operating_velocity`

In this analyzer, reported `cruise_speed` is assigned as `best_range` in the raw
calculation flow. In other words, cruise selection is driven by direct range
maximization, without an additional UI/operational rule at this step.

Raw-data alternatives for cruise definition:

1. `best_endurance`-based cruise:  
   choose speed minimizing $P_{\text{batt,req}}(V)$; objective is maximum loiter time.
2. `best_climb`-based cruise:  
   choose speed maximizing $(P_{\text{avail}}-P_{\text{req}})$; objective is climb-phase priority.
3. `best_ld`-based cruise:  
   choose aerodynamic-efficiency reference speed (`aero.operating_velocity`).

### Performance Metrics and Cruise Point

Maximum speed is taken as the last feasible velocity point. Climb rate is:

$$
\text{RoC}(V)=\frac{P_{\text{avail}}(V)-P_{\text{req}}(V)}{W}
$$

Best climb angle is computed at the maximum excess-power point using
$\sin\gamma=\text{RoC}/V$.

Cruise is not computed by a separate approximate model; it is solved with the same propulsion operating-point solver via `solve_required_thrust_operating_point(...)`.

### Invalidity and Fallback Behavior

If battery voltage is incompatible with motor/ESC voltage windows, analysis returns a zeroed result. If no propulsion solver is available, available-power/thrust curves remain zero and powered metrics are limited. If battery capacity is not available, range/endurance curves and related metrics remain zero.
