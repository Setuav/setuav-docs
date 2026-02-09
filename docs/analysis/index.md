# Setuav Analysis Module

## Purpose

The Setuav analysis module converts a design definition into engineering outputs.
It computes aerodynamic, propulsion, and flight performance results under a
given set of conditions (mass, CG, altitude, temperature, air density).

In practice, it is the computational layer between:

- **Input**: Design inputs and analysis conditions defined in the [Setuav Specification](https://setuav.github.io/specification/)
- **Output**: Performance data defined in the [Setuav Specification](https://setuav.github.io/specification/)

## How It Works

1. **Inputs are prepared**  
   The module reads the aircraft definition and analysis conditions.

2. **Calculations are executed**  
   Aerodynamics, propulsion, and flight performance are evaluated in order.

3. **Results are consolidated**  
   Outputs are combined into a single analysis result set for reporting and UI display.

## Solver Foundation

Setuav analysis uses AeroSandbox as the primary physics solver layer, then
applies Setuav-specific data handling and reporting on top of it.

In practical terms:

1. **AeroSandbox handles core physics solving**  
   Atmospheric state evaluation, aerodynamic coefficient calculation, and selected electric propulsion model utilities.

2. **Setuav handles product-level orchestration**  
   Design/specification mapping, component library integration, constraints/limits, and final output packaging for UI and reports.

## Inputs

The analysis module expects three input groups:

1. **Design definition**  
   Airframe geometry, propulsion setup, and component selections.

2. **Analysis conditions**  
   Flight mass, center of gravity position, altitude, temperature, and air density.

3. **Analysis settings (optional)**  
   Settings that control how calculations are performed, such as aerodynamics sweep range and resolution, propulsion electrical/thermal assumptions and limits, and flight performance speed range, step count, and safety margins.

## Outputs

The analysis module produces three output groups:

1. **Aerodynamics results**  
   Polar data and key aerodynamic metrics used to understand lift, drag, and efficiency behavior.

2. **Propulsion results**  
   Static and curve-based propulsion data such as thrust, power, current, and efficiency trends.

3. **Flight performance results**  
   Operational performance metrics such as stall speed, cruise performance, climb capability, range, and endurance.

## How To Use Results

Use analysis outputs as a decision support layer:

1. **Check feasibility first**  
   Verify that propulsion, stall speed, and basic performance targets are within acceptable limits.

2. **Compare design alternatives**  
   Evaluate multiple design options under the same analysis conditions for fair comparison.

3. **Use trends, not only single values**  
   Read curve behavior (not only one-point metrics) to understand operating margins and trade-offs.
