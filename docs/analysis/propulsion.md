# Propulsion

Propulsion analysis computes how much thrust an electric propulsion system can produce at a given flight speed and what electrical cost is required to produce it. The result depends directly on motor physical properties ($K_v$, $R$, $I_0$, current limit), battery electrical state (nominal voltage, internal resistance, efficiency, usable capacity), ESC conversion efficiency, propeller geometry and aerodynamic characteristics ($C_t$, $C_p$), and ambient conditions (especially air density $\rho$). Physically, the calculation is solved as one coupled energy-loading chain: compute advance ratio $J$ from speed and rotational rate, derive propeller thrust/torque loading, solve motor voltage-current-torque equilibrium, then feed battery/ESC losses and voltage sag back into the same operating point.

## Method

This method treats the physical problem as a coupled equilibrium: for a given flight speed $V$ and throttle $u$ (or required thrust), propeller loading, motor electrical balance, and battery/ESC energy conversion must be satisfied at the same operating point. The solver first defines propeller operating state through $J$, then iteratively matches the resulting torque-thrust demand with motor-side $K_v$-$R$-$I_0$ voltage-current-torque balance. Voltage sag and conversion losses are fed back each iteration until a single converged operating point is found, so static and dynamic outputs come from the same physics model.

## Preparation

### Component Preparation

The goal of this stage is to convert propulsion inputs into a solver-ready and physically consistent data set. After validation, the analysis runs on one unified component specification instead of handling incomplete or conflicting fields at runtime.

Preparation normalizes three groups: motor electrical parameters, battery pack parameters, and propeller geometry. This keeps all operating-point solves on the same data structure.

| Input | Solution mapping |
| --- | --- |
| `propulsion.motors[].kv` | $K_v$ |
| `propulsion.motors[].resistance` | $R$ (temperature-scaled motor resistance) |
| `propulsion.motors[].no_load_current` | $I_0$ |
| `propulsion.motors[].current_max` | $I_{\max}$ |
| `propulsion.batteries[].voltage_nominal` | $V_{\mathrm{nom}}$ |
| `propulsion.batteries[].cells_series` | $n_{\mathrm{cells}}$ |
| `propulsion.batteries[].cells_parallel` | Parallel string count ($n_{\mathrm{parallel}}$) |
| `propulsion.batteries[].cell_resistance` | part of $R_{\mathrm{pack}}$ |
| `propulsion.propellers[].(diameter,pitch,blade_count)` | Geometry input for propeller operating state |

### Condition Preparation

The goal of condition preparation is to translate analysis-side environment and mission inputs into solver variables used directly by propulsion equations. This stage normalizes the atmospheric state and mass-dependent reporting inputs.

| Analysis input | Solution mapping |
| --- | --- |
| `conditions.altitude_msl` | Altitude input to atmosphere model |
| `conditions.temperature` | Temperature input to atmosphere model |
| `conditions.air_density` | $\rho$ (direct density override) |
| `conditions.total_mass` | Mass used for $\mathrm{thrust\_to\_weight}$ reporting |

### Analysis Configuration Inputs

In addition to design and atmosphere inputs, the table below lists only the analysis parameters that are exposed directly in the UI and their solution-side mappings.

| Config field | Solution mapping |
| --- | --- |
| `config.propulsion.use_battery_internal_resistance` | Sag model switch; enables/disables battery internal resistance contribution in the iteration. |
| `config.propulsion.motor_efficiency_default` | Lower-bound motor efficiency for electrical power; used as $P_{\mathrm{motor,elec}}=\max\!\left(V_{\mathrm{motor}}I_{\mathrm{motor}},\,P_{\mathrm{shaft}}/\eta_{\mathrm{default}}\right)$. |
| `config.propulsion.back_emf_scale` | $\eta_{\mathrm{bemf}}$ calibration factor; lower values require higher voltage/current for the same RPM. |
| `config.propulsion.usable_capacity_ratio` | Usable energy factor; scales battery energy budget in propulsion outputs (especially runtime estimate). |
| `config.propulsion.battery_discharge_efficiency` | $\eta_{\mathrm{batt}}$ efficiency; maps motor-side electrical demand to battery-side demand. |
| `config.propulsion.esc_efficiency` | $\eta_{\mathrm{esc}}$ efficiency; maps ESC conversion losses into battery draw. |
| `config.propulsion.rpm_steps` | Static sweep resolution; sets the number of sampled operating points in static solution. |
| `config.propulsion.motor_max_temperature` | $T_{\max}$ limit; sets the upper temperature bound for valid operating points. |
| `config.propulsion.motor_thermal_resistance` | $R_{\mathrm{th}}$ factor; converts motor loss power into temperature rise. |
| `config.propulsion.cooling_level` | Cooling level (1-5); multiplies effective thermal resistance: 1→1.00, 2→0.95, 3→0.80, 4→0.75, 5→0.70. |

