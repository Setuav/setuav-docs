# Analysis Workflow

This page explains how an analysis run progresses from input data to final results.
It is intended to help users understand what happens at each stage and where to
focus when reviewing outputs.

## Workflow At A Glance

The process is sequential:

1. **Input preparation**
2. **Aerodynamics calculation**
3. **Propulsion calculation**
4. **Flight performance calculation**
5. **Result consolidation**

## Dependency Logic

- Flight performance depends on aerodynamics outputs.
- If propulsion data is required and propulsion fails, propulsion-dependent
  performance outputs are limited.
- Consolidation runs after all available domain results are collected.

## Preconditions

Before running analysis, the following should be ready:

- A complete aircraft definition in the design file
- A valid propulsion setup for propulsion/performance calculations
- Analysis conditions (mass, center of gravity position, altitude, temperature, air density)

## Stage Map

| Stage | Main input | What you get |
| --- | --- | --- |
| Input preparation | Design + conditions | Analysis-ready inputs |
| Aerodynamics | Geometry + atmosphere state | Aerodynamic behavior and efficiency indicators |
| Propulsion | Propulsion setup + atmosphere state | Thrust, power, current, and efficiency outputs |
| Flight performance | Aerodynamics (+ propulsion if available) | Stall, cruise, climb, range, endurance metrics |
| Consolidation | Domain outputs | Unified result set for report and UI |

## What To Check At Each Stage

| Stage | Recommended user check |
| --- | --- |
| Before run | Inputs are complete and analysis conditions are realistic |
| After aerodynamics | Lift/drag behavior is consistent with the intended speed regime |
| After propulsion | Thrust and power margins are sufficient for mission expectations |
| After performance | Stall, cruise, climb, range, and endurance match requirements |
| Final review | All result groups are mutually consistent |
