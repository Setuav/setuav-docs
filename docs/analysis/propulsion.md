# Propulsion

## Method

Setuav propulsion is solved through a single speed-dependent flow. Each operating point is defined by flight speed $V$ and throttle $u$. Propeller coefficients are read from the database using $\mathrm{RPM}$ and advance ratio $J$; motor electrical-mechanical balance is solved with $K_v$, $R$, and $I_0$; battery-side sag and efficiency losses are applied in the same loop. This keeps static and dynamic outputs physically consistent because they come from one chain.

## Preparation

### Component Preparation

Before solving, propulsion components are validated for completeness and physical consistency. Motor, battery, and propeller data are normalized into one solver-ready system specification.

| Design field | Solver representation | Validation / Note |
| --- | --- | --- |
| `propulsion.motors[].kv` | $K_v$ | Required, must be $> 0$ |
| `propulsion.motors[].resistance` | $R$ (temperature-scaled) | Required, must be $> 0$, scaled by `motor_resistance_temp_factor` |
| `propulsion.motors[].no_load_current` | $I_0$ | Required, must be $\ge 0$ |
| `propulsion.motors[].current_max` | $I_{\max}$ | Replaced with a high bound if missing |
| `propulsion.batteries[].voltage_nominal` | $V_{\mathrm{nom}}$ | Required, must be $> 0$ |
| `propulsion.batteries[].cells_series` | $n_{\mathrm{cells}}$ | Used in sag; inferred from voltage if missing |
| `propulsion.batteries[].cell_resistance` | part of $R_{\mathrm{pack}}$ | Config fallback when missing |
| `propulsion.propellers[].(diameter,pitch,blade_count)` | DB lookup input | Database match is mandatory |

If torque constant is not overridden in config, it is derived from speed constant:

$$
K_t = \frac{60}{2\pi K_v}
$$

At the end of this stage, the solver has one system object containing motor, battery, propeller, and motor count.

### Condition Preparation

Atmospheric state used in propulsion equations is built from analysis conditions.

| Analysis field | Solver representation | Note |
| --- | --- | --- |
| `conditions.altitude_msl` | Atmosphere model | Contributes to density state |
| `conditions.temperature` | Atmosphere model | Affects density/speed-of-sound state |
| `conditions.air_density` | $\rho$ | Used as direct override when provided |
| `conditions.total_mass` | $\mathrm{thrust\_to\_weight}$ reporting | Used mainly for output metric |

Inside the solve loop, propeller-side input is $V$, while electrical-side input is $V_{\mathrm{pack}}$, updated each iteration through sag feedback.

## Solution

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

Once coefficients are known, torque, thrust, and shaft power are obtained. Here $\rho$ is air density:

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

Physically, higher propeller torque drives higher $I_{\mathrm{motor}}$, which raises both copper loss and required terminal voltage. The iterative solve searches for the $\mathrm{RPM}$-current pair that satisfies this coupled balance.

Motor-side electrical power is mapped to battery side through ESC and battery efficiencies. Here $\eta_{\mathrm{esc}}$ is ESC efficiency and $\eta_{\mathrm{batt}}$ is battery discharge efficiency:

$$
P_{\mathrm{motor,elec}} \approx V_{\mathrm{motor}} I_{\mathrm{motor}}
$$

$$
P_{\mathrm{batt}} = \frac{P_{\mathrm{motor,elec}}}{\eta_{\mathrm{esc}} \eta_{\mathrm{batt}}}
$$

$$
I_{\mathrm{pack}} = \frac{P_{\mathrm{batt}}}{V_{\mathrm{pack}}}
$$

The critical effect in this stage is cumulative loss stacking: for the same shaft requirement, battery power must increase as conversion efficiencies are applied, which then increases battery current.

After pack current is known, voltage sag is updated via pack resistance $R_{\mathrm{pack}}$ and fed back into the next iteration:

$$
V_{\mathrm{pack,new}} = V_{\mathrm{nom}} - I_{\mathrm{pack}}R_{\mathrm{pack}}
$$

This feedback captures the load-dependent voltage drop: when current rises, pack voltage falls, and the motor must satisfy thrust demand under a more constrained electrical state.

Thermal and current safety are checked in the same loop. Here $T_{\mathrm{amb}}$ is ambient temperature, $R_{\mathrm{th}}$ is thermal resistance, and $T_{\max}$ is temperature limit:

$$
T_{\mathrm{motor}} = T_{\mathrm{amb}} + \left(P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}\right)R_{\mathrm{th}}
$$

$$
I_{\mathrm{motor}} > I_{\max}
\quad \text{or} \quad
T_{\mathrm{motor}} > T_{\max}
$$

The thermal equation treats the electrical-to-shaft power gap as heat. As $P_{\mathrm{motor,elec}} - P_{\mathrm{shaft}}$ grows, temperature rises proportionally and can invalidate the point.

If either limit is exceeded, the operating point is rejected. For valid points, system efficiency is reported in gram-per-watt form. With $T_g$ as thrust in grams:

$$
\eta_{g/W} = \frac{T_g}{P_{\mathrm{batt}}}
$$

This metric directly quantifies how much thrust is produced per battery Watt, making cross-speed comparison straightforward. The same chain is evaluated across all speeds for dynamic curves, and the near-zero-speed slice is used for static outputs.