## Solution

The solution is performed as a coupled operating-point search: propeller operating state is set from speed and rotational rate, then the resulting torque-thrust demand is matched with motor voltage-current equilibrium. Battery/ESC losses and voltage sag are fed back into this balance, and the point is accepted only if it passes validity limits.

Power flow is represented directly in the solver as:

$$
P_{\mathrm{bat}}
\;\xrightarrow[\mathrm{ESC}]{\eta_{\mathrm{esc}}}\;
P_{\mathrm{motor}}
\;\xrightarrow[\mathrm{Motor}]{I^2R,\;I_0,\;\eta_{\mathrm{bemf}}}\;
P_{\mathrm{shaft}}
\;\xrightarrow[\mathrm{Propeller}]{C_t,\;C_p,\;J}\;
T
$$

Here, $P_{\mathrm{bat}}$ is battery draw, $P_{\mathrm{motor}}$ is motor electrical input, $P_{\mathrm{shaft}}$ is shaft power, and $T$ is thrust output. Terms written above each transition indicate the dominant conversion/loss mechanism in that stage.

The loss summary for these transitions is:

| Stage | Transition | Loss mechanism |
| --- | --- | --- |
| ESC | $P_{\mathrm{bat}} \rightarrow P_{\mathrm{motor}}$ | Switching/conduction losses, $P_{\mathrm{motor}} \approx P_{\mathrm{bat}}\eta_{\mathrm{esc}}$ |
| Motor | $P_{\mathrm{motor}} \rightarrow P_{\mathrm{shaft}}$ | Copper loss $P_{\mathrm{Cu}} = I_{\mathrm{motor}}^2R$, no-load loss $P_0 \approx V_{\mathrm{motor}}I_0$, additional magnetic/mechanical losses |
| Propeller | $P_{\mathrm{shaft}} \rightarrow T$ | Aerodynamic conversion losses; operating point set by $C_t/C_p$ and $J$ |

### Propeller Operating Point

The calculation starts by defining the propeller operating state. With flight speed $V$, rotational speed $n$, and diameter $D$, advance ratio is:

$$
J = \frac{V}{nD}
$$

This equation makes $J$ the core loading indicator of forward-flight propeller operation. As $J$ changes, the propeller shifts to a different aerodynamic operating condition, which directly changes required torque and produced thrust.

Using $J$ and $\mathrm{RPM}$, coefficients are read from the propeller database:

$$
C_t = C_t(\mathrm{RPM}, J),
\qquad
C_p = C_p(\mathrm{RPM}, J)
$$

### Aerodynamic Load to Torque and Power

Once coefficients are known, torque $Q_{\mathrm{prop}}$, thrust $T$, and shaft power $P_{\mathrm{shaft}}$ are obtained. Here $\rho$ is air density:

$$
Q_{\mathrm{prop}} = \frac{C_p \rho n^2 D^5}{2\pi}
$$

$$
T = C_t \rho n^2 D^4
$$

$$
P_{\mathrm{shaft}} = C_p \rho n^3 D^5
$$

These equations define how aerodynamic loading is reflected into electrical demand. The strong diameter exponents ($D^4$, $D^5$) explain why even small diameter changes can significantly alter power and current levels.

### Motor Electrical Equilibrium

In motor equilibrium, torque-current and voltage balance are solved together. Here $K_v$ is speed constant, $K_t$ is torque constant, $I_0$ is no-load current, and $R$ is temperature-scaled resistance:

$$
I_{\mathrm{motor}} = \frac{Q_{\mathrm{prop}}}{K_t} + I_0
$$

$$
V_{\mathrm{back}} = \frac{\mathrm{RPM}}{K_v \eta_{\mathrm{bemf}}}
$$

$$
V_{\mathrm{motor}} \approx V_{\mathrm{back}} + I_{\mathrm{motor}}R
$$

Here $\eta_{\mathrm{bemf}}$ is the back-EMF calibration factor (`back_emf_scale` in advanced propulsion settings). Physically, higher propeller torque drives higher $I_{\mathrm{motor}}$, which raises both copper loss and required terminal voltage. The iterative solve searches for the $\mathrm{RPM}$-current pair that satisfies this coupled balance.

### Battery and ESC Power Mapping

Motor-side electrical power is mapped to battery side through ESC and battery efficiencies. Here $\eta_{\mathrm{esc}}$ is ESC efficiency and $\eta_{\mathrm{batt}}$ is battery discharge efficiency. The solver uses a conservative electrical-power estimate based on both voltage-current product and shaft/efficiency floor:

$$
P_{\mathrm{motor,elec}} =
\max\!\left(
V_{\mathrm{motor}} I_{\mathrm{motor}},
\frac{P_{\mathrm{shaft}}}{\eta_{\mathrm{default}}}
\right)
$$

$$
P_{\mathrm{batt}} = \frac{P_{\mathrm{motor,elec}}}{\eta_{\mathrm{esc}} \eta_{\mathrm{batt}}}
$$

$$
I_{\mathrm{pack}} = \frac{P_{\mathrm{batt}}}{V_{\mathrm{pack}}}
$$

The critical effect in this stage is cumulative loss stacking: for the same shaft requirement, battery power must increase as conversion efficiencies are applied, which then increases battery current.

### Voltage Sag Feedback

After pack current is known, voltage sag is updated via effective pack resistance and fed back into the next iteration. Here $R_{\mathrm{pack}}$ includes internal battery resistance (if enabled) plus wire resistance:

$$
R_{\mathrm{pack}} = R_{\mathrm{int}} + R_{\mathrm{wire}}
$$

$$
V_{\mathrm{pack,new}} =
\max\!\left(
V_{\mathrm{nom}} - I_{\mathrm{pack}}R_{\mathrm{pack}},
0.5\,V_{\mathrm{nom}}
\right)
$$

This feedback captures the load-dependent voltage drop: when current rises, pack voltage falls, and the motor must satisfy thrust demand under a more constrained electrical state.

### Validity Limits

This step determines whether an operating point is acceptable by checking current and thermal limits. Here $T_{\mathrm{amb}}$ is ambient temperature, $R_{\mathrm{th}}$ is thermal resistance, $k_{\mathrm{cool}}$ is cooling-level factor, and $T_{\max}$ is temperature limit:

$$
T_{\mathrm{motor}} =
T_{\mathrm{amb}} +
\left(P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}\right)
R_{\mathrm{th}}\,k_{\mathrm{cool}}
$$

$$
I_{\mathrm{motor}} > I_{\max}
\quad \text{or} \quad
T_{\mathrm{motor}} > T_{\max}
$$

The thermal equation treats the electrical-to-shaft power gap as heat. As $P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}$ grows, temperature rises proportionally; smaller $k_{\mathrm{cool}}$ reduces temperature rise for the same loss power. If either threshold is exceeded, the operating point is rejected.

### Reported Efficiency

System efficiency is reported only for valid operating points, in gram-per-watt form. With $T_g$ as thrust in grams:

$$
\eta_{g/W} = \frac{T_g}{P_{\mathrm{batt}}}
$$

This metric directly quantifies how much thrust is produced per battery Watt, making cross-speed comparison straightforward. The same chain is evaluated across all speeds for dynamic curves, and the near-zero-speed slice is used for static outputs.
